# K8S调度管理

### 基础内容

1. pod 调度结果最终会记录到 `pod.spec.nodeName`,创建时为空，调度分配后k8s填充内容
2. node 可分配资源为 `status.allocatable.cpu` `status.allocatable.memory`
3. 由于一般会预留系统资源，`status.capacity`为预留前的资源，`status.allocatable`为预留后应用可分配资源
4. `pod.spec.container.resources`分别记录 `request`和`limit`信息
   1. 调度算法使用`request`总和， `limit`不影响调度
   2. initContainer取最大值，container 取累加值，最后取最大者 `max(max(initContainer.requests), sum(container.requests))`
   3. 未指定 request时，默认为0
5. `pod.spec.schedulerName`执行调度的调度器名称，默认为 `default-scheduler`
6. 高级调度策略设置 `pod.spec.nodeSelector` `pod.spec.affinity` `pod.spec.tolerations`
7.  node 资源模型

    ![节点容量](https://d33wubrfki0l68.cloudfront.net/41e928bf747cec2588ede80311938df06a0c7b54/a16b1/images/docs/node-capacity.svg)
8. 调度算法
   1. GeneralPredicates -- CPU、memory 是否满足 request
   2. LeastRequestedPriority -- pod 调度数量均衡算法
   3. BalancedResourceAllocation -- 平衡 cpu/mem 消耗比例 （pod cpu/mem 比例与 节点是否相似）

### k8s 高级调度

#### nodeSelector

​ 语法格式 map\[string]string

​ 全匹配 node 上的 label 信息进行分配

#### nodeAffinity (高级语法 nodeSelector)

* 引入运算符 in, NotIn (labelselector语法)
  * 支持枚举 label 可能的取值 zone in \[az1,az2]
  * 支持硬性过滤（多条件逻辑运算）和软性评分（条件权重）

```yaml
pod:
	spec:
		affinity:
			nodeAffinity:
				requireDuringSchedulingIgnoredDuringExecution: // 硬性过滤
					nodeSelectorTerms: // matchExpression 之间是 或 的关系
					- matchExpression:
						-	key: node-flavor
							operator: In
							values: // values 之间是 与 的关系
								- s1.large.2
         preferredDuringSchedulingIgnoredDuringExecution:  // 软性评分（权重）
           - weiht: 1
             preference:
               matchExpressions:
               - key: node-flavor
                 operator: In
                 values:
                   - s3.large.1
```

#### podAffinity

与 nodeAffinity 区别，是根据 已启动或者准备启动的 pod label 信息，找到这些pod 所在的 node 进行调度亲和

```yaml
pod:
	spec:
		affinity:
			podAffinity:
				requireDuringSchedulingIgnoredDuringExecution: // 硬性过滤
        - labelSelector: // matchExpression 只有与运算
            matchExpression:
            -	key: node-flavor
              operator: In
              values: // values 之间是 与 的关系
                - s1.large.2
          topologyKey: kubernetes.io/zone // 指定 pod 与目标 pod 分配级别,同个机架/同个az 等自定义 node 分组， value 对应 node label 上的key?
        preferredDuringSchedulingIgnoredDuringExecution:  // 软性评分（权重）
        - weiht: 1
         	preference:
            matchExpressions:
            - key: node-flavor
              operator: In
              values:
                - s3.large.1
```

#### podAntiAffinity 反亲和

与 podAffinity 结果相反即可

#### 手工调度

直接指定 nodeName 即可

#### Taints

节点属性，相当于特殊 label ,对 pod排斥性

`node.spec.taints`

NoSchedule 硬性排斥

PreferNoSchedule 软性排斥

```yaml
node:
	spec:
		taints:
		- effect: NoSchedule
			key: accelerator
			timeAdded: null // 超时驱逐
			value: gpu
```

// 追加 taint

kubectl taint node node-n1 foo=bar:NoSchedule

// 删除 taint

kubectl taint node node-n1 foo:NoSchedule-

#### Tolerations

允许调度到特定 taints Node 上

```yaml
pod:
	spec:
	  // 内容必须全部匹配才能调度
		tolerations:
		- key: accelerator
			operator: Equal // Equal 或 Exists
			value: gpu
			effect: NoSchedule
```

#### 调度结果

```shell
kubectl describe pod xxxx
```

#### 多调度器
