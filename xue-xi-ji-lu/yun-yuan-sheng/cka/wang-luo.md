# 网络

## 大纲

Pod 网络

CNI

Service 概念

load balance

Ingress

DNS

### Pod 网络

* 每个 Pod 一个独立IP，Pod 内容器共享 network namespace
* 容器之间直接通信，不需要NAT
* Node 和 容器直接通信，不需要NAT
* 其他容器和容器自身看到的IP一致
* 集群内访问走 services, 集群歪访问走 ingress
* CNI 用于配置 pod 网络
  * 不支持 docker 网络

!\[image-20220712004434748]\(/Users/hanhui/Library/Application Support/typora-user-images/image-20220712004434748.png)

flannel / calico 用于pod 在不同节点时候的通信

#### CNI(container network interface)

容器网络标准化

使用JSON描述网络配置

* 两类接口
  * 配置网络 - 创建容器时调用
    * AddNetwork(net NetworkConfig, rt RuntimeConf) (types.Result, error)
  * 清理网络 - 删除容器时调用
    * DelNetwork(net NetworkConfig, rt RuntimeConf) error

**作用**

CNI 包括用于 Pod 之间跨主机网络通信使用，是 flannel / calico 等网络方案

容器内部的网络方案是 bridge/ veth / p2p 等

!\[image-20220713002802880]\(/Users/hanhui/Library/Application Support/typora-user-images/image-20220713002802880.png)

**使用**

**default plugin**

```shell
cat /etc/cni/net.d/xx.conf
{
	"name":"xxx",
	"type":"bridge",
	"ipam": {
		"type":"host-local",
		"subnet":"10.10.0.0/16"
	}
}
# 二进制程序 非 flannel caclico
/opt/cni/bin/{host-local, bridge...}
```

### Service

#### 原理

service 通过 label selector 与 pod 进行关联

pod runtime and ready 后，会生成 endpoint 信息

k8s 根据 endpoint 信息添加到 DNS中

#### 结构

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: redis
  name: redis
  namespace: default
spec:
  clusterIP: 10.96.50.213
  clusterIPs:
  - 10.96.50.213
  ports:
  - port: 6379 // 对外接口
    protocol: TCP
    targetPort: 6379 // 目标容器端口
  selector:
    app: redis
  type: ClusterIP
  // LoabBalcne 部分内容
  loadBalancerIP: 78.11.24.19 // 外部LB IP (F5 / NGINX IP)
  // type 改为 LoabBalancer
```

#### LoadBalance

同时是 ClusterIP 类型

需要在特定 cloud provider 上用

yaml 文件见上个例子 LoadBalance部分

* Service Controller 自动创建一个外部LB 并且配置为安全组
* 对集群内的访问， kube-proxy 用 iptables 或者 ipvs 实现 cloud provider LB部分功能，主要是4层转发，7层 session cookie header 类的不支持

**创建命令**

```shell
kubectl create service clusterip my-svc --tcp=80:8080
kubectl create service nodeport my-svc-np --tcp=1234:80
kubectl create service headless my-svc-hl --clusterip="None"

kubectl expose deployment hello-nginx --type=ClusterIP --name=hello-nginx --port=8090 --target-port=80
```

#### Ingress

* 集群外访问内部网络的入站授权的规则合集
  * 支持URL将 Service 暴露到K8S集群外，是7层访问入口
  * 支持自定义 Service 访问策略
  * 提供域名访问的虚拟主机功能
  * 支持TLS

!\[image-20220713013214040]\(/Users/hanhui/Library/Application Support/typora-user-images/image-20220713013214040.png)

**yaml内容**

```yaml
ingress:
	spec:
		tls:
		- secretName: testsecret // tls 密文内容，指向 secret testsecret读取内容
    backend:
    	// ingress 对应的 service 信息
    	secretName: testsvc
    	servicePort: 80
    rules:
    - host: api.company.com
    	http:
    		paths:
    		- path: /foo
    			backend: 
    				serviceName: s1
    				servicePort: 80
        - path: /bar
        	backend:
        		serviceName: s2s
    				servicePort: 80
```

#### kubernetes DNS 必考

* 解析 pod 和 services 的域名， 仅k8s 集群内 pod 使用 -- kubelet 配置 cluster-dns 将 DNS 静态IP 传递到每个容器 resouce.conf 文件
* kube-dns / core-dns
* services
  * A 记录
    * 普通 services： my-svc.my-namespace.svc.{ClusterIP} // kubelet 传入， --cluster-domain 配置的伪域名
    * headless services: my-svc.my-namespace.svc.{PodIP} // {PodIP} 为后端 pod ip 列表
  * SRV 记录 // 查 service 端口用的
    * `_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster.local -> {service port}`

```shell
kubectl get svc -n kube-system // 获取 kube-dns 信息
```

### 疑问

网络模型，4层 7层内容具体是什么

service controller 如何实现配置

flannel / calico CNI如何使用

overlay / underlay 是什么 如何实现

service / ingress / loadbalance 如何交互

kubernetes DNS 怎么用
