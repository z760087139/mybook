# management

## 静态 pod

ps -elf | grep kubelet

获取静态文件路径位置 `/etc/kubernetes/manifest` 在路径添加 yaml 文件即可在 master 节点 kubelet启动时自动启动其他静态yaml

## 管理node 节点组件

kubelet / kube-proxy

kubeadm join .... 将节点添加到集群

`kubectl describe node xx # 查看节点运行状态`

追加加点后，需要安装 CNI (比如 flannel)

## KUBECONFIG

### 修改cluster

```shell
kubectl config set-cluster xxx
	--certificate-authority=/path/ca
	--embed-certs=true # 是否记录 ca 到 kubeconfig
	--server=${API-SERVER}
	--kubeconfig=.kube/config
```

### 修改user

```shell
kubectl config set-credentials xxx
	--client-certificate=/path/ca
	--client-key=/pat/private-key
	--embed-certs=true
	--kubeconfig=.kube/config
```

### 修改context

```shell
kubectl config sset-context default
	--cluster=xxx
	--user=xxx
```

### 设置默认context

```shell
kubectl config use-context default
```

## 节点证书签发

kubelet 启动时使用低权token `--bootstrap 属性` 访问 api-server 发送 csr 请求，由 kube-controller-manager 自动签发证书，交由api-server 进行返回

低权角色权限

​ nodeclient: 签发证书

​ selfnodeclient: 更新证书 -- kubelet 访问 server 的证书

​ selfnodeserver: 更新 kubelet server 证书 -- kubelet 监听 1050 端口 kubectl exec 这种进入容器的命令，实际需要通过 api-server 发送到 kubelet 进行转发访问容器内

## 高可用

### ETCD

采用三个节点部署 etcd 集群

#### 数据备份

```shell
```

#### 数据恢复

```shell
```

### API-SERVER

多实例部署，作为 endpoint 挂载到 loadbalance 后面

### controller-manager / scheduler

采用选主形式，多节点部署。通过 LB 访问 api-server

## 集群升级

### master升级

1. 升级 kubelet
2. 通过 manifest 更新组件

### worker 升级

kubectl drain xxx --ignore-daemonset=true // 驱逐节点

kubectl uncordon // 恢复驱逐节点

## 问题

自动签发证书具体如何实现

如何进入 api-server 命令行模式
