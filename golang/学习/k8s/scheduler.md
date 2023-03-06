#### 核心阶段

##### 调度阶段 scheduler cycle 

- predicate 过滤、预选

- priority 优先级

###### example

pkg/scheduler/framework/plugins/noderesources/fit.go

- prefilter

  计算所需资源

- filter

  计算满足需求的节点

- score

  计算得分，根据用户分配策略调整优先级

  LeastAllocated / MostAllocated ...

##### 绑定阶段 bnding cycle



limit -> cfs_quota

request -> cpu.share

period/quota 流量控制  period=10000000us



nodeaffinity

requiredDuringScheduling  required 作用于 predicate 阶段

preferredDuringScheduling preferred 作用于 priority 阶段

ignoredDuringExecution execution 为运行中，如果节点出现标签变动，是否需要重新调度



taints 污点 / tolerations 

noschedule 新 pod 不调度到当前节点

preferNoSchedule 尽量不调度到 node

NoExecute node 上正在运行的 pod 都驱逐，并且不能调度

tolerations 作用于pod，允许调度到 taints node



descheduler 重调度器



拓展调度器

extender 注册外部 webhook 拓展默认调度器 存在性能问题，拓展点数量有限

基于 scheduler framework 实现拓展

 

crane-scheduler



拓扑感知调度

node 单机 cpu 调度时的缓存访问，进程尽量分配在一个cpu上完成，减少cpu对缓存的重分配

