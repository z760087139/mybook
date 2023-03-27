# uber go 缩略

### 指导原则

#### Interface 合理性验证

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

```go
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = make([]Trip, len(trips))
  copy(d.trips, trips)
}

trips := ...
d1.SetTrips(trips)

// 这里我们修改 trips[0]，但不会影响到 d1.trips
trips[0] = ...
```

**返回 slices 或 maps**

同样，请注意用户对暴露内部状态的 map 或 slice 的修改。

```go
type Stats struct {
  mu sync.Mutex

  counters map[string]int
}

func (s *Stats) Snapshot() map[string]int {
  s.mu.Lock()
  defer s.mu.Unlock()

  result := make(map[string]int, len(s.counters))
  for k, v := range s.counters {
    result[k] = v
  }
  return result
}

// snapshot 现在是一个拷贝
snapshot := stats.Snapshot()
```

#### 使用 defer 释放资源

使用 defer 释放资源，诸如文件和锁。

#### 用 time 处理时间

**使用 `time.Time` 表达瞬时时间**

在处理时间的瞬间时使用 [`time.Time`](https://golang.org/pkg/time/#Time)，在比较、添加或减去时间时使用 `time.Time` 中的方法。

```go
func isActive(now, start, stop time.Time) bool {
  return (start.Before(now) || start.Equal(now)) && now.Before(stop) 
}
```

在一个时间瞬间加上 24 小时，我们用于添加时间的方法取决于意图。如果我们想要下一个日历日 (当前天的下一天) 的同一个时间点，我们应该使用 [`Time.AddDate`](https://golang.org/pkg/time/#Time.AddDate)。但是，如果我们想保证某一时刻比前一时刻晚 24 小时，我们应该使用 [`Time.Add`](https://golang.org/pkg/time/#Time.Add)。

```go
newDay := t.AddDate(0 /* years */, 0 /* months */, 1 /* days */)
maybeNewDay := t.Add(24 * time.Hour)
```

**对外部系统使用 `time.Time` 和 `time.Duration`**

* Command-line 标志: [`flag`](https://golang.org/pkg/flag/) 通过 [`time.ParseDuration`](https://golang.org/pkg/time/#ParseDuration) 支持 `time.Duration`
* JSON: [`encoding/json`](https://golang.org/pkg/encoding/json/) 通过其 [`UnmarshalJSON` method](https://golang.org/pkg/time/#Time.UnmarshalJSON) 方法支持将 `time.Time` 编码为 [RFC 3339](https://tools.ietf.org/html/rfc3339) 字符串
* SQL: [`database/sql`](https://golang.org/pkg/database/sql/) 支持将 `DATETIME` 或 `TIMESTAMP` 列转换为 `time.Time`，如果底层驱动程序支持则返回
* YAML: [`gopkg.in/yaml.v2`](https://godoc.org/gopkg.in/yaml.v2) 支持将 `time.Time` 作为 [RFC 3339](https://tools.ietf.org/html/rfc3339) 字符串，并通过 [`time.ParseDuration`](https://golang.org/pkg/time/#ParseDuration) 支持 `time.Duration`。

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

在某些情况下，`init()`可能更可取或是必要的，可能包括：

* 不能表示为单个赋值的复杂表达式。
* 可插入的钩子，如`database/sql`、编码类型注册表等。
* 对 [Google Cloud Functions](https://cloud.google.com/functions/docs/bestpractices/tips#use\_global\_variables\_to\_reuse\_objects\_in\_future\_invocations) 和其他形式的确定性预计算的优化。

#### 追加时优先指定切片容量

追加时优先指定切片容量

在尽可能的情况下，在初始化要追加的切片时为`make()`提供一个容量值。

### 规范

#### 相似的声明放在一组

```go
const (
  a = 1
  b = 2
)
var (
  a = 1
  b = 2
)
type (
  Area float64
  Volume float64
)
func (c *client) request() {
  var (
    caller  = c.name
    format  = "json"
    timeout = 5*time.Second
    err error
  )
  // ...
}
```

#### 避免使用内置名称

Go [语言规范](https://golang.org/ref/spec) 概述了几个内置的， 不应在 Go 项目中使用的 [预先声明的标识符](https://golang.org/ref/spec#Predeclared\_identifiers)。

#### 包名

当命名包时，请按下面规则选择一个名称：

* 全部小写。没有大写或下划线。
* 大多数使用命名导入的情况下，不需要重命名。
* 简短而简洁。请记住，在每个使用的地方都完整标识了该名称。
* 不用复数。例如`net/url`，而不是`net/urls`。
* 不要用“common”，“util”，“shared”或“lib”。这些是不好的，信息量不足的名称。

#### 导入别名

如果程序包名称与导入路径的最后一个元素不匹配，则必须使用导入别名。

```
import (
  "net/http"

  client "example.com/client-go"
  trace "example.com/trace/v2"
)
```

#### 函数名

遵循 Go 社区关于使用 [MixedCaps 作为函数名](https://golang.org/doc/effective\_go.html#mixed-caps) 的约定。有一个例外，为了对相关的测试用例进行分组，函数名可能包含下划线，如：`TestMyFunction_WhatIsBeingTested`.

#### 函数分组与顺序

* 函数应按粗略的调用顺序排序。
* 同一文件中的函数应按接收者分组。

因此，导出的函数应先出现在文件中，放在`struct`, `const`, `var`定义的后面。

在定义类型之后，但在接收者的其余方法之前，可能会出现一个 `newXYZ()`/`NewXYZ()`

由于函数是按接收者分组的，因此普通工具函数应在文件末尾出现

```go
type something struct{ ... }

func newSomething() *something {
    return &something{}
}

func (s *something) Cost() {
  return calcCost(s.weights)
}

func (s *something) Stop() {...}

func calcCost(n []int) int {...}
```

#### 对于未导出的顶层常量和变量，使用\_作为前缀

在未导出的顶级`vars`和`consts`， 前面加上前缀\_，以使它们在使用时明确表示它们是全局符号

```go
const (
  _defaultPort = 8080
  _defaultUser = "user"
)
```

#### nil 是一个有效的 slice

*   您不应明确返回长度为零的切片。应该返回`nil` 来代替。

    ```go
    if x == "" {
      return nil 
    }
    ```
*   要检查切片是否为空，请始终使用`len(s) == 0`。而非 `nil`。

    ```go
    func isEmpty(s []string) bool {
      return len(s) == 0 
    }
    ```
*   零值切片（用`var`声明的切片）可立即使用，无需调用`make()`创建。

    ```go
    var nums []int 
    if add1 {
      nums = append(nums, 1) 
    } if add2 {
      nums = append(nums, 2) 
    }
    ```

记住，虽然 nil 切片是有效的切片，但它不等于长度为 0 的切片（一个为 nil，另一个不是），并且在不同的情况下（例如序列化），这两个切片的处理方式可能不同。

#### 单元测试

#### 表驱动测试

当测试逻辑是重复的时候，通过 [subtests](https://blog.golang.org/subtests) 使用 table 驱动的方式编写 case 代码看上去会更简洁。(goland 能够自动生成样例)

```go
// func TestSplitHostPort(t *testing.T)

tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  {
    give:     "192.0.2.0:8000",
    wantHost: "192.0.2.0",
    wantPort: "8000",
  },
  {
    give:     "192.0.2.0:http",
    wantHost: "192.0.2.0",
    wantPort: "http",
  },
  {
    give:     ":8000",
    wantHost: "",
    wantPort: "8000",
  },
  {
    give:     "1:8",
    wantHost: "1",
    wantPort: "8",
  },
}

for _, tt := range tests {
  t.Run(tt.give, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.give)
    require.NoError(t, err)
    assert.Equal(t, tt.wantHost, host)
    assert.Equal(t, tt.wantPort, port)
  })
}
```

我们遵循这样的约定：将结构体切片称为`tests`。 每个测试用例称为`tt`。此外，我们鼓励使用`give`和`want`前缀说明每个测试用例的输入和输出值。

```go
tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  // ...
}

for _, tt := range tests {
  // ...
}
```

并行测试，比如一些专门的循环（例如，生成goroutine或捕获引用作为循环体的一部分的那些循环） 必须注意在循环的范围内显式地分配循环变量，以确保它们保持预期的值。

```
tests := []struct{
  give string
  // ...
}{
  // ...
}
for _, tt := range tests {
  tt := tt // for t.Parallel
  t.Run(tt.give, func(t *testing.T) {
    t.Parallel()
    // ...
  })
}
```

在上面的例子中，由于下面使用了`t.Parallel()`，我们必须声明一个作用域为循环迭代的`tt`变量。 如果我们不这样做，大多数或所有测试都会收到一个意外的`tt`值，或者一个在运行时发生变化的值。

#### 功能选项

功能选项是一种模式，您可以在其中声明一个不透明 Option 类型，该类型在某些内部结构中记录信息。您接受这些选项的可变编号，并根据内部结构上的选项记录的全部信息采取行动。

将此模式用于您需要扩展的构造函数和其他公共 API 中的可选参数，尤其是在这些功能上已经具有三个或更多参数的情况下。

| Bad                                                                                                                                                                                                | Good                                                                                                                                                                                                                                                 |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `// package db func Open( addr string, cache bool, logger *zap.Logger ) (*Connection, error) { // ... }`                                                                                           | `// package db type Option interface { // ... } func WithCache(c bool) Option { // ... } func WithLogger(log *zap.Logger) Option { // ... } // Open creates a connection. func Open( addr string, opts ...Option, ) (*Connection, error) { // ... }` |
| 必须始终提供缓存和记录器参数，即使用户希望使用默认值。`db.Open(addr, db.DefaultCache, zap.NewNop()) db.Open(addr, db.DefaultCache, log) db.Open(addr, false /* cache */, zap.NewNop()) db.Open(addr, false /* cache */, log)` | 只有在需要时才提供选项。`db.Open(addr) db.Open(addr, db.WithLogger(log)) db.Open(addr, db.WithCache(false)) db.Open( addr, db.WithCache(false), db.WithLogger(log), )`                                                                                           |

我们建议实现此模式的方法是使用一个 `Option` 接口，该接口保存一个未导出的方法，在一个未导出的 `options` 结构上记录选项。

```go
type options struct {
  cache  bool
  logger *zap.Logger
}

type Option interface {
  apply(*options)
}

type cacheOption bool

func (c cacheOption) apply(opts *options) {
  opts.cache = bool(c)
}

func WithCache(c bool) Option {
  return cacheOption(c)
}

type loggerOption struct {
  Log *zap.Logger
}

func (l loggerOption) apply(opts *options) {
  opts.logger = l.Log
}

func WithLogger(log *zap.Logger) Option {
  return loggerOption{Log: log}
}

// Open creates a connection.
func Open(
  addr string,
  opts ...Option,
) (*Connection, error) {
  options := options{
    cache:  defaultCache,
    logger: zap.NewNop(),
  }

  for _, o := range opts {
    o.apply(&options)
  }

  // ...
}
```

### SONAR

摘选部分sonar 需要特别注意的规则

1. 函数复杂度要求，主要针对 for / if / switch 等的层级复杂度要求。复杂度评分要求30以下
2. 常量替换重复字符串
3. if 判断条件复杂度要求
4. 单次嵌套深度要求
