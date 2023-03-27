# CPC协程泄漏

## 功能背景

CPC 对接持续集成，完成对应用运行时镜像的构建动作。其中涉及对 job 的发起、完成状态的监听，需要使用 watch 监听状态变化并获取构建日志、处理构建逻辑

## 故障现象

主节点CPC内存使用率持续增长并没有下降趋势，其他副本内存使用约30Mb左右，主节点内存使用能够增长到1\~2Gb

## 故障排查

通过pprof 获取主节点的 goroutine / heap 信息，发现主节点出现大量的 goroutine 信息，并与 heap 占用量大的调用相同

!\[image-20220819143607484]\(/Users/hanhui/Library/Application Support/typora-user-images/image-20220819143607484.png)

### 重现测试代码

client-go sdk version 0.18.8

```go
package test

import (
	"context"
	"fmt"
	"k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"os"
	"runtime/pprof"
	"sync"
	"testing"
	"time"
)

var (
	phases = map[v1.PodPhase]struct{}{
		v1.PodSucceeded: {}, v1.PodFailed: {},
	}
)

func TestOnlyWatch(t *testing.T) {
	config, err := clientcmd.BuildConfigFromFlags("", "/Users/hanhui/.kube/config")
	if err != nil {
		t.Fatal(err)
	}
	cli, err := kubernetes.NewForConfig(config)
	if err != nil {
		t.Fatal(err)
	}
	wg := sync.WaitGroup{}
	ctx := context.Background()
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func(i int) {
			defer wg.Done()
			name := fmt.Sprintf("t-%v", i)
			defer deletePod(ctx, cli, name)
			if err := createPod(ctx, cli, name); err != nil {
				t.Error(err)
				return
			}
			if err := watchPod(ctx, cli, name); err != nil {
				t.Error(err)
				return
			}
		}(i)
	}

	wg.Wait()
	time.Sleep(time.Second * 5)
	if err := pprofWrite("goroutine", "only-watch.pprof"); err != nil {
		t.Fatal(err)
	}
}

func createPod(ctx context.Context, clientSet *kubernetes.Clientset, name string) error {
	fmt.Println("create pod: ", name)
	_, err := clientSet.CoreV1().Pods("default").Create(ctx, podTemplate(name), metav1.CreateOptions{})
	if err != nil {
		return err
	}
	return nil
}

func deletePod(ctx context.Context, clientSet *kubernetes.Clientset, name string) error {
	fmt.Println("delete pod: ", name)
	err := clientSet.CoreV1().Pods("default").Delete(ctx, name, metav1.DeleteOptions{})
	if err != nil {
		return err
	}
	return nil
}

func watchPod(ctx context.Context, clientset *kubernetes.Clientset, name string) error {
	fmt.Println("watch pod: ", name)
	w, err := clientset.CoreV1().Pods("default").Watch(ctx, metav1.ListOptions{
		LabelSelector: labels.SelectorFromSet(labels.Set{"job": name}).String(),
	})
	if err != nil {
		return err
	}
  // 故障关键操作
	defer w.Stop()
	ch := w.ResultChan()
	for {
		select {
		case <-ctx.Done():
			return nil
		case event, ok := <-ch:
			if !ok {
				return fmt.Errorf("链接中断")
			}
			pod, ok := event.Object.(*v1.Pod)
			if !ok {
				return fmt.Errorf("object 断言失败")
			}
			fmt.Printf("%v status: %v\n", name, pod.Status.Phase)
			if _, ok := phases[pod.Status.Phase]; ok {
        // 故障关键操作
				return deletePod(ctx, clientset, name)
			}
		}
	}
}

func podTemplate(name string) *v1.Pod {
	return &v1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,
			Namespace: "default",
			Labels:    map[string]string{"job": name},
		},
		Spec: v1.PodSpec{
			Containers: []v1.Container{
				{
					Name:    name,
					Image:   "docker.io/library/busybox",
					Command: []string{"/bin/sh", "-c", "sleep 10"},
				},
			},
			RestartPolicy: v1.RestartPolicyNever,
		},
	}
}

func pprofWrite(name string, fileName string) error {
	f, err := os.OpenFile(fileName, os.O_CREATE|os.O_RDWR|os.O_TRUNC, os.ModePerm)
	if err != nil {
		return err
	}
	defer f.Close()
	return pprof.Lookup(name).WriteTo(f, 0)
}

```

### client-go 故障代码

```go
func NewStreamWatcher(d Decoder, r Reporter) *StreamWatcher {
	sw := &StreamWatcher{
		source:   d,
		reporter: r,
		// It's easy for a consumer to add buffering via an extra
		// goroutine/channel, but impossible for them to remove it,
		// so nonbuffered is better.
		result: make(chan Event),
	}
	go sw.receive()
	return sw
}

// receive reads result from the decoder in a loop and sends down the result channel.
func (sw *StreamWatcher) receive() {
	defer utilruntime.HandleCrash()
	defer close(sw.result)
	defer sw.Stop()
	for {
		action, obj, err := sw.source.Decode()
		if err != nil {
			// Ignore expected error.
			if sw.stopping() {
				return
			}
			switch err {
			case io.EOF:
				// watch closed normally
			case io.ErrUnexpectedEOF:
				klog.V(1).Infof("Unexpected EOF during watch stream event decoding: %v", err)
			default:
				if net.IsProbableEOF(err) || net.IsTimeout(err) {
					klog.V(5).Infof("Unable to decode an event from the watch stream: %v", err)
				} else {
					sw.result <- Event{
						Type:   Error,
						Object: sw.reporter.AsObject(fmt.Errorf("unable to decode an event from the watch stream: %v", err)),
					}
				}
			}
			return
		}
    // 故障位置，出现堵塞
		sw.result <- Event{
			Type:   action,
			Object: obj,
		}
	}
}
```

### 故障说明

​ 测试代码中，在 watch 事件的同时，进行了一次 delete 操作。该操作将产生一个新的 event ，并被 watcher 监听。

​ 而此时**可能**并未到测试代码的 defer stop 进行关闭监听， watcher 已经进入了下一次循环并从 sw.source.Decode() 获取了数据，并进入 sw.result <- Event ，而 sw.result 是非缓冲管道 result : make(chan Event) ，需要等待外部读取后才会进行退出，但是测试代码已经不再进行管道获取，因此该堵塞将会永久存在

## 故障修复

### 修复一

client-go v0.21 已修复该问题。通过 sdk 版本升级即可

### 修复二

将 delete 动作放置到 watch.stop 后执行 （不建议），仍旧有可能存在问题

### 修复三

对 watch chan 进行排空处理，防止堵塞
