# 开发流程

## 环境配置

### golang 版本

\> 1.17

### 安装 kratos 工具

```bash
go get -u github.com/go-kratos/kratos/cmd/kratos/v2@latest
```

## 项目代码开发顺序

kratos new projectName -r https://gitee.com/go-kratos/kratos-layout.git

会在当前目录从指定地址下载项目模板内容

### 生成 .proto

#### 快速生成初始模板

```bash
kratos proto add api/helloworld/demo.proto
```

#### 修改 .proto 内容

参考 样例代码 policy.proto

#### 生成 http/ grp .go 文件

_# 可以直接通过 make 命令生成_ make api

_# 或使用 kratos cli 进行生成_ kratos proto client api/helloworld/demo.proto

### 生成 service代码

使用 `-t` 指定生成目录

```bash
kratos proto server api/helloworld/demo.proto -t internal/service
```

### 编写 biz 层代码

注意调整引入库

### 编写 db 层代码

注意引入倒置， db 层对外提供的是符合 biz 层定义的 RepoInterface 及 数据结构

## 目录层级

### cmd

#### main.go

* 启动入口
* 初始化各类的配置信息
* 服务名定义
* http/grpc 协议支持定义
* 其他 kratos 配置项 （微服务版本 metadata endpoint 等）

#### wire.go

根据 data, biz, service, server 提供的 privoer 内容，生成 wire\_gen 的代码注入

#### wire\_gen.go

由 wire.go 生成，用于组装 data, biz, service, server 之间实例化、调用逻辑

* 提供给 main.go 使用，将插件内容进行组装，比如启动 注册中心、grpc服务、http服务、日志、验证等
* 初始化其他服务的 client (见 beer-shop 示例的 shop.cmd.wire\_gen userClient,cartClient)

### internal

#### data

数据库操作层，将多个数据库驱动封装在 data 层，提供 biz 层调用

实现最基础的数据操作动作(CRUD)，或者部分针对数据进行的简单逻辑（biz.Struct 结构转换、数据对比、缓存获取，数组组装）

_**严禁对外提供 internal.Model 内容**_，对外提供的数据结构必须为 biz.Struct 形式

#### biz

业务逻辑层

定义 biz.RepoInterface biz.Strcut

实现依赖倒置， biz 定义 data层需要提供符合使用的 interface，以及对应的 struct 结构

#### 需要关注的案例

[upload](https://github.com/go-kratos/examples/blob/main/http/upload/main.go)

[websocket](https://github.com/go-kratos/examples/tree/main/ws)
