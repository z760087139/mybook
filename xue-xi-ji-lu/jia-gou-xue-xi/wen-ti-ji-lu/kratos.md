---
description: kratos 使用过程及业务代码结构问题记录
---

# Kratos

## mock

使用 mockery 生成 mock 类型，在 biz 层由于依赖倒置的方式，无法将mock分成独立的package

```shell
# 生成例子
/app
 |- biz
    |- mock
       |- mock_biz.go
    |- biz.go
    |- biz_test.go
```

biz 定义了 Repo interface ，并直接在 biz 层实现interface 的调用。此时将实现的实例放在 mock package ，同时在 biz 编写 test文件，会出现依赖循环问题 test(biz) -> mock(mock) -> interface(biz)

### **解决方法**

#### **方法一**

​ 将mock 文件在 biz 同级目录生成，不作为独立package 存放

​ `mockery --all --inpackage`

#### **方法二**

​ 将mock / test 文件独立存放， 但无法测试非导出函数或者方法

## Transaction

微服务设计过程经常存在领域、聚合的划分不清晰问题，在 kratos 体现为数据整合、事务的实现

假如存在一个 API 接口，提供用户信息

```protobuf
Message UserInfo {
	string UserName = 1;
	string RoleName = 2;
	repeated string RolePermissions = 3;
}
```

用户、角色、绑定信息 为三个聚合根，API提供整合后的数据格式。此时应放在 DTO 或 DO 层进行处理
