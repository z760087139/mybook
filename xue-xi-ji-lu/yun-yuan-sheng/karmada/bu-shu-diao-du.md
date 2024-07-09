# 部署调度

## 部署原理

在 karmada api-server 命名空间上创建 deployment ，配置调度策略 propagation policy 后，schedudler 根据策略进行分配，由 controller 实施部署

## Propagation Policy

{% embed url="https://github.com/karmada-io/karmada/blob/master/pkg/apis/policy/v1alpha1/propagation_types.go#L13" %}
propagation policy type
{% endembed %}

### propagtion spec

#### ResourceSelectors

预期部署、调度的资源对象（数组）

需要指定 gvk / namespace ，可以指定 label selector

#### PropagateDeps

可以将关联资源自动同步部署，比如 deployment 关联的 configmap / secret 内容会自动部署或者故障迁移， resourceSelectors 可以不配置 configmap / secret

默认关闭

#### Placement

集群选择规则&#x20;

**ClusterAffinity**

单组集群的定向调度，不能 ClusterAffinities 同时存在；如果两个都不存在，则任意集群都能调度

集群选择规则：

* LabelSeletor 根据标签选择，使用 k8s 原生的 labelSeletor 对象及其效果，多个匹配规则时，要求集群所有条件都满足
  * MatchLabel
  * MatchExpressions
* FieldSeletor   根据字段名过滤，支持 In, NotIn, Exists, DoesNotExist. Gt, and Lt，多个匹配规则时，要求集群所有条件都满足
* ClusterName 根据集群名过滤
* ExcludeCluster 过滤集群名单

**ClusterAffinities**&#x20;

内容是一样的，只是改为数组并增加了名字进行区分，匹配时会按顺序使用 ClustrAffinity 内容单独匹配，每个 ClusterAffinity 的结果互相不影响

**ClusterTolerations**  （待验证）

容忍规则，需要结合污点(taints)进行使用，需要通过 karmadactl 对集群打污点

```
karmadactl taint clusters foo dedicated=special-user:NoSchedule
```

目前仅支持 `NoSchedule` | `NoExecute`

**`NoExecute`** 还将会被使用到**故障迁移**里，需要谨慎使用

**SpreadConstraints**&#x20;

基于集群拓扑的调度，与指定规则之间如何处理冲突仍待测试

该功能主要实现最终落实到多少个地区/多少个集群进行部署

通过这个配置，可以实现HA效果，将资源强制分布在不同地（Enum=cluster;region;zone;provider），不同集群

比如，声明必须在两个地区部署，每个地区一个集群

```
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  #...
  placement:
    replicaScheduling:
     replicaSchedulingType: Duplicated  
    spreadConstraints:
      - spreadByField: region
        maxGroups: 2
        minGroups: 2
      - spreadByField: cluster
        maxGroups: 1
        minGroups: 1
```

使用 spreadByField 的时候，必须有一个条件是 cluster

**replicaScheduling**

副本调度规则

**ReplicaSchedulingType**&#x20;

两种调度规则：&#x20;

* Duplicated 所有集群采用相同的副本数
* Divided       根据下述配置信息进行调配

**ReplicaDivisionPreference**

Divided 模式的调配规则，配合 clusterAffinity 进行集群选择后进行分配

* Aggregated   尽可能少的分配到每个集群（不太懂）
* Weighted       权重分配，包括静态权重（固定集群权重），动态权重两种

动态权重会根据集群最大可分配资源作为权重进行配平，比如 A/B/C三个集群最大可部署 6/12/18 deployment，预期部署 12个deployment，则权重为 1:2:3，最终部署为 2/4/6

采用动态权重后，静态权重将会失效

#### Priority

策略优先级，数字越高，优先级越高

如果资源模板已经被匹配和绑定，则后续即使有更高的优先级策略，也不会受影响（具体参考Preemption 待补充）

同级抢占，则以精准度优先，name > seletor > gvk

#### Preemption

是否抢占，枚举值为 Always / Never

#### DependentOverrides

执行部署前，需要先执行的 overrides 列表，即使执行了，后续部署时也会按照各个集群进行 overrides 覆盖，两者不冲突

#### Failover

故障迁移（重点 待补充）

#### ConflictResolution

资源已经存在于目标集群中时应如何处理潜在冲突 （部署场景之一）

#### ActivationPreference

策略变动时，是否执行下发

默认情况为立即下发

Lazy 则更新策略后，等待资源模板更新的时候才用新的策略进行下发
