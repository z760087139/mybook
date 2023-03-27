# 类型笔记

### 管道

* nil 管道可以读写,但永久堵塞；已关闭管道只读，write 会panic
* 无缓冲区管道读写都会堵塞
* 缓冲区为1的管道，可以实现互斥锁。写入=上锁，读取=解锁

#### 源码位置

src/runtime/chan.go

```go
type hchan struct {
	qcount   uint           // total data in the queue 队列中的元素个数
	dataqsiz uint           // size of the circular queue 环形队列长度（可存放元素个数）
	buf      unsafe.Pointer // points to an array of dataqsiz elements 环形队列指针
	elemsize uint16  // 每个元素的大小
	closed   uint32  // 关闭状态标识
	elemtype *_type // element type 元素类型
	sendx    uint   // send index 队列下标，指元素写入到管道时的存放到队列中的位置
	recvx    uint   // receive index 队列下标，指从管道读取时在队列中的位置
	recvq    waitq  // list of recv waiters 等待读取消息的g 队列
	sendq    waitq  // list of send waiters 等待写入的g队列

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}

```

#### 管道常规操作

```go
var ch chan string

// 读取方式一
func f1(){
  for value := range ch {
    // ...
  }
  // 循环结束标识管道已关闭且已读取所有内容
}

// 读取方式二
func f2() {
  for {
    select {
      case value,ok := <- ch :
        if !ok {
         // 管道已关闭且无可读取内容
         return
        }
    default:      
    }
  }
}

```

### 切片

切片本身只是一个结构体，存放了数组指针，长度，容量信息

```go
type slice struct {
   array unsafe.Pointer
   len   int
   cap   int
}
```

**append 函数**

由于切片只是一个结构体，通过 append 追加元素时，实际上为 **值传递** , append 对复制的切片对应的数组指针内容追加内容，同时修改 **复制后的切片**的长度，并将复制后的切片返回

所以切片通过 append 扩容/追加后，需要使用返回的值进行覆盖动作。

```go
func f1(){
  a := make([]int,0,10)
  // 重新赋值 a
  a = append(a,1)
}
```

### Map

Map 是实际为一个 Hash 表

```go
const (
	// Maximum number of key/elem pairs a bucket can hold.
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits
)

type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}

// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
  
  // 非显式内容
  data []byte // key/key/key.../value/value/.. 形式存放数据
  overflow *bmap // 溢出 bucket 的地址
}
```

bucket 长度为 2^B，每个 bucket 存放8个键值对

#### hash冲突

如果bucket 的8个键值对已满，但是由于hash冲突需要追加内容，则会创建溢出 bucket 并记录对应的地址到 overflow，形成链式记录

#### 负载因子

衡量 hash 表冲突的情况

```
负载因子 = 键数量 / bucket 数量
```

GO语言map由于bucket 能存放8个键值对，设计负载因子在 6.5时候触发 rehash

> redis bucket 键值对只有一个，所以负载因子为1的时候会触发 rehash

#### 扩容

* 负载因子达到 6.5
* overflow 达到 2^(min 15, B)

**增量扩容**

增量扩容会更新 map 的 buckets 属性，创建新的 buckets 列表。

GO采用逐步迁移方式。**每次访问触发一次迁移，每次迁移两个键值对**

**等量扩容**

overflow 过多时，整理bucket 内容使用

#### 增删改查

**查询**

1. 计算 Hash 值
2. Hash 低位与 B 取模确定 bucket 位置
3. Hash 高位到 tophash 查询
4. 尝试对比 tophash\[i] 的 Hash 值，如果相同则获取key对比
5. 当前bucket 未找到则到溢出bucket查找

> 迁移过程中优先从 oldbuckets 查找
>
> 找不到 则返回零值

**添加、更新**

1. 执行查询动作
2. 找到则更新、没有则插入空余位置

**删除**

1. 执行查询动作
2. 删除元素

### Struct

```go
type Kid struct {
  Name string
  Age int
}

func (k Kid) SetName(name string) {
  // 错误使用，无法修改
  k.Name = name
}

func (k *Kid) SetAge(age int) {
  k.Age = age
}
```

类型的方法定义时，如果出现需要修改值的时候，需要用指针进行定义

#### 内嵌字段/隐式继承

```go
type Animal struct {
	Name string
}

func (a *Animal) SetName(name string){
	a.Name = name
}

type Cat struct {
	Animal // 继承 SetName 方法，直接能使用 SetName 方法 和 Name 属性
}

type Dog struct {
	a Animal
}
```

隐式声明可以继承struct 原来的字段和方法，显式声明不行

### String

string 使用 8 bit 字节的集合存储字符，而存储的是 UTF-8 编码；汉字字符的UTF-8编码占用多个字节

**len(string) 返回的是 字节长度**，for range 的 index 返回的是字符首个字节的下标，所以for range 汉字可能返回的 index 并不连续

```go
func f1(){
  s := "中国"
  for index,value := range s {
    fmt.Printf("index:%v, value:%v\n", index, value)
    // index:0, value: 中
    // index:3, value: 国
  }
}
```
