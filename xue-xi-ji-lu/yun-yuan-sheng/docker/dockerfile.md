# dockerfile

#### CMD / ENTRYPOINT 命令

**exec 模式 / shell 模式**

* exec 模式

```docker
CMD ['top']
```

1. 启动进程号 1 的进程时使用的命令内容
2. 无法获取环境变量内容

```dock
CMD ["echo", "$HOME"] // 无 $HOME
CMD ["sh", "-c", "echo $HOME"] // 调用 shell 执行，可以识别环境变量
```

* shell 模式

使用 shell 模式时，docker 会以 /bin/sh -c "task command" 的方式执行任务命令。也就是说容器中的 1 号进程不是任务进程而是 bash 进程

```dock
CMD top
```

**CMD 命令**

为容器提供默认的执行命令

CMD 指令有三种使用方式

1. `CMD ["param1","param2"]` 为ENTRYPOINT 提供默认的参数
2. `CMD ["executable","param1","param2"]`exec 模式的写法，注意需要使用双引号
3. `CMD command param1 param2` shell 模式写法

**ENTRYPOINT**

1. 规则一：运行时追加参数

指定 ENTRYPOINT 指令为 exec 模式时，命令行上指定的参数会作为参数添加到 ENTRYPOINT 指定命令的参数列表中

```dockerfile
FROM ubuntu
ENTRYPOINT [ "top", "-b" ]
```

```shell
$ docker run --rm test1 -c
```

![img](https://images2018.cnblogs.com/blog/952033/201802/952033-20180223131125462-1226384921.png)

2. 规则二：使用 CMD 追加参数

```dockerfile
FROM ubuntu
ENTRYPOINT [ "top", "-b" ]
CMD [ "-c" ]
```

3. 规则三：shell 命令忽略追加参数

```dockerfile
FROM ubuntu
ENTRYPOINT echo $HOME 
```

![img](https://images2018.cnblogs.com/blog/952033/201802/952033-20180223131407554-1561884995.png)

4. 规则四：需要使用显示参数覆盖entrypoint 命令

```shell
docker run --rm --entrypoint hostname test2
```

#### 总结内容

[官方解析](https://docs.docker.com/engine/reference/builder/#understand-how-cmd-and-entrypoint-interact)

|                                   | No ENTRYPOINT                | ENTRYPOINT exec\_entry p1\_entry | ENTRYPOINT \[“exec\_entry”, “p1\_entry”]           |
| --------------------------------- | ---------------------------- | -------------------------------- | -------------------------------------------------- |
| **No CMD**                        | _error, not allowed_         | /bin/sh -c exec\_entry p1\_entry | exec\_entry p1\_entry                              |
| **CMD \[“exec\_cmd”, “p1\_cmd”]** | exec\_cmd p1\_cmd            | /bin/sh -c exec\_entry p1\_entry | exec\_entry p1\_entry exec\_cmd p1\_cmd            |
| **CMD \[“p1\_cmd”, “p2\_cmd”]**   | p1\_cmd p2\_cmd              | /bin/sh -c exec\_entry p1\_entry | exec\_entry p1\_entry p1\_cmd p2\_cmd              |
| **CMD exec\_cmd p1\_cmd**         | /bin/sh -c exec\_cmd p1\_cmd | /bin/sh -c exec\_entry p1\_entry | exec\_entry p1\_entry /bin/sh -c exec\_cmd p1\_cmd |
