# 内存管理

> 程序中的数据和变量都会被分配到程序所在的虚拟内存中，内存空间包含两个重要区域：栈区（Stack）和堆区（Heap）。
>
> 函数调用的参数、返回值以及局部变量大都会被分配到栈上，这部分内存会由编译器进行管理；
>
> 堆中的对象由内存分配器分配并由垃圾收集器回收。

## 内存分配器

内存管理一般包含三个不同的组件，分别是用户程序（Mutator）、分配器（Allocator）和收集器（Collector）[1](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#fn:1)，当用户程序申请内存时，它会通过内存分配器申请新内存，而分配器会负责从堆中初始化相应的内存区域。

![mutator-allocator-collector](https://img.draveness.me/2020-02-29-15829868066411-mutator-allocator-collector.png)

### 分配方式

#### 线性分配器

只需要在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置，即移动下图中的指针

![bump-allocator](https://img.draveness.me/2020-02-29-15829868066435-bump-allocator.png)

无法重复利用已删除内存，需要与合适的垃圾回收算法配合使用，例如：标记压缩（Mark-Compact）、复制回收（Copying GC）和分代回收（Generational GC）等算法

#### 空闲链表分配器

在内部会维护一个类似链表的数据结构。当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表

![free-list-allocator](https://img.draveness.me/2020-02-29-15829868066446-free-list-allocator.png)

* 首次适应（First-Fit）— 从链表头开始遍历，选择第一个大小大于申请内存的内存块；
* 循环首次适应（Next-Fit）— 从上次遍历的结束位置开始遍历，选择第一个大小大于申请内存的内存块；
* 最优适应（Best-Fit）— 从链表头遍历整个链表，选择最合适的内存块；
* **隔离适应**（Segregated-Fit）— 将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块；

![segregated-list](https://img.draveness.me/2020-02-29-15829868066452-segregated-list.png)

#### 线程缓存分配（Thread-Caching Malloc，TCMalloc）

**多级缓存** [**#**](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#%E5%A4%9A%E7%BA%A7%E7%BC%93%E5%AD%98)

内存分配器不仅会区别对待大小不同的对象，还会将内存分成不同的级别分别管理，TCMalloc 和 Go 运行时分配器都会引入线程缓存（Thread Cache）、中心缓存（Central Cache）和页堆（Page Heap）三个组件分级管理内存

![multi-level-cache](https://img.draveness.me/2020-02-29-15829868066457-multi-level-cache.png)
