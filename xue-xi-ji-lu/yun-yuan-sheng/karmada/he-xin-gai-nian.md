# 核心概念

架构图

<figure><img src="../../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

### karmada-apiserver

跟 api-server 一直，提供相同的能力，但是 webhook 使用 karmada-webhook，并且会从 karmada-aggregated-apiserver 获取高级功能信息

### karmada-aggregated-apiserver <a href="#karmada-aggregated-apiserver" id="karmada-aggregated-apiserver"></a>

api 聚合层， [Kubernetes API Aggregation Layer](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) 实现api-server 的能力拓展，karmada 主要用于 cluster.status / cluster.proxy 上面

### kube-controlloer-manager

摘选的部分 k8s 原生 controller 内容

### karmada-controller-manager

重点组件，自定义 CRD 及相关的 controller 内容，与 aggregated-apiserver 进行通信

1. Cluster Controller：将 Kubernetes 集群连接到 Karmada，通过创建集群对象来管理集群的生命周期。
2. Policy Controller：监视 PropagationPolicy 对象。当添加 PropagationPolicy 对象时，Controller 将选择与 resourceSelector 匹配的一组资源，并为每个单独的资源对象创建 ResourceBinding。
3. Binding Controller：监视 ResourceBinding 对象，并为每个带有单个资源清单的集群创建一个 Work 对象。
4. Execution Controller：监视 Work 对象。当创建 Work 对象时，Controller 将把资源分发到成员集群。

### karmada-scheduler

重点组件，将 Kubernetes 原生API资源对象（以及CRD资源）调度到成员集群

### karmada-agent

agent 部署到各个管理集群中，通过 pull 形式将工作负载清单从 Karmada 控制平面同步到成员集群；也负责将成员集群及其资源的状态同步到 Karmada 控制平面

## 资源概念

### 资源模板(Resource Template)

K8S 原生API定义的联邦资源模板，方便用于对接各种 k8s 工具使用

### 调度策略(Propagation Policy)

定义多个集群之间的调度要求

1. 定向调度，通过 clusterName, Label, field 指定调度内容
2. 基于 taint / toleration 调度
3. spreadConstraint 基于集群拓扑的调度?
4. repolicasScheduling 针对 replicas 复制模式与拆分模式

### 差异化策略(Override Policy) <a href="#override-policy" id="override-policy"></a>

不同集群之间可能存在部分差异化内容需要调整

1. 镜像差异化配置
2. 运行参数差异化
3. 运行命令差异化
4. 自定义差异化

<figure><img src="../../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>



