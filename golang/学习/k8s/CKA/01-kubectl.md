# kubectl

### skill

#### 资源缩写

kubectl api-resource

#### 生成yaml

kubectl run --image=ngix my-deploy -o yaml --dry-run > my-deploy.yaml

kubectl get deployment foo -oyaml --export > new.yaml

kubectl explain pod.spec.affinity.podAffinity 

### Command

#### Basic

explain 

set 设置 feature

#### Deploy

rollout

rollout-update

scale -- 修改实例数

autoscale 配置自动伸缩

#### Cluster

top -- 资源使用率

cordon -- 标记 unschedulable

uncordon -- 标记 schedulabel

drain -- 驱逐节点上的应用

taint -- 修改 taint 标记

#### Troubleshooting

describe -- 查看详情

logs -- 查看 pod 容器 日志

exec -- 指定容器内执行命令

attach -- attach 到pod 内的容器

port-forward -- 为 pod 创建本地映射端口

proxy -- 为 API server 创建代理

cp -- 容器内外/容器见复制文件

#### Advanced 

apply

patch

replace 

convert -- 不同版本API之间转换对象定义

#### Setting

label -- 资源设置label

annotate -- 设置 annotation

completion -- 获取 shell 自动补全脚本 ？

#### Other

config 修改 kubectl 配置 kubeconfig 文件



#### 常见命令

```shell
// 生成 pod yaml
kubectl run nginx --image=busybox --dry-run=client -oyaml > pod.yaml 
// 修改 pod image (修改容器 nginx 的镜像为 nginx:latest)
kubectl set image deploy/nginx nginx=nginx:latest 
// 修改 replicas
kubectl scale deploy/nginx --replicas=3
// 暂停 deployment 历史变更记录
kubectl rollout pause deploy/nginx 
// 恢复 deployment 历史变更记录
kubectl rollout resume deploy/nginx
// 查看deployment 历史详情
kubectl rollout history deploy/nginx --revision=2
// 回滚
kubectl rollout undo deploy/nginx --to-revsion=1
```

