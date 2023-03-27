# code-generator

### 使用方式

`code-generator.sh <generators> <output-package> <apis-package> <groups-versions>`

### 标签

通过 go 文件中的标签信息控制生成器生成的代码内容

标签通常具有`// +tag-name`或的形状`// +tag-name=value`

#### 全局标签

package 目录下的 doc.go 文件

全局标签被写入`doc.go`包的文件中。一个典型的`pkg/apis/<apigroup>/<version>/doc.go`样子是这样的：

```
// +k8s:deepcopy-gen=package
// Package v1 is the v1 version of the API.
// +groupName=example.com
 package v1
```

它告诉 deepcopy-gen 默认为该包中的每种类型创建 deepcopy 方法。如果您有不需要或不需要 deepcopy 的类型，您可以选择不使用带有 local tag 的此类类型`// +k8s:deepcopy-gen=false`。如果您不启用包范围的深度复制，则必须通过`// +k8s:deepcopy-gen=true`.

`// +groupName=example.com` 定义完全限定的 API 组名称。如果你弄错了，client-gen 将产生错误的代码。请注意，此标签必须位于上方的注释块中\`package

#### 本地标签

```go
// +genclient
// +genclient:noStatus -- 无 status 字段
// +genclient:nonNamespaced -- 集群级别资源
// +genclient:noVerbs  -- 无 http restful 接口
// +genclient:onlyVerbs=create,delete
// +genclient:skipVerbs=get,list,create,update,patch,delete,deleteCollection,watch
// +genclient:method=Create,verb=create,result=k8s.io/apimachinery/pkg/apis/meta/v1.Status -- 仅创建，并且接口只返回 status 信息
```
