# 错误处理

## 生成错误

* proto 工具生成
* kratos.errors.New 生成

### 工具生成

#### 错误定义

api/helloworld/v1/helloworld.proto

```protobuf
syntax = "proto3";

package api.blog.v1;
import "errors/errors.proto";

// 多语言特定包名，用于源代码引用
option go_package = "github.com/go-kratos/examples/blog/api/v1;v1";
option java_multiple_files = true;
option java_package = "blog.v1.errors";
option objc_class_prefix = "APIBlogErrors";

enum ErrorReason {
  // 设置缺省错误码
  option (errors.default_code) = 500;
  
  // 为某个枚举单独设置错误码
  USER_NOT_FOUND = 0 [(errors.code) = 404];

  CONTENT_MISSING = 1 [(errors.code) = 400];;
}
```

注意事项:

* 当枚举组没有配置缺省错误码时, 当前枚举组的没有配置错误码的枚举值会被忽略
* 当整个枚举组都没配置错误码时，当前枚举组会被忽略
* 错误码的取值范围应该在 0 < code <= 600 之间, 超出范围将抛出异常

#### 错误生成

通过 proto 生成对应的代码：

```bash
protoc --proto_path=. \
         --proto_path=./third_party \
         --go_out=paths=source_relative:. \
         --go-errors_out=paths=source_relative:. \
         $(API_PROTO_FILES)
```

或者在项目根目录使用Makefile指令

```
make errors
```

## 使用方式

### 响应错误

当业务逻辑中需要响应错误时，可以通过使用 kratos errors 包中的 New 方法来响应错误, 或者可以通过proto定义，然后通过 protoc-gen-go-error 工具生成帮助代码来响应错误

```go
// 通过 errors.New() 响应错误
errors.New(500, "USER_NAME_EMPTY", "user name is empty")

// 通过 proto 生成的代码响应错误，并且包名应替换为自己生成代码后的 package name
api.ErrorUserNotFound("user %s not found", "kratos")

// 传递metadata
err := errors.New(500, "USER_NAME_EMPTY", "user name is empty")
err = err.WithMetadata(map[string]string{
    "foo": "bar",
})
```

### 错误断言

```go
err := wrong()

// 通过 errors.Is() 断言
if errors.Is(err,errors.BadRequest("USER_NAME_EMPTY","")) {
    // do something
}

// 通过判断 *Error.Reason 和 *Error.Code
e := errors.FromError(err)
if  e.Reason == "USER_NAME_EMPTY" && e.Code == 500 {
    // do something
}

// 通过 proto 生成的代码断言错误，并且包名应替换为自己生成代码后的 package name
if api.IsUserNotFound(err) {
        // do something
})
```
