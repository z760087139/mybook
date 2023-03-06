# ACP程序崩溃、缓慢

## 现象

无论前后端，在部分资源加载过程中，会出现部分资源加载过慢导致返回时间较长的情况。

## 排查过程

1. 查看资源监控，发现内存使用量非常高
2. 根据client-go 的informer 原理，推测是部分资源量过多
3. 统计ETCD 数据，发现 projectroletemplatebinding / rolebinding数量异常

## 问题本质

使用规划存在问题。原设计采用 1 组件/命名空间 进行资源规划，导致存在大量的命名空间。

而根据 k8s RBAC 的设计，rolebinding 以全局/命名空间为维度划分，导致每个用户都需要重复绑定多个命名空间的权限

同时，rancher 通过 projectroletemplatebinding controller 也在确保 rolebinding 的存在，但该逻辑存在未知缺陷，导致大量的重复绑定数据出现

## 解决办法

- 清理 etcd 上的重复数据
- 排查 rancher 重复创建的逻辑错误原因并修复