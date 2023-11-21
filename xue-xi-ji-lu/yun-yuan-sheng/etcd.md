# ETCD

以下是 etcd 中存储的 kubernetes 所有的元数据类型：

```ini
ThirdPartyResourceData
apiextensions.k8s.io
apiregistration.k8s.io
certificatesigningrequests
clusterrolebindings
clusterroles
configmaps
controllerrevisions
controllers
daemonsets
deployments
events
horizontalpodautoscalers
ingress
limitranges
minions
monitoring.coreos.com
namespaces
persistentvolumeclaims
persistentvolumes
poddisruptionbudgets
pods
ranges
replicasets
resourcequotas
rolebindings
roles
secrets
serviceaccounts
services
statefulsets
storageclasses
thirdpartyresources
```

## 查询方式

etcdctl get \<resource\_key> --prefix --limit 1 --keys-only -w=json

\--prefix 根据 key 的 prefix 匹配结果

\--limit 限制返回

\--keys-only 不返回 value 内容

\-w 返回格式

### 元数据

etcdctl get /registry/\<resource>/\<namespace>/\<resource\_name>

eg: etcdctl get /registry/deployment/default/nginx -w=json

### CRD类型

etcdctl get /registry/apiextensions.k8s.io/customresourcedefinitions/\<resource>

eg: etcdctl get /registry/apiextensions.k8s.io/customresourcedefinitions/apprevisions.project.cattle.io

### CR数据

etcdctl get /registry/\<group>/\<resource>/\<namespace>/\<resource\_name>

eg : etcdctl get /registry/project.cattle.io/apprevisions/p-xxx/apprevision-xxxx -w=json
