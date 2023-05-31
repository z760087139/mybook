---
description: 关于 kratos server 及 client 的 handler 执行顺序及逻辑记录
---

# server

### Middleware

server 的 middleware 执行顺序按照先进先出

```go
// Chain returns a Middleware that specifies the chained handler for endpoint.
func Chain(m ...Middleware) Middleware {
   return func(next Handler) Handler {
      for i := len(m) - 1; i >= 0; i-- {
         next = m[i](next)
      }
      return next
   }
}

// grpc server
h = middleware.Chain(s.middleware...)(h)
```

入参 h 为 server handler 内容，放在最后追加执行

### TransportContext

关于  middleware ,  metadata,  transportContext 的使用和处理流程记录

<figure><img src="../../../.gitbook/assets/kratos - metadata 传递.png" alt=""><figcaption></figcaption></figure>

kratos 通过 middleware 组装串行的 handler 执行流程

metadata.Server 将 context 转换成 transport.Context 对象，并从 http header / gprc metadata 中提取固定的 prefix header 内容，将内容转换成 metadata.Context内容放置到 context 对象中

handler 即 services / biz / data 层可以通过 metadata.FromServerContext 获取上述步骤放置的内容；使用 **metadata.NewClientContext** 追加 metadata 内容

metadata.Client 从 **transport.ClientContext , metadata.ClientContext, metadata.ServerContext** 获取内容并追加到 grpc.meatadat 并向下传递

#### 小结

metadata.Server 负责接收和给handler 提供调用方传递的 metadata 内容，同时也提供 metadata.Client 可能需要的传递内容

metadata.Client 负责从 context 获取待传递的内容并且往下传递

业务场景1 - 需要将 header 中的 user 信息往下传递，因此需要服务增加 metadata.Server ，并通过 metadata.Client 继续传递到下游服务中

业务场景2 - 希望针对某些服务的所有调用前增加传递 metadata 内容，因此在 data层的 grpc.Middleware 增加该服务的 metadata 追加逻辑
