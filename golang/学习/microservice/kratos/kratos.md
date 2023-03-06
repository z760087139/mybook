## 数据统计

### issue

close 692 open 42

### PR

close 1122 open 8

### start

17k

## 快速启动

### 创建项目

kratos create project

#### 执行内容

kratos 到 github.com/go-kratos/kratos-layout 下载项目模板

### 启动项目

kratos run

#### 执行内容

查找当前路径下的 cmd 目录，并执行 go run

## 目录调用顺序

### cmd

#### main.go

- 启动入口
- 初始化各类的配置信息
- 服务名定义
- http/grpc 协议支持定义
- 其他 kratos 配置项 （微服务版本 metadata endpoint 等）

#### wire.go

根据 data, biz, service, server 提供的 privoer 内容，生成 wire_gen 的代码注入

#### wire_gen.go

由 wire.go 生成，用于组装 data, biz, service, server 之间实例化、调用逻辑

- 提供给 main.go 使用，将插件内容进行组装，比如启动 注册中心、grpc服务、http服务、日志、验证等
- 初始化其他服务的 client (见 beer-shop 示例的 shop.cmd.wire_gen userClient,cartClient)

### internal

#### data

数据库操作层，将多个数据库驱动封装在 data 层，提供 biz 层调用

实现最基础的数据操作动作(CRUD)，或者部分针对数据进行的简单逻辑（biz.Struct 结构转换、数据对比、缓存获取，数组组装）

***严禁对外提供 internal.Model 内容***，对外提供的数据结构必须为 biz.Struct 形式

#### biz

业务逻辑层

定义 biz.RepoInterface biz.Strcut 

实现依赖倒置， biz 定义 data层需要提供符合使用的 interface，以及对应的 struct 结构



![kratos](/Users/hanhui/note/draw/kratos/kratos.jpg)

#### 需要关注的案例

[upload](https://github.com/go-kratos/examples/blob/main/http/upload/main.go)

[websocket](https://github.com/go-kratos/examples/tree/main/ws)

