# 面试题目

1. 对k8s 网络及组件相关了解较多，通过 controller 实现了相关业务逻辑，但是client-go相关知识了解不深入，不了解 informer / deltaFIFO 原理
2. go语言开发方面，知道上下文传递及协程管理控制；对锁有基本的认知，但对于死锁、原理方面了解不够。关于切片有一定的了解，但是对于go语言的值传递方式了解不足。知道通过 pprof 工具进行性能排查，实践经验较少
3. 不了解关系型数据库
4. 沟通能力不错

golang 语言原理了解比较基础，不清楚 golang 如何实现 map 扩容；chan 的堵塞、入队出队原理了解比较基础；对于 锁 的机制了解一般，不了解锁的原理，但是能自己总结规避死锁的方式；并发编程能准确说出上下文管理方式，但是对于相关工具包了解不够。

对于分布式事务的实现个人也说经验较少，只有一个相关流程经验，整体设计思路能简单描述但不详细。

性能监控和故障排查有参与过，但是当时能列举的例子和案例没能体现出在这块排查的能力。

整体开发水平评价为能够独立进行设计和开发，但是对golang语言的了解并不是非常深入

描述GPM模型 如何调度g，什么情况下当前运行的g不能被调度 同步包有哪几个模块 描述互斥锁实现原理 docker实现原理 有哪些隔离namespace uts ipc pid mount network user systemcall kubernetes组件 service原理 service涉及哪些资源和组件 pause的作用 istio架构，有哪些组件，执行流（描述应用端即可），业界如何减少网络损耗 prometheus原理 多集群下如何集中获取数据，thanos原理 大集群下如何获取数据，hash原理或kvass原理 kubernetes crd运行原理，主要涉及哪些组件，高可用时如何进行选主 CICD流程 API网关选型 如何进行网络选型 rpc应用网络问题 电商秒杀场景的痛点，如何解决

Container Runtinme Interface CRI Container Network Interface CNI Container Service Interface CSI

DNAT SNAT

golang微服务架构、分布式事务处理

cgroup 描述

kubernetes 组件

kubernetes 编排原理 / 组件用法

CRD资源使用

service 类型、对外暴露

kuernetes 组件的网络通信原则 pod层 pod service 对外暴露

服务网格

GPM 模型

锁机制

map / slice 机制

类型的指针方法与值方法区别

并发编程时，如何管理协程的结束、中断

故障排查 、 优化

单元测试

开发过程问题

日志-问题排查

数据库事务级别

脏读，可重复读，幻读含义及解决方式

分布式事务

kube operator

CSI

CNI
