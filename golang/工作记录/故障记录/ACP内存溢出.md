# ACP 内存溢出排查记录

## 故障现状：
广州环境内存不断增长，最终内存溢出。同为广州环境的 rancher 运行正常  
广州环境采用南海etcd同步过去的数据  
南海环境运行正常  
## 故障分析：
采用 pprof/heap 导出故障环境的内存使用情况  
查看 inuse_space 情况，发现 inuse_space 达到15G内存  
Heap graph 中可以看到，内存的调用流程大致如下：  
```
setPeers -> toRecord -> AddHandler -> newController -> newProcessListener -> NewRingGrowing
```
通过 pprof 工具，加上 -source_path 参数，可以直接指向代码行数

(pprof) list setPeers

通过 pprof 工具，定位到关键函数的所在位置

setPeers 该函数每2分钟执行一次，获取etcd 上保存的集群信息并进行定时更新。

```go
go func() {
		for {
			select {
			case <-ctx.Done():
				return
			case <-time.After(5 * time.Second):
				if err := u.setPeers(nil); err == nil {
					time.Sleep(2 * time.Minute)
				}
			}
		}
	}()
```

peerSync 获取etcd 中的 cluster 列表信息

针对状态记录正常的集群，会进行 start 操作

``` go

func (u *userControllersController) peersSync() error {
	clusters, err := u.clusterLister.List("", labels.Everything())
	if err != nil {
		return err
	}
	var (
		errs []error
	)
	for _, cluster := range clusters {
		if cluster.DeletionTimestamp != nil || !v3.ClusterConditionProvisioned.IsTrue(cluster) {
			u.manager.Stop(cluster)
		} else {
			amOwner := u.amOwner(u.peers, cluster)
			if amOwner {
				metrics.SetClusterOwner(u.peers.SelfID, cluster.Name)
			} else {
				metrics.UnsetClusterOwner(u.peers.SelfID, cluster.Name)
			}
			if err := u.manager.Start(u.ctx, cluster, amOwner); err != nil {
				errs = append(errs, errors.Wrapf(err, "failed to start user controllers for cluster %s", cluster.Name))
			}
		}
	}
	return types.NewErrors(errs...)
}
```

Start 函数先查询缓存中记录的 sync.Map 是否已存在 cluster 实例化的记录，如果没找到则再次创建 toRecord

``` go
func (m *Manager) start(ctx context.Context, cluster *v3.Cluster, controllers, clusterOwner bool) (*record, error) {
	obj, ok := m.controllers.Load(cluster.UID)
	if ok {
		if !m.changed(obj.(*record), cluster, controllers, clusterOwner) {
			return obj.(*record), m.startController(obj.(*record), controllers, clusterOwner)
		}
		m.Stop(obj.(*record).clusterRec)
	}
	clusterRecord, err := m.toRecord(ctx, cluster)
	if err != nil {
		m.markUnavailable(cluster.Name)
		return nil, err
	}
	if clusterRecord == nil {
		return nil, httperror.NewAPIError(httperror.ClusterUnavailable, "cluster not found")
	}
	obj, _ = m.controllers.LoadOrStore(cluster.UID, clusterRecord)
	if err := m.startController(obj.(*record), controllers, clusterOwner); err != nil {
		m.markUnavailable(cluster.Name)
		return nil, err
	}
	return obj.(*record), nil
}
```
在 2.4.4 版本中， toRecord 的动作会进行 roleBinding/ clusterRoleBinding 的 controller 追加动作
``` go

func (m *Manager) toRecord(ctx context.Context, cluster *v3.Cluster) (*record, error) {
	kubeConfig, err := ToRESTConfig(cluster, m.ScaledContext)
	if kubeConfig == nil || err != nil {
		return nil, err
	}
	clusterContext, err := config.NewUserContext(m.ScaledContext, *kubeConfig, cluster.Name)
	if err != nil {
		return nil, err
	}
	s := &record{
		cluster:  clusterContext,
		clusterRec:  cluster,
		accessControl: rbac.NewAccessControl(ctx, cluster.Name, clusterContext.RBACw),
	}
	s.ctx, s.cancel = context.WithCancel(ctx)
	return s, nil
}
```
