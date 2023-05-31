# HTTPS

HTTP + SSL / TLS = HTTPS

## SSL

secure socket layer 安全嵌套字层 曾出现严重 bug ，已停用

## TLS

transport layer security

!\[image-20220810164946511]\(/Users/hanhui/Library/Application Support/typora-user-images/image-20220810164946511.png)

## HTTP 1.1

与 HTTP1.0 相比，增加了长链接与流水线技术。http 1.1 假定tcp连接应该一直打开，直到被通知关闭，允许客户端通过同一连接发送多个请求

缺点：可能由于队头得不到响应，堵塞后续的请求

## HTTP 2.0

采用了二进制框架层编码，请求和响应分成更小的包

http2 在两端简历一个独立的连接，连接包含多个数据流，每个流包含多个请求/响应格式的消息
