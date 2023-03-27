# 存储

#### 为什么要存储卷

* 临时数据，多容器共享
* 配置文件
* 持久化数据

#### 普通存储卷 volume

* configmap
* secret
* emptyDir
* hostPath
* nfs
* ...

#### 持久卷 persistent volume

抽象化 volume 信息，并将 PV 与 pod 生命周期隔离

**状态**

available -- 建完PV 可用装填

bound -- 与 PVC 进行关联

released -- PVC删除后，根据回收策略处理状态

failed

```yaml
spec:
	capacity:
		storage: 5Gi
	persistentVolumeReclaimPolicy: Recycle // 回收策略 Retained 保留/ Recycled 回收/ Deleted 删除
```

#### 持久卷申领 persistent volume claim

PV 资源的预期申请声明，表达的是用户对存储的请求。并由controller 自主分配 pv ，更常见的使用方式为 storage class 的资源申请和分配

#### PV PVC 生命周期管理

**静态制备**

管理员创建、声明 PV 信息，称为静态制备

**动态制备**

管理员创建 Storage Class 信息后，通过 PVC 声明动态创建

**绑定**

master controller 循环监听新建PVC 信息，并查找能与PVC 匹配的PV内容。PV 与 PVC 通过 ClaimRef 属性声明一对一关联关系

**保护机制**

pv / pvc 分别有对应的 finalize pv-protection / pvc-protection， 在 pvc / pod 还在与 pv / pvc 绑定或使用中的情况下，无法删除，pv / pvc 会处于 terminating 状态

**回收机制**

根据PV 设置的回收策略，会有不同的存储卷回收机制

无论哪种机制，已绑定过的PV无法再次与PVC进行绑定，必须要重新创建PV（即使内容相同）才能与PVC绑定

**保留 retain**

PV 与 PVC 解绑后，PV 会处于 released 状态。此时 PV 无法再与 PVC 进行绑定，仅作为记录提供管理员对使用后的存储数据进行处理

**删除 delete**

storage class 的默认回收策略，需要存储插件支持，在与PVC解绑后自动删除PV资源信息，并对 PV 对应的外部基础设备（AWS 、 OSS）中移除存储资产

~~**回收 recycle**~~

已作废，原用于对外部设备进行数据清理后重置为可绑定状态

**问题**

NAS OBS NFT SATA 分别是什么
