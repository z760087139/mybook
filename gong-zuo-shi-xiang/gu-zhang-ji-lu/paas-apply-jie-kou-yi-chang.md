---
description: 2023/4/26 caas 服务的 apply接口并发异常记录
---

# PaaS Apply接口异常

## &#x20;故障背景

PaaS 2.0 上线准备中，需要对1.0的数据进行迁移，其中数据迁移的内容包括现有的 rolebinding 迁移动作，需要读取现有rancher 的RBAC 体系中的 projectRoleTemplateBinding 内容，在 PaaS 2.0 的ucenter 及 k8s 中创建对应的 user 及 rolebinding

## 故障现象

ucenter 发起 rolebinding 创建请求，caas 服务负责执行 apply 动作。在apply过程中，caas日志出现大量的 context deadline 的错误提示，以及部分的 connection rerset by peer 和 server recived too many requests and has asked us to try again later 提示

## 故障分析

connection reset by peer 及 server recived too many requests 的问题，推测是与 api-server 建立链接被拒绝，查看代码内容，发现 apply 动作是由 caas 服务对yaml文件内容进行拆分后，使用 go func() 进行并发请求。分析 caas 代码，发现该动作并没有控制最大并发数，因此增加对 apply动作的协程最大并发数

context deadline 错误一般是由于 kratos 框架在接收请求时候，每个请求都会创建对应的context，并设置 timeout。把该 context 传递到后续的程序代码中，当出现context检查时候则可能出现context超时。故障时是由于长时间调用 caas 功能，发起大量 rolebinding 的 apply 动作引发。
