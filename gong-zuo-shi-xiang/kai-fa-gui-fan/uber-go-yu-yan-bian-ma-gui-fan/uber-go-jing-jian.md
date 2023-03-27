# uber go 精简

### 指导原则

#### Interface 合理性验证

在编译时验证接口的符合性。这包括：

* 将实现特定接口的导出类型作为接口 API 的一部分进行检查
* 实现同一接口的 (导出和非导出) 类型属于实现类型的集合
* 任何违反接口合理性检查的场景，都会终止编译，并通知给用户

补充：上面 3 条是编译器对接口的检查机制， 大体意思是错误使用接口会在编译期报错。 所以可以利用这个机制让部分问题在编译期暴露。

| Bad                                                                                                                                                     | Good                                                                                                                                                                                                                  |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `// 如果 Handler 没有实现 http.Handler，会在运行时报错 type Handler struct { // ... } func (h *Handler) ServeHTTP( w http.ResponseWriter, r *http.Request, ) { ... }` | `type Handler struct { // ... } // 用于触发编译期的接口的合理性检查机制 // 如果 Handler 没有实现 http.Handler，会在编译期报错 var _ http.Handler = (*Handler)(nil) func (h *Handler) ServeHTTP( w http.ResponseWriter, r *http.Request, ) { // ... }` |

如果 `*Handler` 与 `http.Handler` 的接口不匹配， 那么语句 `var _ http.Handler = (*Handler)(nil)` 将无法编译通过。

赋值的右边应该是断言类型的零值。 对于指针类型（如 `*Handler`）、切片和映射，这是 `nil`； 对于结构类型，这是空结构。

```go
type LogHandler struct {
  h   http.Handler
  log *zap.Logger
}
var _ http.Handler = LogHandler{}
func (h LogHandler) ServeHTTP(
  w http.ResponseWriter,
  r *http.Request,
) {
  // ...
}
```

#### 零值 Mutex 是有效的

零值 `sync.Mutex` 和 `sync.RWMutex` 是有效的。所以指向 mutex 的指针基本是不必要的。

| Bad                               | Good                          |
| --------------------------------- | ----------------------------- |
| `mu := new(sync.Mutex) mu.Lock()` | `var mu sync.Mutex mu.Lock()` |

#### 在边界处拷贝 Slices 和 Maps

slices 和 maps 包含了指向底层数据的指针，因此在需要复制它们时要特别注意。

**接收 Slices 和 Maps**

请记住，当 map 或 slice 作为函数参数传入时，如果您存储了对它们的引用，则用户可以对其进行修改。

| Bad                                                                                                                               | Good                                                                                                                                                                                    |
| --------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `func (d *Driver) SetTrips(trips []Trip) { d.trips = trips } trips := ... d1.SetTrips(trips) // 你是要修改 d1.trips 吗？ trips[0] = ...` | `func (d *Driver) SetTrips(trips []Trip) { d.trips = make([]Trip, len(trips)) copy(d.trips, trips) } trips := ... d1.SetTrips(trips) // 这里我们修改 trips[0]，但不会影响到 d1.trips trips[0] = ...` |

**返回 slices 或 maps**

同样，请注意用户对暴露内部状态的 map 或 slice 的修改。

| Bad                                                                                                                                                                                                                                                                                       | Good                                                                                                                                                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type Stats struct { mu sync.Mutex counters map[string]int } // Snapshot 返回当前状态。 func (s *Stats) Snapshot() map[string]int { s.mu.Lock() defer s.mu.Unlock() return s.counters } // snapshot 不再受互斥锁保护 // 因此对 snapshot 的任何访问都将受到数据竞争的影响 // 影响 stats.counters snapshot := stats.Snapshot()` | `type Stats struct { mu sync.Mutex counters map[string]int } func (s *Stats) Snapshot() map[string]int { s.mu.Lock() defer s.mu.Unlock() result := make(map[string]int, len(s.counters)) for k, v := range s.counters { result[k] = v } return result } // snapshot 现在是一个拷贝 snapshot := stats.Snapshot()` |

#### 使用 defer 释放资源

使用 defer 释放资源，诸如文件和锁。

| Bad                                                                                                                                               | Good                                                                                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `p.Lock() if p.count < 10 { p.Unlock() return p.count } p.count++ newCount := p.count p.Unlock() return newCount // 当有多个 return 分支时，很容易遗忘 unlock` | `p.Lock() defer p.Unlock() if p.count < 10 { return p.count } p.count++ return p.count // 更可读` |

Defer 的开销非常小，只有在您可以证明函数执行时间处于纳秒级的程度时，才应避免这样做。使用 defer 提升可读性是值得的，因为使用它们的成本微不足道。尤其适用于那些不仅仅是简单内存访问的较大的方法，在这些方法中其他计算的资源消耗远超过 `defer`。

#### 用 time 处理时间

时间处理很复杂。关于时间的错误假设通常包括以下几点。

1. 一天有 24 小时
2. 一小时有 60 分钟
3. 一周有七天
4. 一年 365 天
5. [还有更多](https://infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time)

例如，_1_ 表示在一个时间点上加上 24 小时并不总是产生一个新的日历日。

因此，在处理时间时始终使用 [`"time"`](https://golang.org/pkg/time/) 包，因为它有助于以更安全、更准确的方式处理这些不正确的假设。

**使用 `time.Time` 表达瞬时时间**

在处理时间的瞬间时使用 [`time.Time`](https://golang.org/pkg/time/#Time)，在比较、添加或减去时间时使用 `time.Time` 中的方法。

| Bad                                                                              | Good                                                                         |
| -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `func isActive(now, start, stop int) bool { return start <= now && now < stop }` | \`func isActive(now, start, stop time.Time) bool { return (start.Before(now) |

**使用 `time.Duration` 表达时间段**

在处理时间段时使用 [`time.Duration`](https://golang.org/pkg/time/#Duration) .

| Bad                                                                                                                  | Good                                                                                       |
| -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `func poll(delay int) { for { // ... time.Sleep(time.Duration(delay) * time.Millisecond) } } poll(10) // 是几秒钟还是几毫秒？` | `func poll(delay time.Duration) { for { // ... time.Sleep(delay) } } poll(10*time.Second)` |

回到第一个例子，在一个时间瞬间加上 24 小时，我们用于添加时间的方法取决于意图。如果我们想要下一个日历日 (当前天的下一天) 的同一个时间点，我们应该使用 [`Time.AddDate`](https://golang.org/pkg/time/#Time.AddDate)。但是，如果我们想保证某一时刻比前一时刻晚 24 小时，我们应该使用 [`Time.Add`](https://golang.org/pkg/time/#Time.Add)。

```
newDay := t.AddDate(0 /* years */, 0 /* months */, 1 /* days */)
maybeNewDay := t.Add(24 * time.Hour)
```

**对外部系统使用 `time.Time` 和 `time.Duration`**

尽可能在与外部系统的交互中使用 `time.Duration` 和 `time.Time` 例如 :

* Command-line 标志: [`flag`](https://golang.org/pkg/flag/) 通过 [`time.ParseDuration`](https://golang.org/pkg/time/#ParseDuration) 支持 `time.Duration`
* JSON: [`encoding/json`](https://golang.org/pkg/encoding/json/) 通过其 [`UnmarshalJSON` method](https://golang.org/pkg/time/#Time.UnmarshalJSON) 方法支持将 `time.Time` 编码为 [RFC 3339](https://tools.ietf.org/html/rfc3339) 字符串
* SQL: [`database/sql`](https://golang.org/pkg/database/sql/) 支持将 `DATETIME` 或 `TIMESTAMP` 列转换为 `time.Time`，如果底层驱动程序支持则返回
* YAML: [`gopkg.in/yaml.v2`](https://godoc.org/gopkg.in/yaml.v2) 支持将 `time.Time` 作为 [RFC 3339](https://tools.ietf.org/html/rfc3339) 字符串，并通过 [`time.ParseDuration`](https://golang.org/pkg/time/#ParseDuration) 支持 `time.Duration`。

当不能在这些交互中使用 `time.Duration` 时，请使用 `int` 或 `float64`，并在字段名称中包含单位。

例如，由于 `encoding/json` 不支持 `time.Duration`，因此该单位包含在字段的名称中。

| Bad                                                                        | Good                                                                                            |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `// {"interval": 2} type Config struct { Interval int` json:"interval" `}` | `// {"intervalMillis": 2000} type Config struct { IntervalMillis int` json:"intervalMillis" `}` |

#### Errors

**错误类型**

声明错误的选项很少。 在选择最适合您的用例的选项之前，请考虑以下事项。

* 调用者是否需要匹配错误以便他们可以处理它？ 如果是，我们必须通过声明顶级错误变量或自定义类型来支持 [`errors.Is`](https://golang.org/pkg/errors/#Is) 或 [`errors.As`](https://golang.org/pkg/errors/#As) 函数。
* 错误消息是否为静态字符串，还是需要上下文信息的动态字符串？ 如果是静态字符串，我们可以使用 [`errors.New`](https://golang.org/pkg/errors/#New)，但对于后者，我们必须使用 [`fmt.Errorf`](https://golang.org/pkg/fmt/#Errorf) 或自定义错误类型。
* 我们是否正在传递由下游函数返回的新错误？ 如果是这样，请参阅[错误包装部分](https://github.com/xxjwxc/uber\_go\_guide\_cn#%E9%94%99%E8%AF%AF%E5%8C%85%E8%A3%85)。

| 错误匹配？ | 错误消息    | 指导                                                                      |
| ----- | ------- | ----------------------------------------------------------------------- |
| No    | static  | [`errors.New`](https://golang.org/pkg/errors/#New)                      |
| No    | dynamic | [`fmt.Errorf`](https://golang.org/pkg/fmt/#Errorf)                      |
| Yes   | static  | top-level `var` with [`errors.New`](https://golang.org/pkg/errors/#New) |
| Yes   | dynamic | custom `error` type                                                     |

例如， 使用 [`errors.New`](https://golang.org/pkg/errors/#New) 表示带有静态字符串的错误。 如果调用者需要匹配并处理此错误，则将此错误导出为变量以支持将其与 `errors.Is` 匹配。

| 无错误匹配                                                                                                                                                                            | 错误匹配                                                                                                                                                                                                                                                                |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `// package foo func Open() error { return errors.New("could not open") } // package bar if err := foo.Open(); err != nil { // Can't handle the error. panic("unknown error") }` | `// package foo var ErrCouldNotOpen = errors.New("could not open") func Open() error { return ErrCouldNotOpen } // package bar if err := foo.Open(); err != nil { if errors.Is(err, foo.ErrCouldNotOpen) { // handle the error } else { panic("unknown error") } }` |

对于动态字符串的错误， 如果调用者不需要匹配它，则使用 [`fmt.Errorf`](https://golang.org/pkg/fmt/#Errorf)， 如果调用者确实需要匹配它，则自定义 `error`。

| 无错误匹配                                                                                                                                                                                                              | 错误匹配                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `// package foo func Open(file string) error { return fmt.Errorf("file %q not found", file) } // package bar if err := foo.Open("testfile.txt"); err != nil { // Can't handle the error. panic("unknown error") }` | `// package foo type NotFoundError struct { File string } func (e *NotFoundError) Error() string { return fmt.Sprintf("file %q not found", e.File) } func Open(file string) error { return &NotFoundError{File: file} } // package bar if err := foo.Open("testfile.txt"); err != nil { var notFound *NotFoundError if errors.As(err, &notFound) { // handle the error } else { panic("unknown error") } }` |

请注意，如果您从包中导出错误变量或类型， 它们将成为包的公共 API 的一部分。

**错误包装**

如果调用其他方法时出现错误, 通常有三种处理方式可以选择：

* 将原始错误原样返回
* 使用 `fmt.Errorf` 搭配 `%w` 将错误添加进上下文后返回
* 使用 `fmt.Errorf` 搭配 `%v` 将错误添加进上下文后返回

如果没有要添加的其他上下文，则按原样返回原始错误。 这将保留原始错误类型和消息。 这非常适合底层错误消息有足够的信息来追踪它来自哪里的错误。

否则，尽可能在错误消息中添加上下文 这样就不会出现诸如“连接被拒绝”之类的模糊错误， 您会收到更多有用的错误，例如“呼叫服务 foo：连接被拒绝”。

使用 `fmt.Errorf` 为你的错误添加上下文， 根据调用者是否应该能够匹配和提取根本原因，在 `%w` 或 `%v` 动词之间进行选择。

* 如果调用者应该可以访问底层错误，请使用 `%w`。 对于大多数包装错误，这是一个很好的默认值， 但请注意，调用者可能会开始依赖此行为。因此，对于包装错误是已知`var`或类型的情况，请将其作为函数契约的一部分进行记录和测试。
* 使用 `%v` 来混淆底层错误。 调用者将无法匹配它，但如果需要，您可以在将来切换到 `%w`。

在为返回的错误添加上下文时，通过避免使用"failed to"之类的短语来保持上下文简洁，当错误通过堆栈向上渗透时，它会一层一层被堆积起来：

| Bad                                                                                                 | Good                                                                               |
| --------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `s, err := store.New() if err != nil { return fmt.Errorf( "failed to create new store: %w", err) }` | `s, err := store.New() if err != nil { return fmt.Errorf( "new store: %w", err) }` |
| `failed to x: failed to y: failed to create new store: the error`                                   | `x: y: new store: the error`                                                       |

然而，一旦错误被发送到另一个系统，应该清楚消息是一个错误（例如`err` 标签或日志中的"Failed"前缀）。

#### 避免使用 `init()`

尽可能避免使用`init()`。当`init()`是不可避免或可取的，代码应先尝试：

1. 无论程序环境或调用如何，都要完全确定。
2. 避免依赖于其他`init()`函数的顺序或副作用。虽然`init()`顺序是明确的，但代码可以更改， 因此`init()`函数之间的关系可能会使代码变得脆弱和容易出错。
3. 避免访问或操作全局或环境状态，如机器信息、环境变量、工作目录、程序参数/输入等。
4. 避免`I/O`，包括文件系统、网络和系统调用。

不能满足这些要求的代码可能属于要作为`main()`调用的一部分（或程序生命周期中的其他地方）， 或者作为`main()`本身的一部分写入。特别是，打算由其他程序使用的库应该特别注意完全确定性， 而不是执行“init magic”

| Bad                                                                                                                                                                                                                    | Good                                                                                                                                                                                                                                                 |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type Foo struct { // ... } var _defaultFoo Foo func init() { _defaultFoo = Foo{ // ... } }`                                                                                                                           | `var _defaultFoo = Foo{ // ... } // or，为了更好的可测试性： var _defaultFoo = defaultFoo() func defaultFoo() Foo { return Foo{ // ... } }`                                                                                                                     |
| `type Config struct { // ... } var _config Config func init() { // Bad: 基于当前目录 cwd, _ := os.Getwd() // Bad: I/O raw, _ := ioutil.ReadFile( path.Join(cwd, "config", "config.yaml"), ) yaml.Unmarshal(raw, &_config) }` | `type Config struct { // ... } func loadConfig() Config { cwd, err := os.Getwd() // handle err raw, err := ioutil.ReadFile( path.Join(cwd, "config", "config.yaml"), ) // handle err var config Config yaml.Unmarshal(raw, &config) return config }` |

考虑到上述情况，在某些情况下，`init()`可能更可取或是必要的，可能包括：

* 不能表示为单个赋值的复杂表达式。
* 可插入的钩子，如`database/sql`、编码类型注册表等。
* 对 [Google Cloud Functions](https://cloud.google.com/functions/docs/bestpractices/tips#use\_global\_variables\_to\_reuse\_objects\_in\_future\_invocations) 和其他形式的确定性预计算的优化。

#### 追加时优先指定切片容量

追加时优先指定切片容量

在尽可能的情况下，在初始化要追加的切片时为`make()`提供一个容量值。

| Bad                                                                                                       | Good                                                                                                            |
| --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `for n := 0; n < b.N; n++ { data := make([]int, 0) for k := 0; k < size; k++{ data = append(data, k) } }` | `for n := 0; n < b.N; n++ { data := make([]int, 0, size) for k := 0; k < size; k++{ data = append(data, k) } }` |
| `BenchmarkBad-4 100000000 2.48s`                                                                          | `BenchmarkGood-4 100000000 0.21s`                                                                               |

### 命名规范

#### 相似的声明放在一组

Go 语言支持将相似的声明放在一个组内。

| Bad                     | Good                 |
| ----------------------- | -------------------- |
| `import "a" import "b"` | `import ( "a" "b" )` |

这同样适用于常量、变量和类型声明：

| Bad                                                                                 | Good                                                                             |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `const a = 1 const b = 2 var a = 1 var b = 2 type Area float64 type Volume float64` | `const ( a = 1 b = 2 ) var ( a = 1 b = 2 ) type ( Area float64 Volume float64 )` |

仅将相关的声明放在一组。不要将不相关的声明放在一组。

| Bad                                                                                         | Good                                                                                              |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `type Operation int const ( Add Operation = iota + 1 Subtract Multiply EnvVar = "MY_ENV" )` | `type Operation int const ( Add Operation = iota + 1 Subtract Multiply ) const EnvVar = "MY_ENV"` |

分组使用的位置没有限制，例如：你可以在函数内部使用它们：

| Bad                                                                                                           | Good                                                                                                               |
| ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `func f() string { red := color.New(0xff0000) green := color.New(0x00ff00) blue := color.New(0x0000ff) ... }` | `func f() string { var ( red = color.New(0xff0000) green = color.New(0x00ff00) blue = color.New(0x0000ff) ) ... }` |

例外：如果变量声明与其他变量相邻，则应将变量声明（尤其是函数内部的声明）分组在一起。对一起声明的变量执行此操作，即使它们不相关。

| Bad                                                                                                              | Good                                                                                                              |
| ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `func (c *client) request() { caller := c.name format := "json" timeout := 5*time.Second var err error // ... }` | `func (c *client) request() { var ( caller = c.name format = "json" timeout = 5*time.Second err error ) // ... }` |

#### 包名

当命名包时，请按下面规则选择一个名称：

* 全部小写。没有大写或下划线。
* 大多数使用命名导入的情况下，不需要重命名。
* 简短而简洁。请记住，在每个使用的地方都完整标识了该名称。
* 不用复数。例如`net/url`，而不是`net/urls`。
* 不要用“common”，“util”，“shared”或“lib”。这些是不好的，信息量不足的名称。

另请参阅 [Go 包命名规则](https://blog.golang.org/package-names) 和 [Go 包样式指南](https://rakyll.org/style-packages/).

#### 函数名

我们遵循 Go 社区关于使用 [MixedCaps 作为函数名](https://golang.org/doc/effective\_go.html#mixed-caps) 的约定。有一个例外，为了对相关的测试用例进行分组，函数名可能包含下划线，如：`TestMyFunction_WhatIsBeingTested`.

#### 导入别名

如果程序包名称与导入路径的最后一个元素不匹配，则必须使用导入别名。

```
import (
  "net/http"

  client "example.com/client-go"
  trace "example.com/trace/v2"
)
```

在所有其他情况下，除非导入之间有直接冲突，否则应避免导入别名。

| Bad                                                   | Good                                                                  |
| ----------------------------------------------------- | --------------------------------------------------------------------- |
| `import ( "fmt" "os" nettrace "golang.net/x/trace" )` | `import ( "fmt" "os" "runtime/trace" nettrace "golang.net/x/trace" )` |

#### 函数分组与顺序

* 函数应按粗略的调用顺序排序。
* 同一文件中的函数应按接收者分组。

因此，导出的函数应先出现在文件中，放在`struct`, `const`, `var`定义的后面。

在定义类型之后，但在接收者的其余方法之前，可能会出现一个 `newXYZ()`/`NewXYZ()`

由于函数是按接收者分组的，因此普通工具函数应在文件末尾出现。

| Bad                                                                                                                                                                                                               | Good                                                                                                                                                                                                              |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `func (s *something) Cost() { return calcCost(s.weights) } type something struct{ ... } func calcCost(n []int) int {...} func (s *something) Stop() {...} func newSomething() *something { return &something{} }` | `type something struct{ ... } func newSomething() *something { return &something{} } func (s *something) Cost() { return calcCost(s.weights) } func (s *something) Stop() {...} func calcCost(n []int) int {...}` |

#### 减少嵌套

代码应通过尽可能先处理错误情况/特殊情况并尽早返回或继续循环来减少嵌套。减少嵌套多个级别的代码的代码量。

| Bad                                                                                                                                                                  | Good                                                                                                                                                        |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `for _, v := range data { if v.F1 == 1 { v = process(v) if err := v.Call(); err == nil { v.Send() } else { return err } } else { log.Printf("Invalid v: %v", v) } }` | `for _, v := range data { if v.F1 != 1 { log.Printf("Invalid v: %v", v) continue } v = process(v) if err := v.Call(); err != nil { return err } v.Send() }` |

#### 顶层变量声明

在顶层，使用标准`var`关键字。请勿指定类型，除非它与表达式的类型不同。

| Bad                                                  | Good                                                                                              |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `var _s string = F() func F() string { return "A" }` | `var _s = F() // 由于 F 已经明确了返回一个字符串类型，因此我们没有必要显式指定_s 的类型 // 还是那种类型 func F() string { return "A" }` |

如果表达式的类型与所需的类型不完全匹配，请指定类型。

```
type myError struct{}

func (myError) Error() string { return "error" }

func F() myError { return myError{} }

var _e error = F()
// F 返回一个 myError 类型的实例，但是我们要 error 类型
```

#### 对于未导出的顶层常量和变量，使用\_作为前缀

在未导出的顶级`vars`和`consts`， 前面加上前缀\_，以使它们在使用时明确表示它们是全局符号。

基本依据：顶级变量和常量具有包范围作用域。使用通用名称可能很容易在其他文件中意外使用错误的值。

| Bad                                                                                                                                                                                                                                  | Good                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------- |
| `// foo.go const ( defaultPort = 8080 defaultUser = "user" ) // bar.go func Bar() { defaultPort := 9090 ... fmt.Println("Default port", defaultPort) // We will not see a compile error if the first line of // Bar() is deleted. }` | `// foo.go const ( _defaultPort = 8080 _defaultUser = "user" )` |
