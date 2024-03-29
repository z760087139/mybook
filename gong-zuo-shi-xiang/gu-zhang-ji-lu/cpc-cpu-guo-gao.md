# CPC CPU过高

根据性能监控报告发现，日常运行中存在CPU使用率过高的情况

特别在业务高峰期尤其明显

通过 pprof 采集 cpu profile 内容发现，CPU使用率由一个函数调用占据

函数内容

```go
func RecvChan(ctx context.Context, ch chan struct{}){
  go func(){
    for {
      select {
        case <- ctx.Done():
          return
        case <- ch():
          // do something
        default:
      }
    }
  }
}
```

cpu profile 中，cpu 消耗主要为 runtime.selectgo

其中 runtime.sellock runtime.selunlock 两个函数为主

该两个函数为 select 进行随机选择分支时候锁定和解锁使用，按照该说法，推测是由于 default 的定义导致 select 未进行堵塞，不断的循环随机选择分支内容导致大量的锁和解锁。
