# json 序列化

json 序列化的 omitempty 参数影响测试

#### 结论：

omitempty 只对序列化结果造成影响，对于反序列化结果没有任何区别

golang 对零值的反序列化需要自行考虑处理。采用指针或者mask 等内容进行解决

```go
func Test_Marshal(t *testing.T) {
   type Param struct {
      Name string `json:"name,omitempty"`
      ID   int    `json:"id,omitempty"`
      Flag bool   `json:"flag,omitempty"`
   }
   b := [][]byte{
      []byte(`{"name":"t1"}`),
      []byte(`{"name":"t1","id":0}`),
      []byte(`{"flag":true}`),
   }
   for _, d := range b {
      var p Param
      if err := json.Unmarshal(d, &p); err != nil {
         t.Error(err)
         return
      }
      fmt.Printf("unmrashal %#v\n", p)

      m, err := json.Marshal(p)
      if err != nil {
         t.Error(err)
      }
      fmt.Printf("marshal %v\n", string(m))
   }
}
```

```
unmrashal data.Param{Name:"t1", ID:0, Flag:false}
marshal {"name":"t1"}
unmrashal data.Param{Name:"t1", ID:0, Flag:false}
marshal {"name":"t1"}
unmrashal data.Param{Name:"", ID:0, Flag:true}
marshal {"flag":true}
```

去除 omitempty

```
unmrashal data.Param{Name:"t1", ID:0, Flag:false}
marshal {"name":"t1","id":0,"flag":false}
unmrashal data.Param{Name:"t1", ID:0, Flag:false}
marshal {"name":"t1","id":0,"flag":false}
unmrashal data.Param{Name:"", ID:0, Flag:true}
marshal {"name":"","id":0,"flag":true}
```
