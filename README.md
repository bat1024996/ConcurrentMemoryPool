# ConcurrentMemoryPool
| **Part Ⅰ**            | **Part Ⅱ**            | **Part Ⅲ**            | **Part Ⅳ**            | **Part Ⅴ**            | Part VI                            |
| --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | ---------------------------------- |
| [项目介绍](#项目介绍) | [项目特点](#项目特点) | [开发环境](#开发环境) | [并发模型](#并发模型) | [模块讲解](#模块讲解) | [与malloc对比](# 项目与malloc对比) |

## 

# 项目介绍

本项目实现的是一个高并发的**内存池**，它的原型是Google的一个开源项目**tcmalloc**，tcmalloc全称Thread-Caching Malloc，即线程缓存的malloc，实现了高效的多线程内存管理，用于替换系统的内存分配相关函数malloc和free。

tcmalloc的知名度也是非常高的，不少公司都在用它，比如Go语言就直接用它做了自己的内存分配器。

该项目就是把tcmalloc中最核心的框架简化后拿出来，模拟实现出一个mini版的高并发内存池，目的就是学习tcmalloc的精华。



# 项目特点

高并发内存池主要解决的就是**内存分配效率**的问题，它能够避免让程序频繁的向系统申请和释放内存。其次，内存池作为系统的内存分配器，还需要尝试解决**内存碎片**的问题。

内存碎片分为内部碎片和外部碎片：

1. 外部碎片是一些空闲的小块内存区域，由于这些内存空间不连续，以至于合计的内存足够，但是不能满足一些内存分配申请需求。2.

2. 内部碎片是由于一些对齐的需求，导致分配出去的空间中一些内存无法被利用。

   

   注意： 内存池尝试解决的是外部碎片的问题，同时也尽可能的减少内部碎片的产生。

应用技术： C**/C++、数据结构（链表、哈希桶）、操作系统内存管理、单例模式、多线程、互斥锁，池化技术，慢增长反馈算法**



# 开发环境

- 操作系统：`Windows`
- 编译器：`Visual Studio 2013`
- 版本控制：`git`





# 并发模型

![](E:\projects uplode to GitHub\ConcurrentMemoryPool\项目模块讲解\pics\1.png)

高并发内存池主要由以下三个部分构成：

1. thread cache： 线程缓存是每个线程独有的，用于小于等于256KB的内存分配，每个线程独享一个thread cache。
2. central cache： 中心缓存是所有线程所共享的，当thread cache需要内存时会按需从central cache中获取内存，而当thread cache中的内存满足一定条件时，central cache也会在合适的时机对其进行回收。
3. page cache： 页缓存中存储的内存是以页为单位进行存储及分配的，当central cache需要内存时，page cache会分配出一定数量的页分配给central cache，而当central cache中的内存满足一定条件时，page cache也会在合适的时机对其进行回收，并将回收的内存尽可能的进行合并，组成更大的连续内存块，缓解内存碎片的问题。
   

### threadcache

![](E:\projects uplode to GitHub\ConcurrentMemoryPool\项目模块讲解\pics\2.png)

**thread cache是哈希桶结构，每个桶是一个按桶位置映射大小的内存块对象的自由链表。每个线程都会**
**有一个thread cache对象，这样每个线程在这里获取对象和释放对象时是无锁的。**



### centralcache

![](E:\projects uplode to GitHub\ConcurrentMemoryPool\项目模块讲解\pics\3.png)

**central cache也是一个哈希桶结构，他的哈希桶的映射关系跟thread cache是一样的。不同的是他的每**
**个哈希桶位置挂是SpanList链表结构，不过每个映射桶下面的span中的大内存块被按映射关系切成了一**
**个个小内存块对象挂在span的自由链表中。**



### pagecache

![](E:\projects uplode to GitHub\ConcurrentMemoryPool\项目模块讲解\pics\4.png)



**页缓存是在central cache缓存上面的一层缓存，存储的内存是以页为单位存储及分**
**配的，central cache没有内存对象时，从page cache分配出一定数量的page，并切割成定长大小**
**的小块内存，分配给central cache**。当一个span的几个跨度页的对象都回收以后，page cache
会回收central cache满足条件的span对象，并且合并相邻的页，组成更大的页，缓解内存碎片
的问题。



# 模块讲解

 [定长内存池的实现](./项目模块讲解/定长内存池的实现.md)

 [高并发内存池整体框架设计.md](项目模块讲解\高并发内存池整体框架设计.md) 

 [Threadcache模块.md](项目模块讲解\Threadcache模块.md) 

 [Centralcache模块.md](项目模块讲解\Centralcache模块.md) 

 [Pagecache模块.md](项目模块讲解\Pagecache模块.md) 

 [分析性能瓶颈并用基数树优化.md](项目模块讲解\分析性能瓶颈并用基数树优化.md) 



# 项目与malloc对比

**malloc底层是采用边界标记法将内存划分成很多块，从而对内存的分配与回收进行管理**。简单来说，**malloc分配内存时会先获取分配区的锁**，然后根据申请内存的大小一级一级的去获取内存空间，最后返回。

所以在高并发的场景下，malloc在申请内存时需要加锁，以避免多个线程同时修改内存分配信息，这会导致性能下降。而内存池可以通过维护自由链表来分配内存，避免了加锁的开销。

总结出本项目效率相对较高的3点原因：

- 1.第一级thread cache通过tls技术实现了无锁访问。
- 2.第二级central cache加的是桶锁，可以更好的实现多线程的并行。
- 3.第三级page cache通过基数树优化，有效减少了锁的竞争。





