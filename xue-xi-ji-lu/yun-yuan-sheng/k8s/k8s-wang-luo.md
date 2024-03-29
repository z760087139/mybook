# K8S网络

## 基础条件

1. 所有 pod 互相直接通信，无需显示使用NAT
2. 所有Node 可以与 pod 直接通信，无需显示使用NAT
3. pod 可见自身 IP 地址与其他 pod 所见 ip相同，不需要显示转换

## 网络层级

* 容器之间的通信
* pod 之间的通信
* pod 与 service 的通信
* 外界与 service 的通信

## 网络方案分类

* Underlay
* Overlay

### Underlay

需要与 host 网络取得协同

### Overlay

并不需要从 host 的ip组件取得IP，可以自由分配

## Network Namespace

隔离的网络空间

* 独立的附属网络设备 lo/ veth 虚拟设备、物理网卡
* 独立的协议栈， IP地址和路由表
* iptable规则
* ipvs

**每个 pod 拥有独立的 network ns**, pod 内的 container 共享该空间，通过 loopback 接口实现通信

或者通过共享 pod-ip 对外提供服务

host 上还有 root net ns ，可以看做一个特殊的容器空间

![img](https://edu.aliyun.com/files/course/2021/04-06/11064993e08f169449.png)

## 网络方案

#### Flannel

![img](https://edu.aliyun.com/files/course/2021/04-06/110743fc63ad493704.png)

Flannel 方案是目前使用最为普遍的。如上图所示，可以看到一个典型的容器网方案。它首先要解决的是 container 的包如何到达 Host，这里采用的是加一个 Bridge 的方式。它的 backend 其实是独立的，也就是说这个包如何离开 Host，是采用哪种封装方式，还是不需要封装，都是可选择的。

现在来介绍三种主要的 backend：

* 一种是用户态的 udp，这种是最早期的实现；
* 然后是内核的 Vxlan，这两种都算是 overlay 的方案。Vxlan 的性能会比较好一点，但是它对内核的版本是有要求的，需要内核支持 Vxlan 的特性功能；
* 如果你的集群规模不够大，又处于同一个二层域，也可以选择采用 host-gw 的方式。这种方式的 backend 基本上是由一段广播路由规则来启动的，性能比较高。

#### Calico

提供 BGP 网络直连

#### Canal

Flannel + Calico

#### WeaveNet

UDP 封装 L2 Overlay，支持用户态 / 内核态 两种实现

## 服务发现

### 基础内容

![img](https://edu.aliyun.com/files/course/2021/04-06/111018a9d728049128.png)

这个示例中我们定义的是 TCP 协议，端口是 80，目的端口是 9376，效果是访问到这个 service 80 端口会被路由到后端的 targetPort，就是只要访问到这个 service 80 端口的都会负载均衡到后端 app：MyApp 这种 label 的 pod 的 9376 端口。

![img](https://edu.aliyun.com/files/course/2021/04-06/11105603a4ef876654.png)

### 集群内访问 Service

在集群里面，其他 pod 要怎么访问到我们所创建的这个 service 呢？有三种方式：

* 首先我们可以通过 service 的虚拟 IP 去访问，比如说刚创建的 my-service 这个服务，通过 kubectl get svc 或者 kubectl discribe service 都可以看到它的虚拟 IP 地址是 172.29.3.27，端口是 80，然后就可以通过这个虚拟 IP 及端口在 pod 里面直接访问到这个 service 的地址。
* **第二种方式直接访问服务名，依靠 DNS 解析**，_**就是同一个 namespace 里 pod 可以直接通过 service 的名字去访问到刚才所声明的这个 service**_。不同的 namespace 里面，我们可以通过 service 名字加“.”，然后加 service 所在的哪个 namespace 去访问这个 service，例如我们直接用 curl 去访问，就是 my-service:80 就可以访问到这个 service。
* 第三种是通过环境变量访问，在同一个 namespace 里的 pod 启动时，K8s 会把 service 的一些 IP 地址、端口，以及一些简单的配置，通过环境变量的方式放到 K8s 的 pod 里面。在 K8s pod 的容器启动之后，通过读取系统的环境变量比读取到 namespace 里面其他 service 配置的一个地址，或者是它的端口号等等。比如在集群的某一个 pod 里面，可以直接通过 curl $ 取到一个环境变量的值，比如取到 MY\_SERVICE\_SERVICE\_HOST 就是它的一个 IP 地址，MY\_SERVICE 就是刚才我们声明的 MY\_SERVICE，SERVICE\_PORT 就是它的端口号，这样也可以请求到集群里面的 MY\_SERVICE 这个 service。

### Headless Service

服务发现不需要申请集群内的IP地址，pod 通过 service\_name 解析到对应的 pod IP

![img](https://edu.aliyun.com/files/course/2021/04-06/1112215e2648446658.png)

### Service类型

* ClusterIP
* ExternalName
* NodePort
* LoadBalancer

#### ClusterIP

Exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster. This is the default ServiceType

#### NodePort

Exposes the service on each Node’s IP at a static port (the NodePort). A ClusterIP service, to which the NodePort service will route, is automatically created. You’ll be able to contact the NodePort service, from outside the cluster, by requesting `<NodeIP>:<NodePort>`.

#### LoadBalancer

Exposes the service externally using a cloud provider’s load balancer. NodePort and ClusterIP services, to which the external load balancer will route, are automatically created.

### 服务发现原理

![img](https://edu.aliyun.com/files/course/2021/04-06/112745100947316833.png)

### 思考

目前系统有配置 network policy 吗？

现在用的是 fannel？

kind 有默认安装的网络插件吗？ -- kindnet

怎么查看当前 cluster 使用的网络插件

sidecar proxy vs kernel proxy -- 新的 islto 网络方案 （ebpf)
