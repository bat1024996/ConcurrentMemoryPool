# Centralcache模块

当线程申请某一大小的内存时，如果thread cache中对应的自由链表不为空，那么直接取出一个内存块进行返回即可，但如果此时该自由链表为空，那么这时thread cache就需要向central cache申请内存了。

central cache的结构与thread cache是一样的，它们都是哈希桶的结构，并且它们遵循的对齐映射规则都是一样的。这样做的好处就是，当thread cache的某个桶中没有内存了，就可以直接到central cache中对应的哈希桶里去取内存就行了。



#### centralcache与threadcache的不同

 central cache与thread cache有两个明显不同的地方，首先，thread cache是每个线程独享的，而central cache是所有线程共享的，因为每个线程的thread cache没有内存了都会去找central cache，因此在访问central cache时是需要加锁的。

  但central cache在加锁时并不是将整个central cache全部锁上了，central cache在加锁时用的是桶锁，也就是说每个桶都有一个锁。此时只有当多个线程同时访问central cache的同一个桶时才会存在锁竞争，如果是多个线程同时访问central cache的不同桶就不会存在锁竞争。

  central cache与thread cache的第二个不同之处就是，thread cache的每个桶中挂的是一个个切好的内存块，而central cache的每个桶中挂的是一个个的span。


![3.png](https://img1.imgtp.com/2023/08/28/0WJtbYVW.png)



每个span管理的都是一个以页为单位的大块内存，每个桶里面的若干span是按照双链表的形式链接起来的，并且每个span里面还有一个自由链表，这个自由链表里面挂的就是一个个切好了的内存块，根据其所在的哈希桶这些内存块被切成了对应的大小。



#### span结构

```cpp
//管理以页为单位的大块内存
struct Span
{
	PAGE_ID _pageId = 0;        //大块内存起始页的页号
	size_t _n = 0;              //页的数量

	Span* _next = nullptr;      //双链表结构
	Span* _prev = nullptr;

	size_t _objSize = 0;        //切好的小对象的大小
	size_t _useCount = 0;       //切好的小块内存，被分配给thread cache的计数
	void* _freeList = nullptr;  //切好的小块内存的自由链表

	bool _isUse = false;        //是否在被使用
};

```



对于span管理的以页为单位的大块内存，我们需要知道这块内存具体在哪一个位置，便于之后page cache进行前后页的合并，因此span结构当中会记录所管理大块内存起始页的页号。

至于每一个span管理的到底是多少个页，这并不是固定的，需要根据多方面的因素来控制，因此span结构当中有一个_n成员，该成员就代表着该span管理的页的数量。

此外，每个span管理的大块内存，都会被切成相应大小的内存块挂到当前span的自由链表中，比如8Byte哈希桶中的span，会被切成一个个8Byte大小的内存块挂到当前span的自由链表中，因此span结构中需要存储切好的小块内存的自由链表。

span结构当中的_useCount成员记录的就是，当前span中切好的小块内存，被分配给thread cache的计数，当某个span的_useCount计数变为0时，代表当前span切出去的内存块对象全部还回来了，此时central cache就可以将这个span再还给page cache。

每个桶当中的span是以双链表的形式组织起来的，当我们需要将某个span归还给page cache时，就可以很方便的将该span从双链表结构中移出。如果用单链表结构的话就比较麻烦了，因为单链表在删除时，需要知道当前结点的前一个结点。



### Centralcache 结构

```cpp
class CentralCache
{
public:
	//提供一个全局访问点
	static CentralCache* GetInstance()
	{
		return &_sInst;
	}

	//从central cache获取一定数量的对象给thread cache
	size_t FetchRangeObj(void*& start, void*& end, size_t n, size_t size);

	//获取一个非空的span
	Span* GetOneSpan(SpanList& spanList, size_t size);

	//将一定数量的对象还给对应的span
	void ReleaseListToSpans(void* start, size_t size);
private:
	SpanList _spanLists[NFREELISTS];

private:
	CentralCache() //构造函数私有
	{}
	CentralCache(const CentralCache&) = delete; //防拷贝

	static CentralCache _sInst;
};
```



### 慢反馈调节算法

当thread cache向central cache申请内存时，如果申请的是较小的对象，那么可以多给一点，但如果申请的是较大的对象，就可以少给一点。

具体在Threadcache模块讲解。





### 分配内存给threadcache

```cpp
//从central cache获取一定数量的对象给thread cache
size_t CentralCache::FetchRangeObj(void*& start, void*& end, size_t n, size_t size)
{
	size_t index = SizeClass::Index(size);
	_spanLists[index]._mtx.lock(); //加锁

	//在对应哈希桶中获取一个非空的span
	Span* span = GetOneSpan(_spanLists[index], size);
	assert(span); //span不为空
	assert(span->_freeList); //span当中的自由链表也不为空

	//从span中获取n个对象
	//如果不够n个，有多少拿多少
	start = span->_freeList;
	end = span->_freeList;
	size_t actualNum = 1;
	while (NextObj(end)&&n - 1)
	{
		end = NextObj(end);
		actualNum++;
		n--;
	}
	span->_freeList = NextObj(end); //取完后剩下的对象继续放到自由链表
	NextObj(end) = nullptr; //取出的一段链表的表尾置空
	span->_useCount += actualNum; //更新被分配给thread cache的计数

	_spanLists[index]._mtx.unlock(); //解锁
	return actualNum;
}
```



由于central cache是所有线程共享的，所以我们在访问central cache中的哈希桶时，需要先给对应的哈希桶加上桶锁，在获取到对象后再将桶锁解掉。

在向central cache获取对象时，先是在central cache对应的哈希桶中获取到一个非空的span，然后从这个span的自由链表中取出n个对象即可，但可能这个非空的span的自由链表当中对象的个数不足n个，这时该自由链表当中有多少个对象就给多少就行了。

也就是说，thread cache实际从central cache获得的对象的个数可能与我们传入的n值是不一样的，因此我们需要统计本次申请过程中，实际thread cache获取到的对象个数，然后根据该值及时更新这个span中的小对象被分配给thread cache的计数。



### 向pagecache获取对象

```
//获取一个非空的span
Span* CentralCache::GetOneSpan(SpanList& spanList, size_t size)
{
	//1、先在spanList中寻找非空的span
	Span* it = spanList.Begin();
	while (it != spanList.End())
	{
		if (it->_freeList != nullptr)
		{
			return it;
		}
		else
		{
			it = it->_next;
		}
	}
	//2、spanList中没有非空的span，只能向page cache申请
	//先把central cache的桶锁解掉，这样如果其他对象释放内存对象回来，不会阻塞
	spanList._mtx.unlock();
	PageCache::GetInstance()->_pageMtx.lock();
	Span* span = PageCache::GetInstance()->NewSpan(SizeClass::NumMovePage(size));
	span->_isUse = true;
	span->_objSize = size; //该span将会被切成一个个size大小的对象
	PageCache::GetInstance()->_pageMtx.unlock();
	//获取到span后不需要立刻重新加上central cache的桶锁

	//计算span的大块内存的起始地址和大块内存的大小（字节数）
	char* start = (char*)(span->_pageId << PAGE_SHIFT);
	size_t bytes = span->_n << PAGE_SHIFT;

	//把大块内存切成size大小的对象链接起来
	char* end = start + bytes;
	//先切一块下来去做尾，方便尾插
	span->_freeList = start;
	start += size;
	void* tail = span->_freeList;
	//尾插
	while (start < end)
	{
		NextObj(tail) = start;
		tail = NextObj(tail);
		start += size;
	}
	NextObj(tail) = nullptr; //尾的指向置空
	
	//将切好的span头插到spanList
	spanList._mtx.lock(); //span切分完毕后，需要挂到桶里时再重新加桶锁
	spanList.PushFront(span);

	return span;
}
```



#### centralcache 回收内存

这时当thread cache还对象给central cache时，就可以依次遍历这些对象，将这些对象插入到其对应span的自由链表当中，并且及时更新该span的_usseCount计数即可。

在thread cache还对象给central cache的过程中，如果central cache中某个span的_useCount减到0时，说明这个span分配出去的对象全部都还回来了，那么此时就可以将这个span再进一步还给page cache。


```cpp
//将一定数量的对象还给对应的span
void CentralCache::ReleaseListToSpans(void* start, size_t size)
{
	size_t index = SizeClass::Index(size);
	_spanLists[index]._mtx.lock(); //加锁
	while (start)
	{
		void* next = NextObj(start); //记录下一个
		Span* span = PageCache::GetInstance()->MapObjectToSpan(start);
		//将对象头插到span的自由链表
		NextObj(start) = span->_freeList;
		span->_freeList = start;

		span->_useCount--; //更新被分配给thread cache的计数
		if (span->_useCount == 0) //说明这个span分配出去的对象全部都回来了
		{
			//此时这个span就可以再回收给page cache，page cache可以再尝试去做前后页的合并
			_spanLists[index].Erase(span);
			span->_freeList = nullptr; //自由链表置空
			span->_next = nullptr;
			span->_prev = nullptr;

			//释放span给page cache时，使用page cache的锁就可以了，这时把桶锁解掉
			_spanLists[index]._mtx.unlock(); //解桶锁
			PageCache::GetInstance()->_pageMtx.lock(); //加大锁
			PageCache::GetInstance()->ReleaseSpanToPageCache(span);
			PageCache::GetInstance()->_pageMtx.unlock(); //解大锁
			_spanLists[index]._mtx.lock(); //加桶锁
		}

		start = next;
	}

	_spanLists[index]._mtx.unlock(); //解锁
}
```



如果要把某个span还给page cache，我们需要先将这个span从central cache对应的双链表中移除，然后再将该span的自由链表置空，因为page cache中的span是不需要切分成一个个的小对象的，以及该span的前后指针也都应该置空，因为之后要将其插入到page cache对应的双链表中。但span当中记录的起始页号以及它管理的页数是不能清除的，否则对应内存块就找不到了。

并且在central cache还span给page cache时也存在锁的问题，此时需要先将central cache中对应的桶锁解掉，然后再加上page cache的大锁之后才能进入page cache进行相关操作，当处理完毕回到central cache时，除了将page cache的大锁解掉，还需要立刻获得central cache对应的桶锁，然后将还未还完对象继续还给central cache中对应的span。

