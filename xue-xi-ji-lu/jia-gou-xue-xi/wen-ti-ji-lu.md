# 问题记录

### protobuf

#### proto引入

```shell
/api
  |- user
     |- v1
        |- user.proto
        |- global_enum.proto
  |- third_party
     |- google
        |- protobuf
           |- struct.proto
```

```shell
> pwd
/api/user

> cat user.proto
syntax = 'proto3';
package api.user.v1

import "google/protobuf/struct.proto";
import "v1/global_enum.proto"; // 与 protoc 引入参数有关
...

> # 在 /api/uesr 目录下执行 protoc 命令，同时注明proto文件路径为当前目录 . 和 ../third_party
> protoc --proto_path=. --proto_path=../third_party v1/user.proto v1/global_enum.proto
> # protobuf 中 import 的路径必须在 --proto_path 参数中能找到对应文件
```

#### 类型问题

pb 反序列化成 golang 类型 的过程，需要先在 pb 声明字段的类型

其中与 golang 类型转换异常的类型：

* interface{}
* time.Time

**interface{}**

**涉及问题**

pb 中，存在wellknowtype的说明，其中在 stackoverflow 中常用 pb.Any 来表示 interface{}

但实际使用过程中, 通过 proto-gen-go 生成的 pb.go 对 pb.Any 类型的转换并非 interface{}, 是 anypb.Any。

如果希望将应用层与领域层分离，此时 anypb.Any 就会成为难点。

```protobuf
// `Any` contains an arbitrary serialized protocol buffer message along with a
// URL that describes the type of the serialized message.
//
// Protobuf library provides support to pack/unpack Any values in the form
// of utility functions or additional generated methods of the Any type.
```

根据 protobuf 对 Any 的描述， Any 是针对 protobuf 中声明的 Message 使用的，表示任意已声明的 Message，因此 Any 的 Unmarshal 实际为 protobuf.Message 类型的转换

在领域层可能需要屏蔽对应用层数据结构的感知，将 protobuf Message 与领域层解耦。此时使用 Any Unmarshal 转换的仍然是 pb.Message，无法将 pb Message 与 领域层解耦。

**解决方法**

pb 存在 struct 类型，通过 proto-gen-go 生成的 pb.go 对 struct 类型转换为 map\[string]interface{} 。

另外还有 sliceValue 类型及 value 类型，分别转换为 \[]interface{} 与 interface{}

**time.Time**

pb中，使用 timestamppb.timestamp 类型表示 golang time.Time。proto-gen-go 转换的类型为 timestamppb.Timestamp，非直接表示 golang time.Time

因此，如果直接通过 json.Marshal ，会得到一个 timestamppb.Timestamp 结构体的序列化结果

{secondes: xx , nanos: xxx}

**解决方法**

使用 protobuf 提供的 protojson 工具格式化，可以将将 timestamppb.timestamp 序列化成 RFC 3339 格式

同时，protojson 工具还提供常用的其他类型转换，比如 enum 的字符串与num 的序列化和反序列化，零值是否固定输出，是否采用驼峰形式输出

#### 字段名

根据 google api 指南，对外提供API接口时应采用 蛇形（下划线分割）字段名规范。

与字段名相关的工具：

proto-gen-go -- API 接口 生成 golang struct 的字段名及 tag 标签内容

proto-gen-openapiv2 -- API 接口 swagger 文档生成字段名

protojson -- 序列化、反序列化使用的工具，能够设置驼峰及蛇形字段名的序列化

#### 零值问题

**通病**

由于golang 的反序列化特性，反序列化过程会把未输入的字段赋予零值，并在序列化时把零值字段进行忽略。最终导致输入提供 "",0,false,\[] 零值字段时，无法保证输出与输入结果的一致性

因此在与前端交互时，一般需要与前端协商处理零值问题，在设计时考虑对零值进行忽略或者强制更新等方向。或者通过指针进行零值识别。

**protobuf**

**序列化返回处理**

protojson 工具有一个开关： 忽略或强制返回零值字段。

忽略情况：

1. 前端对不存在的字段进行默认值处理或者零值处理
2. protojson 不返回空数组的字段（即不返回 slice:\[]），对于不存在的数组字段同样需要处理

强制零值：

1. 存在大量冗余字段的接口会有大量无效返回
2. 前端需要对应显示的内容进行逻辑处理：对大量的零值返回，识别实际需要使用的报文内容

**反序列化接收处理**

由于反序列化会填充零值内容，无法正确识别该零值是预期中的值或者反序列的填充内容

设计角度：

​ 所有零值结果都认为反序列填充结果，当做报文未发送处理。

* 数据入库不记录零值字段，返回时正常读取并忽略零值内容返回
* 数据更新时忽略该字段，对于非必填字段确实需要进行内容清空，可以考虑使用 replace ," "，-1等形式处理，规避零值字段入库

开发角度：

​ 使用指针接收零值内容

* bool / int 使用指针接收，如果字段为 nil ，识别为报文未发送，不做处理
* 如果字段不为 nil ，则认为报文已发送，预期改为 false / 0 ，入库时强制更新零值字段信息

其中，需要注意反序列化后 enum 类型的值是否正常。如果报文使用默认值，反序列化后接收的内容则为零值。

#### swagger

由于 protobuf 内容较多，经常会拆分成多个 proto 文件。使用 proto-gen-openapiv2 时需要考虑对 swagger 内容进行合并，同时还要考虑蛇形命名规则

\--openapiv2\_opt=allow\_merge=true,merge\_file\_name=apidocs,json\_names\_for\_fields=false

合并时，注意proto 文件的 package 信息需要统一

### Kratos

#### mock

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

**解决方法**

**方法一**

​ 将mock 文件在 biz 同级目录生成，不作为独立package 存放

​ `mockery --all --inpackage`

**方法二**

​ 将mock / test 文件独立存放， 但无法测试非导出函数或者方法

#### Transaction

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
