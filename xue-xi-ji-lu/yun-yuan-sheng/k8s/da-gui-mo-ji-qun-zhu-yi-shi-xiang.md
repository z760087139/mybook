# 大规模集群注意事项

官网说明地址

https://kubernetes.io/docs/setup/best-practices/cluster-large/

### 官网摘选

1. 每个节点的 Pod 数量不超过 110
2. 节点数不超过 5,000
3. Pod 总数不超过 150,000
4. 容器总数不超过 300,000
5. ETCD的高可用方案

* 请求增加云资源的配额，例如：
  * 计算实例
  * CPU
  * 存储卷
  * 使用中的 IP 地址
  * 数据包过滤规则集
  * 负载均衡数量
  * 网络子网
  * 日志流
