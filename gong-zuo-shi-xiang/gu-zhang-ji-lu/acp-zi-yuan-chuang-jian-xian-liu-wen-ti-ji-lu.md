# ACP 资源创建限流问题记录

批量创建、修改RB资源出现限流

## 问题背景

2.0系统上线前，需要对现有的权限进行迁移。过程中需要重新创建rolebinding信息，将新用户与角色进行重新绑定

## 问题现状

迁移 rolebinding 过程中出现执行缓慢的问题，大约需要重新创建 1W 条数据

## 问题分析

api-server 本身存在请求限流的设置，通过调整 api-server 的参数，加大限流仍无法解决

后续排查发现是 controller manager 也存在限流情况（为什么 rb 会引起 controller 限流仍在排查）需要调整 controller manager 的设置
