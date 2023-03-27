# kubeconfig

kubeconfig 内容分为三部分

* 集群信息
* 用户信息
* 上下文

### 集群信息操作

#### 修改集群信息

kubectl config set-cluster

该动作可以修改 kubeconfig 中的集群信息

```shell
kubectl config --kubeconfig=config-demo set-cluster development --server=https://1.2.3.4 --certificate-authority=fake-ca-file  --insecure-skip-tls-verify
```

#### 删除集群信息

```shell
kubectl --kubeconfig=config-demo config unset clusters.<name>
```

### 用户信息

#### 创建/修改用户信息

kubectl config set-credentials

```shell
kubectl config --kubeconfig=config-demo set-credentials developer --client-certificate=fake-cert-file --client-key=fake-key-seefile
```

#### 删除用户信息

```shell
kubectl --kubeconfig=config-demo config unset users.<name>
```

### 上下文

将集群、命名空间、用户信息 组合形成的上下文内容

#### 创建上下文

```shell
kubectl config --kubeconfig=config-demo set-context dev-frontend --cluster=development --namespace=frontend --user=developer
```

创建时候需要指定之前定义的集群、用户名称

#### 切换上下文

由于kubeconfig可以同时记录多个集群、用户信息，组合成多个上下文内容。

kubectl config set-context

通过切换上下文，改变kubectl 链接集群、用户信息
