---
description: 关于 kubectl logs 方法的源码理解
---

# kubectl logs pod

kubectl logs 提供 follow 、 tail 等方法，针对一个或者多个 pod 实现日志打印和追踪，对于kubectl 的实现方式，进行源码阅读

简单总结，kubectl 通过将输入参数进行转换成 pod.PodLogOptions 并使用 client-set 调用 cord.pod.GetLogs 实现，实际调用的返回及流式处理都是 api-server 实现

```go
// staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go:186
// 将入参转成 PodLogOptions
func (o *LogsOptions) ToLogOptions() (*corev1.PodLogOptions, error) {
   logOptions := &corev1.PodLogOptions{
      Container:                    o.Container,
      Follow:                       o.Follow,
      Previous:                     o.Previous,
      Timestamps:                   o.Timestamps,
      InsecureSkipTLSVerifyBackend: o.InsecureSkipTLSVerifyBackend,
   }
   //....
   
// staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go:326
// logs 函数启动入口，o.LogsForObject 为 client-go logs 函数封装入口
func (o LogsOptions) RunLogs() error {
   requests, err := o.LogsForObject(o.RESTClientGetter, o.Object, o.Options, o.GetPodTimeout, o.AllContainers)

// o.LogsForObject 封装内容
// staging/src/k8s.io/kubectl/pkg/polymorphichelpers/logsforobject.go:37
func logsForObject(restClientGetter genericclioptions.RESTClientGetter, object, options runtime.Object, timeout time.Duration, allContainers bool) (map[corev1.ObjectReference]rest.ResponseWrapper, error) {
	clientConfig, err := restClientGetter.ToRESTConfig()
	if err != nil {
		return nil, err
	}

	clientset, err := corev1client.NewForConfig(clientConfig)
	if err != nil {
		return nil, err
	}
	// 该函数将多个不同的pod 内容组装的 map[corev1.ObjectReference]rest.ResponseWrapper 对象提供外部使用，request 为流式内容
	return logsForObjectWithClient(clientset, object, options, timeout, allContainers)
}

// staging/src/k8s.io/kubectl/pkg/cmd/logs/logs.go:345
// 并发输出请求，入参内容则是 logsForObject 返回的内容
func (o LogsOptions) parallelConsumeRequest(requests map[corev1.ObjectReference]rest.ResponseWrapper) error {
	reader, writer := io.Pipe()
	wg := &sync.WaitGroup{}
	wg.Add(len(requests))
	for objRef, request := range requests {
		go func(objRef corev1.ObjectReference, request rest.ResponseWrapper) {
			defer wg.Done()
			out := o.addPrefixIfNeeded(objRef, writer)
			if err := o.ConsumeRequestFn(request, out); err != nil {
				if !o.IgnoreLogErrors {
					writer.CloseWithError(err)

					// It's important to return here to propagate the error via the pipe
					return
				}

				fmt.Fprintf(writer, "error: %v\n", err)
			}

		}(objRef, request)
	}

	go func() {
		wg.Wait()
		writer.Close()
	}()

	_, err := io.Copy(o.Out, reader)
	return err
}
```
