# 参与项目

## rancher 二次开发

* 资源申请入口
* 调整项目、命名空间资源配额

​ 与外部系统对接，根据工单内容推荐最佳资源分配集群，并根据工单内容，创建、调整命名空间

* 用户授权（RBAC资源对象）

​ 与外部系统对接，根据工单内容将用户、命名空间、角色进行绑定，完成k8s 的RBAC 信息创建

* BigIp / virtual Cluster 创建

​ 行内的高可用方案要求，需要实现集群的高可用，因此将 deployment 、pod 信息通过 F5 CC 与 configmap 的交互，实现VIP 与 pod IP 绑定功能。其中还需要通过一个 virtual Cluster 进行流量中转

* agent 编写

​ 为了实现多集群宿主机管理，通过 daemonset 在所有集群的宿主机上部署 agent ，并与 rancher 进行通信，实现宿主机的管理，包括证书下发，文件更新

* SA 权限管理

​ 针对行内应用所使用的的资源调度，比如模型训练。允许用户在授权范围内自行创建SA，并针对 rancher 的权限管理进行调整，允许任何人在 rancher 平台录入SA信息。

* 自定义CRD资源及 controller
  *   VIP功能

      为了解决高并发下的冲突问题，采用了 controller 形式实现vip 资源分配及重试。实现与F5的交互管理VIP信息
  *   Appsystem

      行内应用信息记录，在 rancher 上实现行内应用信息的记录及管理。其中与 rancher 的项目资源实现交互，经常需要使用 indexer 进行资源索引及关联
  *   rancher.cluster

      通过 cluster annotation 形式标记 cluster 信息，并在日常维护中使用
  *   project role template binding

      为了扩充 rancher 默认的权限模型，在rancher 的 prtb CRD 资源上追加了 controller 内容。针对某些角色绑定由controller 自动追加额外的角色绑定
  *   f5loadbalance

      CRD资源，记录 F5 VIP信息，通过 controller 创建并维护 configmap内容，提供CC获取并更新到F5中。后由于管理方式调整，废弃该CRD资源，并对 controller 内容进行作废

​

​ 关于 rancher lifecycle 及 finalize 问题

​ 由于 rancher 调用 lifecycle 会在资源的 finalize 上追加 controller 信息，导致 controller 一旦创建则无法删除，否则会令资源找不到 finalize 无法删除。因此一般不采用 lifecycle 形式，即 finalize 形式的controller

## DevOps 平台建设

​ 多组件构建 (CI)

​ 配合行内 CICD 流程，通过 MQ 实现CI 任务的启动、监听、重试等流程机制。主要通过 MQ 获取初始化或者构建中的任务列表，通过主节点形式，监听k8s ci job 的状态并更新 MQ 信息。并且通过主节点形式实现节点消亡后的任务重试效果

​ 多组件部署(CD)及状态校验（rollout status源码）

​ 配合行内的环境，实现多资源、多集群的部署功能（不涉及调度）。参考kubectl rollout status 源码，实时监听 deployment 状态信息，及时反馈部署是否存在失败问题及原因
