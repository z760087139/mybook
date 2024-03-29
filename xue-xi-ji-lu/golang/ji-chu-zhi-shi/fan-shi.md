# 泛式

#### 说明例子

**形参(parameter)** 和 **实参(argument)** 这一基本概念：

```go
func Add(a int, b int) int {  
    // 变量a,b是函数的形参   "a int, b int" 这一串被称为形参列表
    return a + b
}

Add(100,200) // 调用函数时，传入的100和200是实参
```

将 **形参 实参** 这个概念推广一下，给变量的类型也引入和类似形参实参的概念的话，问题就迎刃而解：在这里我们将其称之为 **类型形参(type parameter)** 和 **类型实参(type argument)**，如下：

```go
// 假设 T 是类型形参，在定义函数时它的类型是不确定的，类似占位符
func Add(a T, b T) T {  
    return a + b
}
```

在上面这段伪代码中， T 被称为 **类型形参(type parameter)**， 它不是具体的类型，在定义函数时类型并不确定。因为 T 的类型并不确定，所以我们需要像函数的形参那样，在调用函数的时候再传入具体的类型。这样我们不就能一个函数同时支持多个不同的类型了吗？在这里被传入的具体类型被称为 **类型实参(type argument)**

```go
// [T=int]中的 int 是类型实参，代表着函数Add()定义中的类型形参 T 全都被 int 替换
Add[T=int](100, 200)  
// 传入类型实参int后，Add()函数的定义可近似看成下面这样：
func Add( a int, b int) int {
    return a + b
}

// 另一个例子：当我们想要计算两个字符串之和的时候，就传入string类型实参
Add[T=string]("Hello", "World") 
// 类型实参string传入后，Add()函数的定义可近似视为如下
func Add( a string, b string) string {
    return a + b
}
```

通过引入 **类型形参** 和 **类型实参** 这两个概念，我们让一个函数获得了处理多种不同类型数据的能力，这种编程方式被称为 **泛型编程**。

#### 泛式类型

单纯的形参实参是远远不能实现泛型编程的，所以Go还引入了非常多全新的概念：

* 类型形参 (Type parameter)
* 类型实参(Type argument)
* 类型形参列表( Type parameter list)
* 类型约束(Type constraint)
* 实例化(Instantiations)
* 泛型类型(Generic type)
* 泛型接收器(Generic receiver)
* 泛型函数(Generic function)

```go
type Slice[T int|float32|float64 ] []T
```

不同于一般的类型定义，这里类型名称 `Slice` 后带了中括号，对各个部分做一个解说就是：

* `T` 就是上面介绍过的**类型形参(Type parameter)**，在定义Slice类型的时候 T 代表的具体类型并不确定，类似一个占位符
* `int|float32|float64` 这部分被称为**类型约束(Type constraint)**，中间的 `|` 的意思是告诉编译器，类型形参 T 只可以接收 int 或 float32 或 float64 这三种类型的实参
* 中括号里的 `T int|float32|float64` 这一整串因为定义了所有的类型形参(在这个例子里只有一个类型形参T），所以我们称其为 **类型形参列表(type parameter list)**
* 这里新定义的类型名称叫 `Slice[T]`
* 类型定义中带 **类型形参** **的类型，称之为** **泛型类型(Generic type)**

泛型类型不能直接拿来使用，必须传入**类型实参(Type argument)** 将其确定为具体的类型之后才可使用。而传入类型实参确定具体类型的操作被称为 **实例化(Instantiations)**

**泛式函数**

```go
func Add[T int | float32 | float64](a T, b T) T {
    return a + b
}
Add[int](1,2) // 传入类型实参int，计算结果为 3
Add[float32](1.0, 2.0) // 传入类型实参float32, 计算结果为 3.0

Add[string]("hello", "world") // 错误。因为泛型函数Add的类型约束中并不包含string
```

或许你会觉得这样每次都要手动指定类型实参太不方便了。所以Go还支持类型实参的自动推导：

```go
Add(1, 2)  // 1，2是int类型，编译请自动推导出类型实参T是int
Add(1.0, 2.0) // 1.0, 2.0 是浮点，编译请自动推导出类型实参T是float32
```

#### 几种错误写法

1.  定义泛型类型的时候，**基础类型不能只有类型形参**，如下：

    ```go
    // 错误，类型形参不能单独使用
    type CommonType[T int|string|float32] T
    ```
2.  当类型约束的一些写法会被编译器误认为是表达式时会报错。如下：

    ```go
    //✗ 错误。T *int会被编译器误认为是表达式 T乘以int，而不是int指针
    type NewType[T *int] []T
    // 上面代码再编译器眼中：它认为你要定义一个存放切片的数组，数组长度由 T 乘以 int 计算得到
    type NewType [T * int][]T 

    //✗ 错误。和上面一样，这里不光*被会认为是乘号，| 还会被认为是按位或操作
    type NewType2[T *int|*float64] []T 

    //✗ 错误
    type NewType2 [T (int)] []T 
    ```

    为了避免这种误解，解决办法就是给类型约束包上 `interface{}` 或加上逗号消除歧义（关于接口具体的用法会在后半篇提及）

    ```go
    type NewType[T interface{*int}] []T
    type NewType2[T interface{*int|*float64}] []T 

    // 如果类型约束中只有一个类型，可以添加个逗号消除歧义
    type NewType3[T *int,] []T

    //✗ 错误。如果类型约束不止一个类型，加逗号是不行的
    type NewType4[T *int|*float32,] []T 
    ```

    因为上面逗号的用法限制比较大，这里推荐统一用 interface{} 解决问题
3.  匿名结构体不支持泛式

    ```go
    // 错误，不支持匿名结构体泛式
    testCase := struct[T int|string] {
            caseName string
            got      T
            want     T
        }[int]{
            caseName: "test OK",
            got:      100,
            want:     100,
        }
    ```
4. 匿名函数不支持泛式
5. 不支持泛式方法

#### 关于接口(interface)的调整

泛式的类型可以通过 interface 进行定义

interface 的含义存 methods set 改为了 type set

> An interface type specifies a **method set** called its interface
>
> An interface type defines a \***type set\*** _(一个_接口类型定义了一个类型集)

```go
// ~ 符号表示底层类型
type Int interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type Uint interface {
    ~uint | ~uint8 | ~uint16 | ~uint32
}
type Float interface {
    ~float32 | ~float64
}

type Slice[T Int | Uint | Float] []T 

var s Slice[int] // 正确

type MyInt int
var s2 Slice[MyInt]  // MyInt底层类型是int，所以可以用于实例化

type MyMyInt MyInt
var s3 Slice[MyMyInt]  // 正确。MyMyInt 虽然基于 MyInt ，但底层类型也是int，所以也能用于实例化

type MyFloat32 float32  // 正确
var s4 Slice[MyFloat32]
```

**类型交集**

```go
type AllInt interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint32
}

type Uint interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

type A interface { // 接口A代表的类型集是 AllInt 和 Uint 的交集
    AllInt
    Uint
}

type B interface { // 接口B代表的类型集是 AllInt 和 ~int 的交集
    AllInt
    ~int
}
```

**空接口(interface{})与关键词any**

interface{} 表示所有类型的集合，并非空集合，1.18新增关键词 any 表示等价与 interface{}

```go
// 空接口代表所有类型的集合。写入类型约束意味着所有类型都可拿来做类型实参
type Slice[T interface{}] []T

var s1 Slice[int]    // 正确
var s2 Slice[map[string]string]  // 正确
var s3 Slice[chan int]  // 正确
var s4 Slice[interface{}]  // 正确
```

**comparable(可比较) 和 可排序(ordered)**

对于一些数据类型，我们需要在类型约束中限制只接受能 `!=` 和 `==` 对比的类型，如map：

```go
// 错误。因为 map 中键的类型必须是可进行 != 和 == 比较的类型
type MyMap[KEY any, VALUE any] map[KEY]VALUE 
```

所以Go直接内置了一个叫 `comparable` 的接口，它代表了所有可用 `!=` 以及 `==` 对比的类型：

```go
type MyMap[KEY comparable, VALUE any] map[KEY]VALUE // 正确
```

`comparable` 比较容易引起误解的一点是很多人容易把他与可排序搞混淆。可比较指的是 可以执行 `!=` `==` 操作的类型，并没确保这个类型可以执行大小比较（ `>,<,<=,>=` ）。如下：

```go
type OhMyStruct struct {
    a int
}

var a, b OhMyStruct

a == b // 正确。结构体可使用 == 进行比较
a != b // 正确

a > b // 错误。结构体不可比大小
```

而可进行大小比较的类型被称为 `Orderd` 。目前Go语言并没有像 `comparable` 这样直接内置对应的关键词，所以想要的话需要自己来定义相关接口，比如我们可以参考Go官方包`golang.org/x/exp/constraints` 如何定义：

```go
// Ordered 代表所有可比大小排序的类型
type Ordered interface {
    Integer | Float | ~string
}

type Integer interface {
    Signed | Unsigned
}

type Signed interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type Unsigned interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

type Float interface {
    ~float32 | ~float64
}
```

> 💡 这里虽然可以直接使用官方包 [golang.org/x/exp/constraints](https://link.segmentfault.com/?enc=Yk%2FSZ1CGYBZDH5cXum2hPg%3D%3D.yt%2FZfT72qRWHDiQMs1XXiar7MwcSSdlcQYF%2FlJfpRSUAqwb7SyEE%2Fn2Fsf4DrAJD) ，但因为这个包属于实验性质的 x 包，今后可能会发生非常大变动，所以并不推荐直接使用
