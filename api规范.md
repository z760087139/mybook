# API开发规范

## 原则

> ​	API接口设计必须遵守 Google API 设计指南，详细规则内容可以参考 confluence 内容。本文档是基于 Google API 精简后的开发规范。
>
> ​	如果文档未说明的内容请参考 Google API 文档并更新本文档规范要求。

### 本指南中使用的惯例

本文档中使用的要求级别关键字（“必须”、“不得”、“必需”，“应”、“不应”、“应该”、“不应该”、“建议”、“可以”和“可选”）将按 [RFC 2119 ](https://www.ietf.org/rfc/rfc2119.txt)中的描述进行解释。

### 面向资源的设计

本设计指南的目标是帮助开发者设计**简单、一致且易用**的网络 API。同时，它还有助于将 RPC API（基于套接字）与 REST API（基于 HTTP）的设计融合起来。

REST Google API 的标准方法（也称为“REST 方法”）包括 `List`、`Get`、`Create`、`Update` 和 `Delete`。API 设计者还可以使用“自定义方法”（也称为“自定义动词”或“自定义操作”）来实现无法轻易映射到标准方法的功能（例如数据库事务）。

## 设计流程

设计指南建议在设计面向资源的 API 时采取以下步骤（更多细节将在下面的特定部分中介绍）：

- 确定 API 提供的资源类型。
- 确定资源之间的关系。
- 根据类型和关系确定资源名称方案。
- 确定资源架构。
- 将最小的方法集附加到资源。

### 资源

面向资源的 API 通常被构建为资源层次结构，其中每个节点是一个“简单资源”或“集合资源”。 为方便起见，它们通常被分别称为资源和集合。

- 一个集合包含**相同类型**的资源列表。 例如，一个用户拥有一组联系人。
- 资源具有一些状态和零个或多个子资源。 每个子资源可以是一个简单资源或一个集合资源。

例如，Gmail API 有一组用户，每个用户都有一组消息、一组线程、一组标签、一个个人资料资源和若干设置资源。

具体例子见 API 详细文档

### 资源名称

在面向资源的 API 中，“资源”是被命名的实体，“资源名称”是它们的标识符。每个资源都**必须**具有自己唯一的资源名称。 资源名称由资源自身的 ID、任何父资源的 ID 及其 API 服务名称组成。

资源名称由集合 ID 和资源 ID 构成，按分层方式组织并以正斜杠分隔。如果资源包含子资源，则子资源的名称由父资源名称后跟子资源的 ID 组成，也以正斜杠分隔。

示例 1：存储服务具有一组 `buckets`，其中每个存储分区都有一组 `objects`：

| API 服务名称             | 集合 ID  | 资源 ID    | 集合 ID  | 资源 ID    |
| :----------------------- | :------- | :--------- | :------- | :--------- |
| //storage.googleapis.com | /buckets | /bucket-id | /objects | /object-id |

#### 集合ID

标识其父资源中集合资源的非空 URI 段

- **必须**是复数形式的首字母小写驼峰体。如果该词语没有合适的复数形式，例如“evidence（证据）”和“weather（天气）”，则**应该**使用单数形式。

- **应该**避免过于笼统的词语，或对其进行限定后再使用。例如，`rowValues`优先于`value`。

- **应该**避免在不加以限定的情况下使用以下词语：

  - elements

  - entries

  - instances

  - items

  - objects

  - resources

  - types

  - values

#### 资源ID

 通常包含一个或多个用于标识非父资源中资源的非空 URI 段。资源名称中的非末尾资源 ID 必须只有 1 个网址段，而资源名称末尾的资源 ID **可以**具有多个 URI 段

#### 资源名称为字符串

除非存在向后兼容问题，否则 Google API **必须**使用纯字符串来表示资源名称。资源名称**应该**像普通文件路径一样处理。当资源名称在不同组件之间传递时，必须将它视为原子值，并且不得有任何数据损失。

对于资源定义，第一个字段**应该**是资源名称的字符串字段，并且**应该**称为 `name`。

```protobuf
service LibraryService {
  rpc GetBook(GetBookRequest) returns (Book) {
    option (google.api.http) = {
      get: "/v1/{name=shelves/*/books/*}"
    };
  };
  rpc CreateBook(CreateBookRequest) returns (Book) {
    option (google.api.http) = {
      post: "/v1/{parent=shelves/*}/books"
      body: "book"
    };
  };
}

message Book {
  // Resource name of the book. It must have the format of "shelves/*/books/*".
  // For example: "shelves/shelf1/books/book2".
  string name = 1;

  // ... other properties
}

message GetBookRequest {
  // Resource name of a book. For example: "shelves/shelf1/books/book2".
  string name = 1;
}

message CreateBookRequest {
  // Resource name of the parent resource where to create the book.
  // For example: "shelves/shelf1".
  string parent = 1;
  // The Book resource to be created. Client must not set the `Book.name` field.
  Book book = 2;
}
```

## 路由规范

> 根据 google api 文档暂定路由设计规范

API URI 必须严格遵守以下规则:

/api/{serviceName}/{apiVersion}/{collectionID}(/{resourceID}{:action}{?args})

| 变量名         | 中文含义                                    |
| -------------- | ------------------------------------------- |
| {serviceName}  | 微服务名，英文                              |
| {apiVersion}   | 接口版本，采用 v1/v2/v3 等标识              |
| {collectionID} | 集合ID，见上述文档解释                      |
| {resourceID}   | 资源ID，见上述文档解释                      |
| {:action}      | 自定义动作，使用 : 分割资源与自定义动作名称 |
| {?args}        | URI参数，如分页参数                         |

#### 例子1

`/api/policy/v1/policies/10:deploy?nooutput=true`

表示请求  微服务`policy`，接口版本 `v1` , 集合资源 `policies`,  资源ID `10`，执行动作 `deploy`，参数`nooutput=true`

#### 例子2

获取、设置某个部署策略的依赖信息

GET `/api/policy/v1/policies/10/dependences`

PUT `/api/policy/v1/policies/10/dependences`

### 标准方法

| 标准方法                                                     | HTTP 映射                     | HTTP 请求正文 | HTTP 响应正文             |
| :----------------------------------------------------------- | :---------------------------- | :------------ | :------------------------ |
| [`List`](https://cloud.google.com/apis/design/standard_methods#list) | `GET <collection URL>`        | 无            | 资源*列表                 |
| [`Get`](https://cloud.google.com/apis/design/standard_methods#get) | `GET <resource URL>`          | 无            | 资源*                     |
| [`Create`](https://cloud.google.com/apis/design/standard_methods#create) | `POST <collection URL>`       | 资源          | 资源*                     |
| [`Update`](https://cloud.google.com/apis/design/standard_methods#update) | `PUT or PATCH <resource URL>` | 资源          | 资源*                     |
| [`Delete`](https://cloud.google.com/apis/design/standard_methods#delete) | `DELETE <resource URL>`       | 不适用        | `google.protobuf.Empty`** |

> 每个方法的具体 proto 使用及编写见 Google API文档

#### 列表

- `List` 方法 **必须**使用 HTTP `GET` 动词。
- 接收其资源正在列出的集合名称的请求消息字段**应该**映射到网址路径。如果集合名称映射到网址路径，则网址模板的最后一段（[集合 ID](https://cloud.google.com/apis/design/resource_names#CollectionId)）**必须**是字面量。
- 所有剩余的请求消息字段**应该**映射到网址查询参数。
- 没有请求正文，API 配置**不得**声明 `body` 子句。
- 响应正文**应该**包含资源列表以及可选元数据。

#### 获取

- `Get` 方法 **必须**使用 HTTP `GET` 动词。
- 接收资源名称的请求消息字段**应该**映射到网址路径。
- 所有剩余的请求消息字段**应该**映射到网址查询参数。
- 没有请求正文，API 配置**不得**声明 `body` 子句。
- 返回的资源**应该**映射到整个响应正文。

#### 创建

- `Create` 方法 **必须**使用 HTTP `POST` 动词。
- 所有剩余的请求消息字段**应该**映射到网址查询参数。
- 返回的资源**应该**映射到整个 HTTP 响应正文。

#### 更新

- `Update`方法分为部分资源更新`PATCH`与完整资源更新`PUT`

- 需要更高级修补语义的 `Update` 方法（例如附加到重复字段）**应该**由[自定义方法](https://cloud.google.com/apis/design/custom_methods)提供。
- 如果 `Update` 方法仅支持完整资源更新，则**必须**使用 HTTP 动词 `PUT`，部分更新使用HTTP动词`PATCH`。但是，强烈建议不要进行完整更新，因为在添加新资源字段时会出现向后兼容性问题。
- 接收资源名称的消息字段**必须**映射到网址路径。该字段**可以**位于资源消息本身中。
- 包含资源的请求消息字段**必须**映射到请求正文。
- 所有剩余的请求消息字段**必须**映射到网址查询参数。
- 响应消息**必须**是更新的资源本身。

#### 删除

- `Delete` 方法 **必须**使用 HTTP `DELETE` 动词。
- 接收资源名称的请求消息字段**应该**映射到网址路径。
- 所有剩余的请求消息字段**应该**映射到网址查询参数。
- 没有请求正文，API 配置**不得**声明 `body` 子句。
- 如果 `Delete` 方法立即移除资源，则**应该**返回空响应。
- 如果 `Delete` 方法启动长时间运行的操作，则**应该**返回长时间运行的操作。
- 如果 `Delete` 方法仅将资源标记为已删除，则**应该**返回更新后的资源。

### 自定义方法

- 自定义方法**应该**使用 HTTP `POST` 动词，因为该动词具有最灵活的语义，但作为替代 get 或 list 的方法（如有可能，**可以**使用 `GET`）除外。（详情请参阅第三条。）
- 自定义方法**不应该**使用 HTTP `PATCH`，但**可以**使用其他 HTTP 动词。在这种情况下，方法**必须**遵循该动词的标准 [HTTP 语义](https://tools.ietf.org/html/rfc2616#section-9)。
- 请注意，使用 HTTP `GET` 的自定义方法**必须**具有幂等性并且无负面影响。例如，在资源上实现特殊视图的自定义方法**应该**使用 HTTP `GET`。
- 接收与自定义方法关联的资源或集合的资源名称的请求消息字段**应该**映射到网址路径。
- 网址路径**必须**以包含冒号（后跟自定义动词）的后缀结尾。
- 如果用于自定义方法的 HTTP 动词允许 HTTP 请求正文（其适用于 `POST`、`PUT`、`PATCH` 或自定义 HTTP 动词），则此自定义方法的 HTTP 配置**必须**使用 `body: "*"` 子句，所有其他请求消息字段都**应**映射到 HTTP 请求正文。
- 如果用于自定义方法的 HTTP 动词不接受 HTTP 请求正文（`GET`、`DELETE`），则此方法的 HTTP 配置**不得**使用 `body` 子句，并且所有其他请求消息字段都**应**映射到网址查询参数。

### 标准字段

优先使用标准字段定义接口字段内容

**其中 create_time / update_time 等时间字段，建议统一使用一般现在时（包括数据库字段设计）**

| 名称               | 类型                                                         | 说明                                                         |
| :----------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `name`             | `string`                                                     | `name` 字段应包含[相对资源名称](https://cloud.google.com/apis/design/resource_names#relative_resource_name)。 |
| `parent`           | `string`                                                     | 对于资源定义和 List/Create 请求，`parent` 字段应包含父级[相对资源名称](https://cloud.google.com/apis/design/resource_names#relative_resource_name)。 |
| `create_time`      | [`Timestamp`](https://github.com/google/protobuf/blob/master/src/google/protobuf/timestamp.proto) | 创建实体的时间戳。                                           |
| `update_time`      | [`Timestamp`](https://github.com/google/protobuf/blob/master/src/google/protobuf/timestamp.proto) | 最后更新实体的时间戳。注意：执行 create/patch/delete 操作时会更新 update_time。 |
| `delete_time`      | [`Timestamp`](https://github.com/google/protobuf/blob/master/src/google/protobuf/timestamp.proto) | 删除实体的时间戳，仅当它支持保留时才适用。                   |
| `expire_time`      | [`Timestamp`](https://github.com/google/protobuf/blob/master/src/google/protobuf/timestamp.proto) | 实体到期时的到期时间戳。                                     |
| `start_time`       | [`Timestamp`](https://github.com/google/protobuf/blob/master/src/google/protobuf/timestamp.proto) | 标记某个时间段开始的时间戳。                                 |
| `end_time`         | [`Timestamp`](https://github.com/google/protobuf/blob/master/src/google/protobuf/timestamp.proto) | 标记某个时间段或操作结束的时间戳（无论其成功与否）。         |
| `read_time`        | [`Timestamp`](https://github.com/google/protobuf/blob/master/src/google/protobuf/timestamp.proto) | 应读取（如果在请求中使用）或已读取（如果在响应中使用）特定实体的时间戳。 |
| `time_zone`        | `string`                                                     | 时区名称。它应该是 [IANA TZ](http://www.iana.org/time-zones) 名称，例如“America/Los_Angeles”。如需了解详情，请参阅 https://en.wikipedia.org/wiki/List_of_tz_database_time_zones。 |
| `region_code`      | `string`                                                     | 位置的 Unicode 国家/地区代码 (CLDR)，例如“US”和“419”。如需了解详情，请访问 http://www.unicode.org/reports/tr35/#unicode_region_subtag。 |
| `language_code`    | `string`                                                     | BCP-47 语言代码，例如“en-US”或“sr-Latn”。如需了解详情，请参阅 http://www.unicode.org/reports/tr35/#Unicode_locale_identifier。 |
| `mime_type`        | `string`                                                     | IANA 发布的 MIME 类型（也称为媒体类型）。如需了解详情，请参阅 https://www.iana.org/assignments/media-types/media-types.xhtml。 |
| `display_name`     | `string`                                                     | 实体的显示名称。                                             |
| `title`            | `string`                                                     | 实体的官方名称，例如公司名称。它应被视为 `display_name` 的正式版本。 |
| `description`      | `string`                                                     | 实体的一个或多个文本描述段落。                               |
| `filter`           | `string`                                                     | List 方法的标准过滤器参数。请参阅 [AIP-160](https://google.aip.dev/160)。 |
| `query`            | `string`                                                     | 如果应用于搜索方法（即 [`:search`](https://cloud.google.com/apis/design/custom_methods#common_custom_methods)），则与 `filter` 相同。 |
| `page_token`       | `string`                                                     | List 请求中的分页令牌。                                      |
| `page_size`        | `int32`                                                      | List 请求中的分页大小。                                      |
| `total_size`       | `int32`                                                      | 列表中与分页无关的项目总数。                                 |
| `next_page_token`  | `string`                                                     | List 响应中的下一个分页令牌。它应该用作后续请求的 `page_token`。空值表示不再有结果。 |
| `order_by`         | `string`                                                     | 指定 List 请求的结果排序。                                   |
| `progress_percent` | `int32`                                                      | 指定操作的进度百分比 (0-100)。值 `-1` 表示进度未知.          |
| `request_id`       | `string`                                                     | 用于检测重复请求的唯一字符串 ID。                            |
| `resume_token`     | `string`                                                     | 用于恢复流式传输请求的不透明令牌。                           |
| `labels`           | `map<string, string>`                                        | 表示 Cloud 资源标签。                                        |
| `show_deleted`     | `bool`                                                       | 如果资源允许恢复删除行为，相应的 List 方法必须具有 `show_deleted` 字段，以便客户端可以发现已删除的资源。 |
| `update_mask`      | [`FieldMask`](https://github.com/google/protobuf/blob/master/src/google/protobuf/field_mask.proto) | 它用于 `Update` 请求消息，该消息用于对资源执行部分更新。此掩码与资源相关，而不是与请求消息相关。 |
| `validate_only`    | `bool`                                                       | 如果为 true，则表示仅应验证给定请求，而不执行该请求。        |

## 报文规范

### 正常返回

**必须**以json对象形式或空内容返回报文结果，禁止直接返回字符串、json列表等其他格式

#### 列表返回

```protobuf
message ListBooksResponse {
  // The field name should match the noun "books" in the method name.  There
  // will be a maximum number of items returned based on the page_size field
  // in the request.
  repeated Book books = 1;

  // Token to retrieve the next page of results, or empty if there are no
  // more results in the list.
  string next_page_token = 2;
}
```

资源列表放置于对象下某个字段中返回，字段名称不作强制要求，但建议为资源复数形式（旧系统为data)

### 错误返回

> 错误报文返回采用 kratos 标准格式

json 例子

```json
// json 报文格式
{
    // 错误码，跟 http-status 一致，并且在 grpc 中可以转换成 grpc-status
    "code": 500,
    // 错误原因，定义为业务判定错误码
    "reason": "USER_NOT_FOUND",
    // 错误信息，为用户可读的信息，可作为用户提示内容
    "message": "invalid argument error",
    // 错误元信息，为错误添加附加可扩展信息
    "metadata": {
      "foo": "bar"
    }
}
```

protobuf 例子

```prot
syntax = "proto3";

// 定义包名
package api.kratos.v1;
import "errors/errors.proto";

// 多语言特定包名，用于源代码引用
option go_package = "kratos/api/helloworld;helloworld";
option java_multiple_files = true;
option java_package = "api.helloworld";

enum ErrorReason {
  // 设置缺省错误码
  option (errors.default_code) = 500;

  // 为某个枚举单独设置错误码
  USER_NOT_FOUND = 0 [(errors.code) = 404];

  CONTENT_MISSING = 1 [(errors.code) = 400];
}
```

``` shell
# 代码生成
protoc --proto_path=. \
         --proto_path=./third_party \
         --go_out=paths=source_relative:. \
         --go-errors_out=paths=source_relative:. \
         $(API_PROTO_FILES)
```

golang 使用例子

```go
// api/error.go
// 生成代码

func IsUserNotFound(err error) bool {
	if err == nil {
		return false
	}
	e := errors.FromError(err)
	return e.Reason == OrderServiceErrorReason_USER_NOT_FOUND.String() && e.Code == 404
}

func ErrorUserNotFound(format string, args ...interface{}) *errors.Error {
	return errors.New(404, OrderServiceErrorReason_USER_NOT_FOUND.String(), fmt.Sprintf(format, args...))
}

func IsContentMissing(err error) bool {
	if err == nil {
		return false
	}
	e := errors.FromError(err)
	return e.Reason == OrderServiceErrorReason_CONTENT_MISSING.String() && e.Code == 400
}

func ErrorContentMissing(format string, args ...interface{}) *errors.Error {
	return errors.New(400, OrderServiceErrorReason_CONTENT_MISSING.String(), fmt.Sprintf(format, args...))
}
```

为了能错误内容管理及方便调用方进行错误类型断言，**建议** server 采用方式二进行错误内容定义。

另外，程序内部的错误嵌套及处理建议查看 uber go 语言编码规范。

``` go
// server.go
//
// 通过 errors.New() 响应错误，方式一
err1 := errors.New(500, "USER_NAME_EMPTY", "user name is empty")

// 通过 proto 生成的代码响应错误，并且包名应替换为自己生成代码后的 package name，方式二
err2 := api.ErrorUserNotFound("user %s not found", "kratos")

// 传递metadata
err := errors.New(500, "USER_NAME_EMPTY", "user name is empty")
err = err.WithMetadata(map[string]string{
    "foo": "bar",
})

// client.go
//
err := wrong()

// 通过 errors.Is() 断言,方式一
if errors.Is(err,errors.BadRequest("USER_NAME_EMPTY","")) {
// do something
}

// 通过判断 *Error.Reason 和 *Error.Code，方式二
e := errors.FromError(err)
if  e.Reason == "USER_NAME_EMPTY" && e.Code == 500 {
// do something
}

// 通过 proto 生成的代码断言错误，并且包名应替换为自己生成代码后的 package name（此处对应上面生成的 helloworld 包，调用定义的方法），方式三
if helloworld.IsUserNotFound(err) {
// do something
})
```

#### http code 标准

| HTTP | gRPC                 | 说明                                                         |
| :--- | :------------------- | :----------------------------------------------------------- |
| 200  | `OK`                 | 无错误。                                                     |
| 400  | `INVALID_ARGUMENT`   | 客户端指定了无效参数。如需了解详情，请查看错误消息和错误详细信息。 |
| 401  | `UNAUTHENTICATED`    | 由于 OAuth 令牌丢失、无效或过期，请求未通过身份验证。        |
| 403  | `PERMISSION_DENIED`  | 客户端权限不足。这可能是因为 OAuth 令牌没有正确的范围、客户端没有权限或者 API 尚未启用。 |
| 404  | `NOT_FOUND`          | 未找到指定的资源。                                           |
| 409  | `ABORTED`            | 并发冲突，例如读取/修改/写入冲突。                           |
| 409  | `ALREADY_EXISTS`     | 客户端尝试创建的资源已存在。                                 |
| 429  | `RESOURCE_EXHAUSTED` | 资源配额不足或达到速率限制。如需了解详情，客户端应该查找 google.rpc.QuotaFailure 错误详细信息。 |
| 499  | `CANCELLED`          | 请求被客户端取消。                                           |
| 500  | `INTERNAL`           | 出现内部服务器错误。通常是服务器错误。                       |
| 501  | `NOT_IMPLEMENTED`    | API 方法未通过服务器实现。                                   |
| 502  | 不适用               | 到达服务器前发生网络错误。通常是网络中断或配置错误。         |
| 503  | `UNAVAILABLE`        | 服务不可用。通常是服务器已关闭。                             |
| 504  | `DEADLINE_EXCEEDED`  | 超出请求时限。仅当调用者设置的时限比方法的默认时限短（即请求的时限不足以让服务器处理请求）并且请求未在时限范围内完成时，才会发生这种情况。 |

## 设计模式

### 列表分页

可列表集合**应该**支持分页，即使结果通常很小

**说明**：如果某个 API 从一开始就不支持分页，稍后再支持它就比较麻烦，因为添加分页会破坏 API 的行为。 不知道 API 正在使用分页的客户端可能会错误地认为他们收到了完整的结果，而实际上只收到了第一页。

根据目前系统使用情况，可以实时数据进行分页，无需采用分页码进行分页缓存。

关于分页请求、分页返回，统一采用以下字段

| 字段名     | 说明                                     |
| ---------- | ---------------------------------------- |
| page_size  | 请求、返回报文使用，表示当前分页预期大小 |
| page_num   | 请求、返回报文使用，表示当前分页为第几页 |
| total_size | 返回报文使用，表示列表对象总数           |

```json
{
  "policies":[....],
  "page_size":10,
  "page_num": 1,
  "total_size": 25
}
```

### 列出子集合

有时，API 需要让客户跨子集执行 `List/Search` 操作。例如，“API 图书馆”有一组书架，每个书架都有一系列书籍，而客户希望在所有书架上搜索某一本书。在这种情况下，建议在子集合上使用标准 `List`，并为父集合指定通配符集合 ID `"-"`。对于“API 图书馆”示例，我们可以使用以下 REST API 请求：

```
GET https://library.googleapis.com/v1/shelves/-/books?filter=xxx
```

**注意**：选择 `"-"` 而不是 `"*"` 的原因是为了避免需要进行 URL 转义。

### 从子集合中获取唯一资源

有时子集合中的资源具有在其父集合中唯一的标识符。此时，在不知道哪个父集合包含它的情况下使用 `Get` 检索该资源可能很有用。在这种情况下，建议对资源使用标准 `Get`，并为资源在其中是唯一的所有父集合指定通配符集合 ID `"-"`。例如，在 API 图书馆中，如果书籍在所有书架上的所有书籍中都是唯一的，我们可以使用以下 REST API 请求：

```
GET https://library.googleapis.com/v1/shelves/-/books/{id}
```

响应此调用的资源名称**必须**使用资源的规范名称，并使用实际的父集合标识符而不是每个父集合都使用 `"-"`。例如，上面的请求应返回名称为 `shelves/shelf713/books/book8141`（而不是 `shelves/-/books/book8141`）的资源。

### 排序顺序

如果 API 方法允许客户端指定列表结果的排序顺序，则请求消息**应该**包含一个字段：

```proto
string order_by = ...;
```

字符串值**应该**遵循 SQL 语法：逗号分隔的字段列表。例如：`"foo,bar"`。默认排序顺序为升序。要将字段指定为降序，**应该**将后缀 `" desc"` 附加到字段名称中。例如：`"foo desc,bar"`。

语法中的冗余空格字符是无关紧要的。 `"foo,bar desc"` 和 `" foo , bar desc "` 是等效的。

## 命名规范

### 包名

包名为小写，并且同目录结构一致，例如：api/policy/v1/

```
// package <path>
package api.policy.v1;
// option go_package = "github.com/cgb-acp/services/<package_name>;<version>";
option go_package = "github.com/cgb-acp/services/api/policy/v1;v1";
```

### 文件结构

文件应该命名为：`lower_snake_case.proto` 所有Proto应按下列方式排列:

1. License header (if applicable)
2. File overview
3. Syntax
4. Package
5. Imports (sorted)
6. File options
7. Everything else

### Message 和 字段命名

使用驼峰命名法（首字母大写）命名 message，使用下划线命名字段

``` protobuf
Message Policy {
	string policy_name = 1;
}
```

#### 枚举 Enums

使用驼峰命名法（首字母大写）命名枚举类型，使用 “大写*下划线*大写” 的方式命名枚举值：

```protobuf
enum Foo {
  FIRST_VALUE = 0;
  SECOND_VALUE = 1;
}
```

#### 服务 Services

如果你在 .proto 文件中定义 RPC 服务，你应该使用驼峰命名法（首字母大写）命名 RPC 服务以及其中的 RPC 方法：

```protobuf
service FooService {
  rpc GetSomething(FooRequest) returns (FooResponse);
}
```

Copy

#### Comment

- Service，描述清楚服务的作用
- Method，描述清楚接口的功能特性
- Field，描述清楚字段准确的信息

### Examples

API Service接口定义(demo.proto)

```protobuf
syntax = "proto3";

package kratos.demo.v1;

// 多语言特定包名，用于源代码引用
option go_package = "github.com/go-kratos/kratos/demo/v1;v1";

// 描述该服务的信息
service Greeter {
    // 描述该方法的功能
    rpc SayHello (HelloRequest) returns (HelloReply);
}
// Hello请求参数
message HelloRequest {
    // 用户名字
    string name = 1;
}
// Hello返回结果
message HelloReply {
    // 结果信息
    string message = 1;
}
```

## 参数校验

> 对于参数边界值内容校验，采用 kratos 提供的方法

1. 安装[proto-gen-validate](https://github.com/envoyproxy/protoc-gen-validate)。

```shell
go install github.com/envoyproxy/protoc-gen-validate@latest
```

2. 编写 protobuf

```proto
// 参数必须为 /hello
string path = 6 [(validate.rules).string.const = "/hello"];
// 参数文本长度必须为 11
string phone = 7 [(validate.rules).string.len = 11];
// 参数文本长度不能小于 10 个字符
string explain = 8 [(validate.rules).string.min_len =  10];
// 参数文本长度不能小于 1 个字符并且不能大于 10 个字符
string name = 9 [(validate.rules).string = {min_len: 1, max_len: 10}];
// 参数文本使用正则匹配,匹配必须是非空的不区分大小写的十六进制字符串
string card = 10 [(validate.rules).string.pattern = "(?i)^[0-9a-f]+$"];
// 参数文本必须是 email 格式
string email = 11 [(validate.rules).string.email = true];
```

3. 生成代码

``` shell
protoc --proto_path=. \
           --proto_path=./third_party \
           --go_out=paths=source_relative:. \
           --validate_out=paths=source_relative,lang=go:. \
           xxxx.proto
```

4. kratos 引用

http

```go
httpSrv := http.NewServer(
    http.Address(":8000"),
    http.Middleware(
        validate.Validator(),
    ))
```

grpc

```go
grpcSrv := grpc.NewServer(
    grpc.Address(":9000"),
    grpc.Middleware(
        validate.Validator(),
    ))
```

### 校验规则

https://github.com/envoyproxy/protoc-gen-validate#constraint-rules

## Swagger 

由gateway实现聚合