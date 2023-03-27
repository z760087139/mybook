# 单元测试及性能测试

## BenchMark

```shell
go test ./fib              // 不进行基准测试，对 fib 进行单元测试

go test -bench=. -run=none  // 进行基准测试，不进行单元测试，-run 表示执行哪些单元测试和测试函数，一般函数名不会是 none，所以不执行单元测试
// 上面的测试命令还可以用空格隔开，意义是一样
go test -bench . -run none

go test -bench=.    // 对所有的进行基准测试

go test -bench='fib$'     // 只运行以 fib 结尾的基准测试，-bench 可以进行正则匹配

go test -bench=. -benchtime=6s  // 基准测试默认时间是 1s，-benchtime 可以指定测试时间
go test -bench=. -benchtime=50x  // 参数 -benchtime 除了指定时间，还可以指定运行的次数

go test -bench=. -benchmem // 进行时间、内存的基准测试

// 上面的命令中，-bench 后面都有一个 . ，这个点并不是指当前文件夹，而是一个匹配所有测试的正则表达式。
```

* cpu 使用分析：-cpuprofile=cpu.pprof
* 内存使用分析：-benchmem -memprofile=mem.pprof
* block分析：-blockprofile=block.pprof

## cover

```shell
go test ./... -coverprofile=cover.out # 输出覆盖率文件到 cover.out
go tool cover -html=cover.out -o cover.html # 读取 cover.out 文件形成 html 格式，并输出到 cover.html（不适用 -o cover.html 则直接在浏览器打开）
```
