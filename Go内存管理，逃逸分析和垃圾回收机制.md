# Go内存管理，逃逸分析和垃圾回收机制





# Go内存管理

## TCMalloc内存管理

Go的内存管理来自于tcmalloc，首先介绍几个概念：

- page：go和操作系统进行内存管理的单位，是8K大小的内存空间（64位系统）；
- span：一组连续的page构成了span，Span是比page更高一级的管理单位；
- ThreadCache：是各个进程各自的cache，一个cache包括多个空闲块链表，同一个链表上内存块大小是相等的，也可以说是根据内存块大小给内存块分了类（一共有67类），这样申请空闲块的时候，就可以快速从合适的链表选择内存块。由于每个进程都有一个ThreadCache，因此对ThreadCache的访问是无锁的；
- CentralCache：所有线程共享的缓存，也保存空闲内存块链表，链表数量与ThreadCache中链表数量相同，当ThreadCache中内存块不足时，可以从CentralCache中获取内存块；当ThreadCache内存块过多时，就可以放回CentralCache. 由于访问CentralCache是共享的，需要对其上锁。
- PageHeap：go中堆内存的抽象，里面存储的也是若干链表，链表保存的是span。当CentralCache中的内存块不足时，会从PageHeap中获取空闲的span，然后把一个span拆成若干内存块，添加到对应大小的链表中并分配内存；当CentralCache中内存块过多，则会将内存块放回PageHeap中。

![preview](Go内存管理，逃逸分析和垃圾回收机制.assets/v2-17a205c8fdfe0d21ab06f469df72b2fe_r.jpg)

![img](Go内存管理，逃逸分析和垃圾回收机制.assets/v2-8987e2cd5a39e7218b7a5c7689a1da7d_720w.jpg)

如图所示，从上至下依次是1页，2页。。。128页的span。还有存储大对象的large span set.

在tcmalloc中，小对象指0-256KB，中对象指256KB-1MB，大对象指1MB以上。

小对象的分配流程： ThreadCache->CentralCache->HeapPage，大部分时候ThreadCache对象都是足够 的，**没有系统调用和锁导致的上下文切换，因此效率非常高。**

中对象分配流程：直接在PageHeap中分配合适大小的内存块即可。

大对象分配：从large span set分配合适大小的页面组成span。

## Go内存管理

mheap对应于PageHeap

arenas对应于large span set（64MB on 64-bit and 4MB on 32-bit）

mcentral对应于CentralCache

mcahe对应于ThreadCache

> fixalloc: a free-list allocator for fixed-size off-heap objects, used to manage storage used by the allocator.
>
>mheap: the malloc heap, managed at page (8192-byte) granularity.
>
>mspan: a run of in-use pages managed by the mheap.
>
>mcentral: collects all spans of a given size class.
>
>mcache: a per-P cache of mspans with free space.
>
>mstats: allocation statistics.



![preview](Go内存管理，逃逸分析和垃圾回收机制/v2-1aa731ddd77b6cad73c8f68f864ea5ef_r.jpg)

我们着重讲一下不一样的地方。

## mheap

mheap并没有像tcmalloc那样将page组织成链表，而是组织成了树的结构。然后把Span分配到heapArena进行管理，它包含地址映射和span十分包含指针等位图。

![image-20211011145146420](Go内存管理，逃逸分析和垃圾回收机制.assets/image-20211011145146420.png)

mheap.spans ：用来存储 page 和 span 信息，比如一个 span 的起始地址是多少，有几个 page，已使用了多大等等。

 mheap.bitmap 存储着各个 span 中对象的标记信息，比如对象是否可回收等等。

 mheap.arena_start : 将要分配给应用程序使用的空间。

![img](https://pic4.zhimg.com/80/v2-ce68750f087fc37aa8f37b0c30ba38f7_720w.jpg)

1. object size：代码里简称size，指申请内存的对象大小。
2. size class：代码里简称class，它是size的级别，相当于把size归类到一定大小的区间段，比如size[1,8]属于size class 1，size(8,16]属于size class 2。
3. span class：指span的级别，但span class的大小与span的大小并没有正比关系。span class主要用来和size class做对应，1个size class对应2个span class，2个span class的span大小相同，只是功能不同，1个用来存放包含指针的对象，一个用来存放不包含指针的对象，不包含指针对象的Span就无需GC扫描了。
4. num of page：代码里简称npage，代表Page的数量，其实就是Span包含的页数，用来分配内存。

Go的内存分配并不像TCMalloc那样分成小中大对象，它只分了小对象和大对象，其中小对象分为Tiny对象和其它小对象。

- 小对象
  - Tiny：大小1Byte-16Byte
  - 其它：不包含指针

- 大对象：大于32KB

小对象是在mcache中分配的，大对象是直接从mheap中分配的。

大小转换这一小节，我们介绍了转换表，size class从1到67共67个，代码中_NumSizeClasses=68代表了实际使用的size class数量，即68个，从0到67，size class 0实际并未使用到。

上文提到1个size class对应2个span class：

```go
numSpanClasses = _NumSizeClasses << 1
```

numSpanClasses为span class的数量为136个，所以span class的下标是从0到135，所以上图中mcache标注了的span class是，span class 0到span class 135。每1个span class都指向1个span，也就是mcache最多有136个span。

## 小对象内存分配

### 为对象寻找span

> 官方给的定义如下
>
> 1. Round the size up to one of the small size classes and look in the corresponding mspan in this P's mcache. Scan the mspan's free bitmap to find a free slot. If there is a free slot, allocate it. This can all be done without acquiring a lock.
> 2.   If the mspan has no free slots, obtain a new mspan from the mcentral's list of mspans of the required sizeclass that have free space. Obtaining a whole span amortizes the cost of locking the mcentral.
>
> 3. If the mcentral's mspan list is empty, obtain a run of pages from the mheap to use for the mspan.
>
> 4. If the mheap is empty or has no page runs large enough, allocate a new group of pages (at least 1MB) from the operating system. Allocating a large run of pages amortizes the cost of talking to the operating system.

对应翻译，寻找span的流程如下：

1. 计算对象所需内存大小size
2. 根据size到size class的映射，计算出size_class
3. 根据size class和对象是否包含指针计算出span class
4. 获取该span class指向的span

![img](Go内存管理，逃逸分析和垃圾回收机制.assets/threadheap.gif)

以分配一个不包含指针的，大小为24Byte的对象为例，根据映射表：

```bash
// class  bytes/obj  bytes/span  objects  tail waste  max waste  min align
//     1          8        8192     1024           0     87.50%          8
//     2         16        8192      512           0     43.75%         16
//     3         24        8192      341           8     29.24%          8
//     4         32        8192      256           0     21.88%         32
```

对应的size class为3，它的对象大小范围是(16,32]Byte，24Byte刚好在此区间，所以此对象的size class为3。

从size_class到span_class计算公式如下：

```
type spanClass uint8
func makeSpanClass(sizeclass uint8, noscan bool) spanClass {
	return spanClass(sizeclass<<1) | spanClass(bool2int(noscan))
}
```

uint8(3<<1)|uint8(1) = 7，所以该对象需要的是span class 7指向的span。

### 从span分配对象空间

Span可以按照对象大小切成很多份，这些都可以从映射表上计算出来，以size class 3对应的span为例子，span的大小为8KB，每个对象实际所占的空间为32Byte，这个span 就被分为了256块，可以根据span的起始地址计算每个对象块的内存地址。

当span内所有的内存块都被占用时，mcache便会向mcentral申请一个span，mcache拿到span后继续分配对象。

![img](Go内存管理，逃逸分析和垃圾回收机制.assets/spanmap.gif)

### 从mcentral申请span

mcentral和mcache一样，都是0~135这136个span class级别，但每个级别都保存了2个span list，即2个span链表：

1. nonempty：这个链表里的span，所有span都至少	有1个空闲的对象空间。这些span是mcache释放span时加入到该链表的。
2. empty：这个链表里的span，所有的span都不确定里面是否有空闲的对象空间。当一个span交给mcache的时候，就会加入到empty链表。

这两个东西名称一直有点绕，建议直接把empty理解为没有对象空间就好了。

![img](Go内存管理，逃逸分析和垃圾回收机制.assets/v2-a0046c0be529175b1e6101698215f1f2_720w.jpg)

mcache向mcentral申请span时，mcentral会先从nonempty搜索满足条件的span，如果没有找到再从emtpy搜索满足条件的span，然后把找到的span交给mcache。

### mheap的span管理

mheap里保存了两棵二叉排序树，按span的page数量进行排序：

1. free：free中保存的span是空闲并且非垃圾回收的span。
2. scav：scav中保存的是空闲并且已经垃圾回收的span。

如果是垃圾回收导致的span释放，span会被加入到scav，否则加入到free，比如刚从OS申请的的内存也组成的Span。

mheap中还有arenas，由一组heapArena组成，每一个heapArena都包含了连续的pagesPerArena个span，这个主要是为mheap管理span和垃圾回收服务。mheap本身是一个全局变量，它里面的数据，也都是从OS直接申请来的内存，并不在mheap所管理的那部分内存以内。



- **mcentral向mheap申请span**

当mcentral向mcache提供span时，如果empty里也没有符合条件的span，mcentral会向mheap申请span。

此时，mcentral需要向mheap提供需要的内存页数和span class级别，然后它优先从free中搜索可用的span。如果没有找到，会从scav中搜索可用的span。如果还没有找到，它会向OS申请内存，再重新搜索2棵树，必然能找到span。如果找到的span比需要的span大，则把span进行分割成2个span，其中1个刚好是需求大小，把剩下的span再加入到free中去，然后设置需要的span的基本信息，然后交给mcentral。

- **mheap向OS申请内存**

当mheap没有足够的内存时，mheap会向OS申请内存，把申请的内存页保存为span，然后把span插入到free树。在32位系统中，mheap还会预留一部分空间，当mheap没有空间时，先从预留空间申请，如果预留空间内存也没有了，才向OS申请。

## 大对象的内存分配

大对象的分配比小对象省事多了，99%的流程与mcentral向mheap申请内存的相同，所以不重复介绍了。不同的一点在于mheap会记录一点大对象的统计信息，详情见mheap.alloc_m()。

### Arenas

堆由一组 arena 组成，它们在 64 位上为 64MB，在 32 位上为 4MB (heapArenaBytes)。 每个 arena 的起始地址也与 arena 大小对齐。

每个 arena 都有一个关联的 heapArena 对象，用于存储该 arena 的元数据：arena 中所有word的堆位图和 arena 中所有页面的 span 映射。 heapArena 对象本身是在堆外分配的。

由于 arena 是对齐的，因此地址空间可以被视为一系列 arena 帧。 arena 映射（mheap_.arenas）从 arena 帧号映射到 *heapArena，或者对于没有 Go 堆支持的地址空间部分映射为 nil。 竞技场地图被构造为一个由“L1”area映射和许多“L2”arena映射组成的两级数组； 然而，由于 arena 很大，因此在许多架构上，arena 映射由单个大型 L2 映射组成。

arena 映射覆盖了整个可能的地址空间，允许 Go 堆使用地址空间的任何部分。 分配器尝试保持 arena 连续，以便大span以及因此大对象）可以跨越 arenas。

## 总结

1. 使用缓存提高效率。在存储的整个体系中到处可见缓存的思想，Go内存分配和管理也使用了缓存，利用缓存一是减少了系统调用的次数，二是降低了锁的粒度、减少加锁的次数，从这2点提高了内存管理效率。
2. 以空间换时间，提高内存管理效率。空间换时间是一种常用的性能优化思想，这种思想其实非常普遍，比如Hash、Map、二叉排序树等数据结构的本质就是空间换时间，在数据库中也很常见，比如数据库索引、索引视图和数据缓存等，再如Redis等缓存数据库也是空间换时间的思想。

---

# Go 逃逸分析

问题来了，我们如何知道一个对象是在栈上还是堆上？https://golang.org/doc/faq#stack_or_heap

>### How do I know whether a variable is allocated on the heap or the stack?
>
>From a correctness standpoint, you don't need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.
>
>The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function's stack frame. However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.
>
>In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic *escape analysis* recognizes some cases when such variables will not live past the return from the function and can reside on the stack.

官方的解释是，你不需要知道对象是在栈上还是堆上。只要有其它对象引用它，该对象就存在。对象存储位置和语言语义无关。

如果可能，Go编译器会尽量在栈上分配。但是当编译器无法证明函数返回后该变量没有被引用，那么编译器就必须在支持GC的堆上分配空间以避免悬挂指针。另外如果局部变量很大，在栈上分配也比在堆上分配好。

目前，如果一个变量有地址，那么它适合在堆上分配。然而，编译器通过**逃逸分析**就可以知道该变量生命周期在函数return已经结束，因此可以在栈上。**<u>换言之，如果一个变量的生命周期完全已知，那么可以在栈上分配，否则在堆上分配。</u>**

- 逃逸分析是在编译器完成的，这是不同于jvm的运行时逃逸分析;
- 如果变量在函数外部没有引用，则优先放到栈中；
- 如果变量在函数外部存在引用，则必定放在堆中；

我们可以通过`go builf -gcflags '-m -m -l'`来进行逃逸分析，请务必记住这条语句，非常有用。

E.g. 1变量类型不确定

```
package main

import "fmt"

func main() {
    a := 666
    fmt.Println(a)
}
```

我们发现a被分配到堆上。

a逃逸是因为它被传入了`fmt.Println`的参数中，这个方法参数自己发生了逃逸。因为`fmt.Println`的函数参数为`interface`类型，编译期不能确定其参数的具体类型，所以将其分配于堆上。



E.g. 2 暴露给了外部指针

```
package main

func foo() *int {
    a := 666
    return &a
}

func main() {
    _ = foo()
}
```

我们发现a被分配到堆上。

这种情况直接满足我们上述中的原则：变量在函数外部存在引用。这个很好理解，因为当函数执行完毕，对应的栈帧就被销毁，但是引用已经被返回到函数之外。如果这时外部从引用地址取值，虽然地址还在，但是这块内存已经被释放回收了，这就是非法内存，问题可就大了。所以，很明显，这种情况必须分配到堆上。

E.g. 3 变量所占内存较大

```
func foo() {
    s := make([]int, 10000, 10000)
    for i := 0; i < len(s); i++ {
        s[i] = i
    }
}

func main() {
    foo()
}
```

可以看到，当我们创建了一个容量为10000的`int`类型的底层数组对象时，由于对象过大，它也会被分配到堆上。这里我们不禁要想一个问题，为啥大对象需要分配到堆上。这是因为不同平台上的系统栈最大限制不同。

以x86_64架构为例，它的系统栈大小最大可为8Mb。我们常说的goroutine初始大小为2kb，其实说的是用户栈，它的最小和最大可以在`runtime/stack.go`中找到，分别是2KB和1GB。

而堆则会大很多，从1.11之后，Go采用了稀疏的内存布局，在Linux的x86-64架构上运行时，整个堆区最大可以管理到256TB的内存。所以，为了不造成栈溢出和频繁的扩缩容，大的对象分配在堆上更加合理。那么，多大的对象会被分配到堆上呢。

通过测试，小菜刀发现该大小为64KB（这在Go内存分配中是属于大对象的范围：>32kb），即`s :=make([]int, n, n)`中，一旦`n`达到8192，就一定会逃逸。注意，网上有人通过`fmt.Println(unsafe.Sizeof(s))`得到`s`的大小为24字节，就误以为只需分配24个字节的内存，这是错误的，因为实际还有底层数组的内存需要分配。

E.g. 4 ：变量大小不确定

```go
package main

func foo() {
    n := 1
    s := make([]int, n)
    for i := 0; i < len(s); i++ {
        s[i] = i
    }
}

func main() {
    foo()
}
```



这次，我们在`make`方法中，没有直接指定大小，而是填入了变量`n`，这时Go逃逸分析也会将其分配到堆区去。可见，为了保证内存的绝对安全，Go的编译器可能会将一些变量不合时宜地分配到堆上，但是因为这些对象最终也会被垃圾收集器处理，所以也能接受。

## 总结

逃逸分析主要是判断栈上对象会不变逃逸到堆上。有如下几种情况

1. 变量类型不确定
2. 变量大小不确定
3. 变量分配的内存超过用户栈极限
4. 暴露给了外部指针

# Go垃圾回收

## 标记清除法（Go 1.3）

Go 1.3使用的是**标记清除法**，分下面四步进行：

1. 进行STW（Stop the World, 暂停程序业务逻辑），然后从main函数开始搜索内存空间
2. 标记可达内存区域
3. 标记结束清除未标记的内存占用
4. 结束STW，让程序继续运行，循环该过程直至main函数生命周期结束。

缺点：STW暂停用户逻辑对程序性能的影响非常大。

## 三色标记法（Go 1.5）

1. 将所有对象标记为白色
2. 从根节点出发，将第一次遍历到的节点标记为灰色放入到集合列表中
3. 遍历灰色对象，将遇到的白色对象标记为灰色，遇到的灰色对象标记为黑色。
4. 循环上一步，直到原灰色对象变为黑色

上述步骤结束后，标记为白色的对象就是不可达对象，需要进行垃圾回收。

上述算法存在明显的问题。如果一个白色对象新增了其它对象对它的引用，但颜色还是白色，就会被错误的回收。错误的原因在于：

1. 一个白色对象被黑色对象引用
2. 灰色对象与它之间的可达关系的白色对象遭到破坏



因此在此基础上提出了两种改进方法：

强三色不变式和弱三色不变式。

强三色不变式：不允许黑色对象引用白色对象。

弱三色不变式：黑色对象可引用白色对象，白色对象存在其他灰色对象对他的引用，或者他的链路上存在灰色对象。

为了实现这两种不变式的设计思想，提出了屏蔽机制。强三色对应于插入屏障，弱三色对应于删除屏障。

值得注意的是为了保证栈的运行效率，屏障只对堆上的内存对象使用，栈上的内存会在GC结束后启用STW重新扫描。由于栈上没有屏蔽机制，栈上被引用的白色对象仍会被回收。所以栈在GC迭代结束时（没有灰色节点），会对栈执行STW，重新进行扫描清除白色节点（STW时间为10-100ms）。

删除屏障：对象被删除时触发的机制。如果灰色对象引用的白色对象被删除时，白色对象会被标记为灰色。缺点是存在二次扫描影响程序效率。

## 三色标记+混合写屏障（Go 1.8）

基于插入写屏障和删除写屏障在结束时需要STW来重新扫描栈，带来性能瓶颈。混合写屏障分为以下四步：

1. GC开始时，将栈上的全部对象标记为黑色（不需要二次扫描，无需STW）；
2. GC期间，任何栈上创建的新对象均为黑色
3. 被删除引用的对象标记为灰色
4. 被添加引用的对象标记为灰色

这个改进之间使得GC 2s降低到2us。

![](Go内存管理，逃逸分析和垃圾回收机制.assets/vpza2yp4to.jpeg)

https://www.nowcoder.com/discuss/145338?type=2&order=3&pos=15&page=1

## GC触发的条件

GC触发有三种方式：

1. 辅助GC，在分配内存时，会判断当前的Heap内存分配量是否到达了触发一轮GC的阈值

2. 调用runtimee.GC()强制启动GC。
3. sysmon是运行时的守护进程，当到达一定周期`forcegcperiod`会自动触发GC。默认为2min。

### **GC调节参数**

Go垃圾回收不像Java垃圾回收那样，有很多参数可供调节，Go为了保证使用GC的简洁性，只提供了一个参数`GOGC`。

`GOGC`代表了占用中的内存增长比率，达到该比率时应当触发1次GC，该参数可以通过环境变量设置。

该参数取值范围为0~100，默认值是100，单位是百分比。

假如当前heap占用内存为4MB，`GOGC = 75`，

```javascript
4 * (1+75%) = 7MB
```

等heap占用内存大小达到7MB时会触发1轮GC。

`GOGC`还有2个特殊值：

1. `"off"` : 代表关闭GC
2. `0` : 代表持续进行垃圾回收，只用于调试



## 总结：

Go 1.3 引用了标记清除法，通过标记清理未被引用的部分，缺点是要STW，暂停用户逻辑；

Go 1.5采用的是三色标记方法+写屏障，包括插入屏障和删除屏障；

Go 1.8之后采用混合写屏障，对于栈内对象使用。

```
// Copyright 2014 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Memory allocator.
//
// This was originally based on tcmalloc, but has diverged quite a bit.
// http://goog-perftools.sourceforge.net/doc/tcmalloc.html

// The main allocator works in runs of pages.
// Small allocation sizes (up to and including 32 kB) are
// rounded to one of about 70 size classes, each of which
// has its own free set of objects of exactly that size.
// Any free page of memory can be split into a set of objects
// of one size class, which are then managed using a free bitmap.
//
// The allocator's data structures are:
//
//	fixalloc: a free-list allocator for fixed-size off-heap objects,
//		used to manage storage used by the allocator.
//	mheap: the malloc heap, managed at page (8192-byte) granularity.
//	mspan: a run of in-use pages managed by the mheap.
//	mcentral: collects all spans of a given size class.
//	mcache: a per-P cache of mspans with free space.
//	mstats: allocation statistics.
//
// Allocating a small object proceeds up a hierarchy of caches:
//
//	1. Round the size up to one of the small size classes
//	   and look in the corresponding mspan in this P's mcache.
//	   Scan the mspan's free bitmap to find a free slot.
//	   If there is a free slot, allocate it.
//	   This can all be done without acquiring a lock.
//
//	2. If the mspan has no free slots, obtain a new mspan
//	   from the mcentral's list of mspans of the required size
//	   class that have free space.
//	   Obtaining a whole span amortizes the cost of locking
//	   the mcentral.
//
//	3. If the mcentral's mspan list is empty, obtain a run
//	   of pages from the mheap to use for the mspan.
//
//	4. If the mheap is empty or has no page runs large enough,
//	   allocate a new group of pages (at least 1MB) from the
//	   operating system. Allocating a large run of pages
//	   amortizes the cost of talking to the operating system.
//
// Sweeping an mspan and freeing objects on it proceeds up a similar
// hierarchy:
//
//	1. If the mspan is being swept in response to allocation, it
//	   is returned to the mcache to satisfy the allocation.
//
//	2. Otherwise, if the mspan still has allocated objects in it,
//	   it is placed on the mcentral free list for the mspan's size
//	   class.
//
//	3. Otherwise, if all objects in the mspan are free, the mspan's
//	   pages are returned to the mheap and the mspan is now dead.
//
// Allocating and freeing a large object uses the mheap
// directly, bypassing the mcache and mcentral.
//
// If mspan.needzero is false, then free object slots in the mspan are
// already zeroed. Otherwise if needzero is true, objects are zeroed as
// they are allocated. There are various benefits to delaying zeroing
// this way:
//
//	1. Stack frame allocation can avoid zeroing altogether.
//
//	2. It exhibits better temporal locality, since the program is
//	   probably about to write to the memory.
//
//	3. We don't zero pages that never get reused.

// Virtual memory layout
//
// The heap consists of a set of arenas, which are 64MB on 64-bit and
// 4MB on 32-bit (heapArenaBytes). Each arena's start address is also
// aligned to the arena size.
//
// Each arena has an associated heapArena object that stores the
// metadata for that arena: the heap bitmap for all words in the arena
// and the span map for all pages in the arena. heapArena objects are
// themselves allocated off-heap.
//
// Since arenas are aligned, the address space can be viewed as a
// series of arena frames. The arena map (mheap_.arenas) maps from
// arena frame number to *heapArena, or nil for parts of the address
// space not backed by the Go heap. The arena map is structured as a
// two-level array consisting of a "L1" arena map and many "L2" arena
// maps; however, since arenas are large, on many architectures, the
// arena map consists of a single, large L2 map.
//
// The arena map covers the entire possible address space, allowing
// the Go heap to use any part of the address space. The allocator
// attempts to keep arenas contiguous so that large spans (and hence
// large objects) can cross arenas.

```

