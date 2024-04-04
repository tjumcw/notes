# GOLANG内存机制

Go语言内置运行时（就是runtime），抛弃了传统的内存分配方式，改为自主管理。这样可以自主地实现更好的内存使用模式，比如内存池、预分配等等。这样，不会每次内存分配都需要进行系统调用。



### 一、设计思想

- 内存分配算法采用Google的`TCMalloc算法`，每个线程都会自行维护一个独立的内存池，进行内存分配时优先从该内存池中分配，当内存池不足时才会向加锁向全局内存池申请，减少系统调用并且避免不同线程对全局内存池的锁竞争

- 把内存切分的非常的细小，分为多级管理，以降低锁的粒度
- 回收对象内存时，并没有将其真正释放掉，只是放回预先分配的大块内存中，以便复用。只有内存闲置过多的时候，才会尝试归还部分内存给操作系统，降低整体开销



### ==二、分配组件==

Go的内存管理组件主要有：`mspan`、`mcache`、`mcentral`和`mheap`

#### 1、内存管理单元：mspan

- MSPan是内存分配器的基础工具组件，==简称span==，是用来管理一组组page对象。

- 结构体中包含 `next` 和 `prev` 两个字段，分别指向了前后的mspan，形成双向链表。

- 每个`span` 都管理 `npages` 个大小为 8KB 的页，一个span 是由多个page组成的。
  - 页不是操作系统中的内存页，它们是操作系统内存页的整数倍。

- span将这一个个连续的page给管理起来，切分成一个一个连续的内存才能在链表freelist上（小对象）
- Go有68种不同大小的spanClass，用于对象的分配（[0, 8, 16...32768]）
  - 0表示大对象，会去堆获取内存，小于等于32k的对象就是小对象，其它都是大对象。

```go
type mspan struct {
    next *mspan // 后指针
    prev *mspan // 前指针
    startAddr uintptr // 管理页的起始地址，指向page
    npages    uintptr // 页数
    spanclass   spanClass // 规格,表示要存在freelist中的对象规格，按其对page切分
    freelist *MLink //list of free objects
    ...
}
type spanClass uint8
```

![image-20220814120152874](C:\Users\mcw\AppData\Roaming\Typora\typora-user-images\image-20220814120152874.png)



#### 2、线程缓存：mcache

- 线程私有的，每个线程都有一个cache，用来存放小对象。
- 由于每个线程都有cache，所以获取空闲内存是不用加锁的。
- cache层的主要目的就是提高小内存的频繁分配释放速度。
- 实际申请一般小于32k，这样内存分配走本地cache，不用向操作系统申请非常高效。

- cache在源码中结构为MCache，其结构体定义如下：

```go
type mcache struct {
    alloc [numSpanClasses]*mspan
}

_NumSizeClasses = 68
numSpanClasses = _NumSizeClasses << 1
```

>注意事项：
>
>- `mcache`用`Span Classes`作为索引管理多个用于分配的`mspan`，它包含所有规格的`mspan`。它是`_NumSizeClasses`的2倍，也就是`68*2=136`。
>- 其中*2是将spanClass分成了有指针和没有指针两种,方便与垃圾回收。对于每种规格，有2个mspan，一个mspan不包含指针，另一个mspan则包含指针。对于无指针对象的`mspan`在进行垃圾回收的时候无需进一步扫描它是否引用了其他活跃的对象。
>- 简单说来就是有一个数组，为136大小，存68种按spanClass分割内存大小的span（有无指针各一种）
>- `mcache`在初始化的时候是没有任何`mspan`资源的，在使用过程中会动态地从`mcentral`申请，之后会缓存下来。当对象小于等于32KB大小时，使用`mcache`的相应规格的`mspan`进行分配。



#### 3、中心缓存：mcentral

- mcentral管理全局的mspan供所有线程使用，每个 mcentral 结构都维护在**mheap**结构内
- 所有线程共享的组件，不是独占的，因此==需要加锁操作==，它其实也是一个缓存。

```go
type mcentral struct {
    spanclass spanClass // 指当前规格大小

    partial [2]spanSet // 有空闲object的mspan列表
    full    [2]spanSet // 没有空闲object的mspan列表
}
```

>细节：
>
>- 每个mcentral管理一种spanClass的mspan，并将有空闲空间和没有空闲空间的mspan分开管理。
>- partial和 full`的数据类型为`spanSet，表示 `mspans`集，可以通过pop、push来获得mspans
>
>`mcache`从`mcentral`获取和归还`mspan`的流程：
>
>- 获取：加锁，从`partial`链表找到一个可用的`mspan`；并将其从`partial`链表删除；将取出的`mspan`加入到`full`链表；将`mspan`返回给工作线程，解锁。
>- 归还： 加锁，将`mspan`从`full`链表删除；将`mspan`加入到`partial`链表，解锁。



#### 4、页堆：mheap

- mheap管理Go的所有动态分配内存，可以认为是Go程序持有的整个堆空间，全局唯一

- 所有`mcentral`的集合则是存放于`mheap`中的。

>申请内存时：
>
>- 当申请内存时，依次经过 `mcache` 和 `mcentral` 都没有可用合适规格的大小内存，这时候会向 `mheap` 申请一块内存。
>- 然后按指定规格划分为一些列表，并将其添加到相同规格大小的 `mcentral` 的 `非空闲列表` 后面
>- 若是堆内存种依旧无free，则通过系统调用向操作系统申请内存



### 三、分配对象

- 微对象 (0, 16B)：先使用线程缓存上的微型分配器，再依次尝试线程缓存、中心缓存、堆分配内存；

- 小对象 [16B, 32KB]：依次尝试线程缓存、中心缓存、堆分配内存；

- 大对象 (32KB, +∞)：直接尝试堆分配内存；



### 四、分配流程

- 首先通过计算使用的大小规格
- 然后使用`mcache`中对应大小规格的块分配。
- 如果`mcentral`中没有可用的块，则向`mheap`申请，并根据算法找到最合适的`mspan`。
- 如果申请到的`mspan` 超出申请大小，将会根据需求进行切分，以返回用户所需的页数。剩余的页构成一个新的 mspan 放回 mheap 的空闲列表。
- 如果 mheap 中没有可用 span，则向操作系统申请一系列新的页（最小 1MB）

![image-20220814131527874](C:\Users\mcw\AppData\Roaming\Typora\typora-user-images\image-20220814131527874.png)





> 详细流程：
>
> 1. 首先判定 对象是 大对象 还是 普通对象还是 小对象
> 2. 如果是 小对象
>    1. 从 mcache 的alloc 找到对应 classsize 的 mspan
>    2. 如果当前mspan有足够的空间，分配并修改mspan的相关属性（nextFreeFast函数中实现）
>    3. 如果当前mspan没有足够的空间，从 mcentral重新获取一块 对应 classsize的 mspan，替换原先的mspan，然后 分配并修改mspan的相关属性
>    4. 如果mcentral没有足够的对应的classsize的span，则去向mheap申请
>    5. 如果 对应classsize的span没有了，则找一个相近的classsize的span，切割并分配
>    6. 如果 找不到相近的classsize的span，则去向系统申请，并补充到mheap中
> 3. 如果是普通对象，逻辑大致同小对象的 内存分配
>    1. 首先查表，以确定 需要分配内存的对象的 sizeclass，并找到 对应 classsize的 mspan
>    2. 如果当前mspan有足够的空间，分配并修改mspan的相关属性（nextFreeFast函数中实现）
>    3. 如果当前mspan没有足够的空间，从 mcentral重新获取一块 对应 classsize的 mspan，替换原先的mspan，然后 分配并修改mspan的相关属性
>    4. 如果mcentral没有足够的对应的classsize的span，则去向mheap申请
>    5. 如果 对应classsize的span没有了，则找一个相近的classsize的span，切割并分配
>    6. 如果 找不到相近的classsize的span，则去向系统申请，并补充到mheap中
> 4. 如果是大对象，直接从mheap进行分配
>    1. 如果 对应classsize的span没有了，则找一个相近的classsize的span，切割并分配
>    2. 如果 找不到相近的classsize的span，则去向系统申请，并补充到mheap中



### 五、总结

- **多层次Cache来减少分配的冲突, 加快分配. 从无锁到粒度较低的锁, 再到全局一个锁, 或系统调用.**

- 小对象的内存分配是通过一级一级的缓存来实现的，就是为了提升内存分配释放的速度以及避免内存碎片。

- 最后分配的内存都是通过构建成span的形式来分配的