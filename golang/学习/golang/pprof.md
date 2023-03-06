# pprof 

```go
package main

import(
	_ "net/http/pprof"
	"github.com/felixge/fgprof"
)

func main() {
	http.DefaultServeMux.Handle("/debug/fgprof", fgprof.Handler())
	go func() {
		log.Println(http.ListenAndServe(":6060", nil))
	}()

	// <code to profile>
}
```

采集命令

```shell
curl 127.0.0.1:6060/debug/pprof/profile?seconds=10s > cpu.pprof
```

实时获取

```shell
go tool pprof -http=:6061 http://localhost:6060/debug/pprof/profile?seconds=10
# off-cpu
go tool pprof --http=:6061 http://localhost:6060/debug/fgprof?seconds=10
```



### 采集内容

- cpu（CPU Profiling）: `$HOST/debug/pprof/profile`，默认进行 30s 的 CPU Profiling，得到一个分析用的 profile 文件
- block（Block Profiling）：`$HOST/debug/pprof/block`，查看导致阻塞同步的堆栈跟踪
- goroutine：`$HOST/debug/pprof/goroutine`，查看当前所有运行的 goroutines 堆栈跟踪
- heap（Memory Profiling）: `$HOST/debug/pprof/heap`，查看活动对象的内存分配情况
- mutex（Mutex Profiling）：`$HOST/debug/pprof/mutex`，查看导致互斥锁的竞争持有者的堆栈跟踪
- threadcreate：`$HOST/debug/pprof/threadcreate`，查看创建新OS线程的堆栈跟踪



### pprof 工具

#### 命令行交互

| 名称      | 参数             | 标签    | 说明                                                         |
| :-------- | :--------------- | :------ | :----------------------------------------------------------- |
| gv        | [focus]          |         | 将当前概要文件以图形化和层次化的形式显示出来。当没有任何参数时，在概要文件中的所有抽样都会被显示。如果指定了focus参数，则只显示调用栈中有名称与此参数相匹配的函数或方法的抽样。focus参数应该是一个正则表达式。 |
| web       | [focus]          |         | 与gv命令类似，web命令也会用图形化的方式来显示概要文件。但不同的是，web命令是在一个Web浏览器中显示它。如果你的Web浏览器已经启动，那么它的显示速度会非常快。如果想改变所使用的Web浏览器，可以在Linux下设置符号链接/etc/alternatives/gnome-www-browser或/etc/alternatives/x-www-browser，或在OS X下改变SVG文件的关联Finder。 |
| list      | [routine_regexp] |         | 列出名称与参数“routine_regexp”代表的正则表达式相匹配的函数或方法的相关源代码。 |
| weblist   | [routine_regexp] |         | 在Web浏览器中显示与list命令的输出相同的内容。它与list命令相比的优势是，在我们点击某行源码时还可以显示相应的汇编代码。 |
| top[N]    |                  | [--cum] | top命令可以以本地取样计数为顺序列出函数或方法及相关信息。如果存在标记“--cum”则以累积取样计数为顺序。默认情况下top命令会列出前10项内容。但是如果在top命令后面紧跟一个数字，那么其列出的项数就会与这个数字相同。 |
| disasm    | [routine_regexp] |         | 显示名称与参数“routine_regexp”相匹配的函数或方法的反汇编代码。并且，在显示的内容中还会标注有相应的取样计数。 |
| callgrind | [filename]       |         | 利用callgrind工具生成统计文件。在这个文件中，说明了程序中函数的调用情况。如果未指定“filename”参数，则直接调用kcachegrind工具。kcachegrind可以以可视化的方式查看callgrind工具生成的统计文件。 |
| help      |                  |         | 显示帮助信息。                                               |
| quit      |                  |         | 退出go tool pprof命令。Ctrl-d也可以达到同样效果。            |