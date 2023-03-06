# API文档插件

自动生成 swagger API文档，不需要再手动维护API文档接口

```
import	"github.com/go-kratos/swagger-api/openapiv2"

h := openapiv2.NewHandler()
//将/q/路由放在最前匹配
httpSrv.HandlePrefix("/q/", h)
```

#### 给swagger api添加说明、examples

请参考[a_bit_of_everything](https://github.com/grpc-ecosystem/grpc-gateway/blob/master/examples/internal/proto/examplepb/a_bit_of_everything.proto)，同时注意需要将[annotations.proto](https://github.com/go-kratos/kratos/blob/main/third_party/protoc-gen-openapiv2/options/annotations.proto)和[openapiv2.proto](https://github.com/go-kratos/kratos/blob/main/third_party/protoc-gen-openapiv2/options/openapiv2.proto)这两个proto文件复制到third_party/protoc-gen-openapiv2/options目录下