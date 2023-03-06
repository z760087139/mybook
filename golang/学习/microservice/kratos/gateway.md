# kratos gateway

fileloader 每5秒检查一次 config 文件的 SHA256是否发生变化，并执行 watch 列表的函数

``` go
func (f *fileLoader) watchproc(ctx context.Context) {
	LOG.Info("start watch config file")
	for {
		select {
		case <-ctx.Done():
			return
		case <-time.After(time.Second * 5):
		}
		func() {
			sha256hex, err := f.configSHA256()
			if err != nil {
				LOG.Errorf("watch config file error: %+v", err)
				return
			}
			if sha256hex != f.confSHA256 {
				LOG.Infof("config file changed, reload config, last sha256: %s, new sha256: %s", f.confSHA256, sha256hex)
				if err := f.executeLoader(); err != nil {
					LOG.Errorf("execute config loader error with new sha256: %s: %+v, config digest will not be changed until all loaders are succeeded", sha256hex, err)
					return
				}
				f.confSHA256 = sha256hex
				return
			}
		}()
	}
}
```



``` go
type Gateway struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	Name        string        `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty"`
	Version     string        `protobuf:"bytes,2,opt,name=version,proto3" json:"version,omitempty"`
	Hosts       []string      `protobuf:"bytes,3,rep,name=hosts,proto3" json:"hosts,omitempty"`
	Endpoints   []*Endpoint   `protobuf:"bytes,4,rep,name=endpoints,proto3" json:"endpoints,omitempty"`
	Middlewares []*Middleware `protobuf:"bytes,5,rep,name=middlewares,proto3" json:"middlewares,omitempty"`
}
```

