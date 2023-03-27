# Deployment

* 监控组件
* 监控应用
* 管理组件、应用日志
* deployment 升级 回滚
* 弹性伸缩、自动恢复

### 集群状态

```shell
kubectl cluster-info dump
```

### 组件 metrics

```shell
curl localhost:10250/stats/summary
curl localhost:10250/healthz
```

监控组件

heapster + cAdvisor

kubectl top node

收集 cpu 、 内存

### 修改deployment

```shell
kubectl set image deployment/nginx-deployment
kubectl set resources deployment/nginx-deployment --limits=cpu=200m,memory=512Mi
```

### 滚动升级

deployment.strategy

rollingUpdate.maxSurge / maxUnavailable

### 暂停、恢复deployment

// 不会记录期间修改内容

kubectl rollout pause deployment/nginx-deployment

kubectl set image ...

kubectl rollout resume deployment/nginx-deployment

### deployment 历史及回滚

kubectl rollout history deployment/nginx (--revision=2)

kubectl rollout undo deployment/nginx-deployment --to-revision=2

### 弹性伸缩

kubectl scale deployment/nginx --replicas=10 // 修改replicas

kubectl autosacle deployment/nginx --min=10 --max=15 --cpu-percent=80 // cpu 使用率80 扩容

### 应用自恢复

restartPolicy + livenessProbe

#### RestartPolicy

Always / OnFailure / Never

#### LivenessProbe

http / https Get, shell exec , tcpSocket

```yaml
pod:
	spec:
		restartPolicy: Always
		containers:
			- name: nginx
				livenessProbe:
					tcpSocket:
						port: 8080
						initialDeplaySeconds: 15 // 初始化延迟 15 秒
						periodSeconds: 20 // 每 20 秒
```
