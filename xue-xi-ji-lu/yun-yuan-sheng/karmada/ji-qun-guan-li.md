# 集群管理

## 管理模式

### 集群纳管

karmadactl join clusterName --cluster-kubeconfig=xxxx

### push 模式

控制面将业务操作直送到业务集群

### pull 模式

在业务集群部署 karmada-agent，agent 从 控制面获取操作内容并执行

#### agent 功能

1. 注册集群信息
2. 监听集群状态
3. 监听、执行控制面下发的资源内容，并将信息进行上送

## 集群链接

### 信息维护

cluster.spec.apiEndpoint 字段维护业务集群的 api-server 地址

cluster.spec.secretRef.namespace / cluster.spec.secretRef.name 保存业务集群链接使用的 secret kubeconfig&#x20;

通过 secret\[token] 读取 rest.Config BeaerToken 内容，与 apiEndpoint 组成 rest.Config&#x20;

cluster.spec.InsecureSkipTLSVerification 控制是否开启 TLS

cluster.spec.proxyURL 设置代理地址

## 集群健康检查

### 集群状态

karmada controller 里面创建了 clusterStautsController 进行集群状态检查

通过 /readyz 接口请求，检查 httpcode 状态，如果接口不可用，会尝试 /healthz 接口

集群状态会记录到 cluster.status.conditions 字段中

| 请求结果                                           | condition.Type | condition.Reason    |
| ---------------------------------------------- | -------------- | ------------------- |
| <p>请求接口异常<br>(api-server不可达)</p>               | Ready          | ClusterNotReachable |
| <p>接口请求 http code 非200<br>(api-server返回异常)</p> | Ready          | ClusterNotReady     |
| 接口请求200                                        | Ready          | ClusterReady        |

### 健康检查机制

{% hint style="info" %}
cluster-status-update-frequency 参数设置检查间隔，即下述延时入队的时间
{% endhint %}

karmada 通过 controller-runtime manager 实现 clsuterstatuscontroller 的管理

controller-runtime manager 大致有以下内容

* Queue
* Reconciler interface  &#x20;
* watcher

manager 将 Reconciler 转换成多个 worker 并进行管理， 由 worker 抢占 Queue 的内容进行处理

Reconciler 通过 Result 进行结果返回，并由 manager 对结果进行处理

manager 对 Reconcil 结果处理

| reconcil 结果         | 处理方法                 |
| ------------------- | -------------------- |
| TerminalError       | 日志、metrics 记录，不重新入队  |
| 其他异常                | 日志、metrics 记录、立即重新入队 |
| result.RequeueAfter | 延时入队                 |
| result.Requeue      | 立即重新入队               |
| 默认处理                | 从队列中删除               |

Reconcil 场景返回结果

| 场景                         | cluster.status.condition             | Queue |
| -------------------------- | ------------------------------------ | ----- |
| 无法从 api-server 获取集群        | 不更新                                  | 从队列删除 |
| api-server 找不到对应集群         | 删除缓存及 informer 内容，删除 lease           | 从队列删除 |
| 业务集群 client set 无法连接       | condition.Reason StatusCollectFailed | 延时入队  |
| 等待 cluster controller 执行完成 | 不更新                                  | 立即入队  |
| 离线且健康检查失败                  | 更新状态                                 | 延时入队  |
|                            |                                      |       |

补充内容：检查结果如果未达到成功健康检查或失败健康检查持续时间，将不会产生新的
