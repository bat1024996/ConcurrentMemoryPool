# Pagecache模块

![](E:\projects uplode to GitHub\ConcurrentMemoryPool\项目模块讲解\pics\4.png)

central cache的映射规则与thread cache保持一致，而page cache的映射规则与它们都不相同。page cache的哈希桶映射规则采用的是直接定址法，比如1号桶挂的都是1页的span，2号桶挂的都是2页的span，以此类推。

central cache每个桶中的span被切成了一个个对应大小的对象，以供thread cache申请。而page cache当中的span是没有被进一步切小的，因为page cache服务的是central cache，当central cache没有span时，向page cache申请的是某一固定页数的span，而如何切分申请到的这个span就应该由central cache自己来决定。


#### 整体设计

当每个线程的thread cache没有内存时都会向central cache申请，此时多个线程的thread cache如果访问的不是central cache的同一个桶，那么这些线程是可以同时进行访问的。这时central cache的多个桶就可能同时向page cache申请内存的，所以page cache也是存在线程安全问题的，因此在访问page cache时也必须要加锁。

但是在page cache这里我们不能使用桶锁，因为当central cache向page cache申请内存时，page cache可能会将其他桶当中大页的span切小后再给central cache。此外，当central cache将某个span归还给page cache时，page cache也会尝试将该span与其他桶当中的span进行合并。


```cpp

//单例模式
class PageCache
{
public:
	//提供一个全局访问点
	static PageCache* GetInstance()
	{
		return &_sInst;
	}
	//获取一个k页的span
	Span* NewSpan(size_t k);

	//获取从对象到span的映射
	Span* MapObjectToSpan(void* obj);

	//释放空闲的span回到PageCache，并合并相邻的span
	void ReleaseSpanToPageCache(Span* span);

	std::mutex _pageMtx; //大锁
private:
	SpanList _spanLists[NPAGES];
	//std::unordered_map<PAGE_ID, Span*> _idSpanMap;
	TCMalloc_PageMap1<32 - PAGE_SHIFT> _idSpanMap;

	ObjectPool<Span> _spanPool;
	
	PageCache() //构造函数私有
	{}
	PageCache(const PageCache&) = delete; //防拷贝

	static PageCache _sInst;
};

```



### 在页缓存中获取一个span的过程

1. 如果central cache要获取一个n页的span，那我们就可以在page cache的第n号桶中取出一个span返回给central cache即可，但如果第n号桶中没有span了，这时我们并不是直接转而向堆申请一个n页的span，而是要继续在后面的桶当中寻找span。
2. 直接向堆申请以页为单位的内存时，我们应该尽量申请大块一点的内存块，因为此时申请到的内存是连续的，当线程需要内存时我们可以将其切小后分配给线程，而当线程将内存释放后我们又可以将其合并成大块的连续内存。如果我们向堆申请内存时是小块小块的申请的，那么我们申请到的内存就不一定是连续的了。
3. 当第n号桶中没有span时，我们可以继续找第n+1号桶，因为我们可以将n+1页的span切分成一个n页的span和一个1页的span，这时我们就可以将n页的span返回，而将切分后1页的span挂到1号桶中。但如果后面的桶当中都没有span，这时我们就只能向堆申请一个128页的内存块，并将其用一个span结构管理起来，然后将128页的span切分成n页的span和128-n页的span，其中n页的span返回给central cache，而128-n页的span就挂到第128-n号桶中。
4. 我们每次向堆申请的都是128页大小的内存块，central cache要的这些span实际都是由128页的span切分出来的。
   



### 获取一个k页的span

```cpp
//获取一个k页的span
Span* PageCache::NewSpan(size_t k)
{
	//assert(k > 0 && k < NPAGES);
	assert(k > 0);
	if (k > NPAGES - 1) //大于128页直接找堆申请
	{
		void* ptr = SystemAlloc(k);
		//Span* span = new Span;
		Span* span = _spanPool.New();

		span->_pageId = (PAGE_ID)ptr >> PAGE_SHIFT;
		span->_n = k;
		//建立页号与span之间的映射
		//_idSpanMap[span->_pageId] = span;
		_idSpanMap.set(span->_pageId, span);

		return span;
	}
	//先检查第k个桶里面有没有span
	if (!_spanLists[k].Empty())
	{
		Span* kSpan = _spanLists[k].PopFront();

		//建立页号与span的映射，方便central cache回收小块内存时查找对应的span
		for (PAGE_ID i = 0; i < kSpan->_n; i++)
		{
			//_idSpanMap[kSpan->_pageId + i] = kSpan;
			_idSpanMap.set(kSpan->_pageId + i, kSpan);
		}

		return kSpan;
	}
	//检查一下后面的桶里面有没有span，如果有可以将其进行切分
	for (size_t i = k + 1; i < NPAGES; i++)
	{
		if (!_spanLists[i].Empty())
		{
			Span* nSpan = _spanLists[i].PopFront();
			//Span* kSpan = new Span;
			Span* kSpan = _spanPool.New();

			//在nSpan的头部切k页下来
			kSpan->_pageId = nSpan->_pageId;
			kSpan->_n = k;

			nSpan->_pageId += k;
			nSpan->_n -= k;
			//将剩下的挂到对应映射的位置
			_spanLists[nSpan->_n].PushFront(nSpan);
			//存储nSpan的首尾页号与nSpan之间的映射，方便page cache合并span时进行前后页的查找
			//_idSpanMap[nSpan->_pageId] = nSpan;
			//_idSpanMap[nSpan->_pageId + nSpan->_n - 1] = nSpan;
			_idSpanMap.set(nSpan->_pageId, nSpan);
			_idSpanMap.set(nSpan->_pageId + nSpan->_n - 1, nSpan);

			//建立页号与span的映射，方便central cache回收小块内存时查找对应的span
			for (PAGE_ID i = 0; i < kSpan->_n; i++)
			{
				//_idSpanMap[kSpan->_pageId + i] = kSpan;
				_idSpanMap.set(kSpan->_pageId + i, kSpan);
			}

			//cout << "dargon" << endl; //for test
			return kSpan;
		}
	}
	//走到这里说明后面没有大页的span了，这时就向堆申请一个128页的span
	//Span* bigSpan = new Span;
	Span* bigSpan = _spanPool.New();

	void* ptr = SystemAlloc(NPAGES - 1);
	bigSpan->_pageId = (PAGE_ID)ptr >> PAGE_SHIFT;
	bigSpan->_n = NPAGES - 1;

	_spanLists[bigSpan->_n].PushFront(bigSpan);

	//尽量避免代码重复，递归调用自己
	return NewSpan(k);
}
```

如果page cache的第k号桶中没有span，我们就应该继续找后面的桶，只要后面任意一个桶中有一个n页span，我们就可以将其切分成一个k页的span和一个n-k页的span，然后将切出来k页的span返回给central cache，再将n-k页的span挂到page cache的第n-k号桶即可。

但如果后面的桶中也都没有span，此时我们就需要向堆申请一个128页的span了，在向堆申请内存时，直接调用我们封装的SystemAlloc函数即可。

需要注意的是，向堆申请内存后得到的是这块内存的起始地址，此时我们需要将该地址转换为页号。由于我们向堆申请内存时都是按页进行申请的，因此我们直接将该地址除以一页的大小即可得到对应的页号。





### pagecache回收内存

如果central cache中有某个span的_useCount减到0了，那么central cache就需要将这个span还给page cache了。

这个过程看似是非常简单的，page cache只需将还回来的span挂到对应的哈希桶上就行了。但实际为了缓解内存碎片的问题，page cache还需要尝试将还回来的span与其他空闲的span进行合并。

##### pagecache的前后页合并

​		合并的过程可以分为**向前合并**和**向后合并**。如果还回来的span的起始页号是num，该span所管理的页数是n。那么在向前合并时，就需要判断第num-1页对应span是否空闲，如果空闲则可以将其进行合并，并且合并后还需要继续向前尝试进行合并，直到不能进行合并为止。而在向后合并时，就需要判断第num+n页对应的span是否空闲，如果空闲则可以将其进行合并，并且合并后还需要继续向后尝试进行合并，直到不能进行合并为止。

  因此page cache在合并span时，是需要通过页号获取到对应的span的，这就是我们要把页号与span之间的映射关系存储到page cache的原因。

  但需要注意的是，当我们通过页号找到其对应的span时，这个span此时可能挂在page cache，也可能挂在central cache。而在合并时我们只能合并挂在page cache的span，因为挂在central cache的span当中的对象正在被其他线程使用。

  可是我们不能通过span结构当中的_useCount成员，来判断某个span到底是在central cache还是在page cache。因为当central cache刚向page cache申请到一个span时，这个span的_useCount就是等于0的，这时可能当我们正在对该span进行切分的时候，page cache就把这个span拿去进行合并了，这显然是不合理的。

  鉴于此，我们可以在span结构中再增加一个_isUse成员，用于标记这个span是否正在被使用，而当一个span结构被创建时我们默认该span是没有被使用的。



```cpp
//释放空闲的span回到PageCache，并合并相邻的span
void PageCache::ReleaseSpanToPageCache(Span* span)
{
	if (span->_n > NPAGES - 1) //大于128页直接释放给堆
	{
		void* ptr = (void*)(span->_pageId << PAGE_SHIFT);
		SystemFree(ptr);
		//delete span;
		_spanPool.Delete(span);

		return;
	}
	//对span的前后页，尝试进行合并，缓解内存碎片问题
	//1、向前合并
	while (1)
	{
		PAGE_ID prevId = span->_pageId - 1;
		//auto ret = _idSpanMap.find(prevId);
		////前面的页号没有（还未向系统申请），停止向前合并
		//if (ret == _idSpanMap.end())
		//{
		//	break;
		//}
		Span* ret = (Span*)_idSpanMap.get(prevId);
		if (ret == nullptr)
		{
			break;
		}
		//前面的页号对应的span正在被使用，停止向前合并
		//Span* prevSpan = ret->second;
		Span* prevSpan = ret;
		if (prevSpan->_isUse == true)
		{
			break;
		}
		//合并出超过128页的span无法进行管理，停止向前合并
		if (prevSpan->_n + span->_n > NPAGES - 1)
		{
			break;
		}
		//进行向前合并
		span->_pageId = prevSpan->_pageId;
		span->_n += prevSpan->_n;

		//将prevSpan从对应的双链表中移除
		_spanLists[prevSpan->_n].Erase(prevSpan);

		//delete prevSpan;
		_spanPool.Delete(prevSpan);
	}
	//2、向后合并
	while (1)
	{
		PAGE_ID nextId = span->_pageId + span->_n;
		//auto ret = _idSpanMap.find(nextId);
		////后面的页号没有（还未向系统申请），停止向后合并
		//if (ret == _idSpanMap.end())
		//{
		//	break;
		//}
		Span* ret = (Span*)_idSpanMap.get(nextId);
		if (ret == nullptr)
		{
			break;
		}
		//后面的页号对应的span正在被使用，停止向后合并
		//Span* nextSpan = ret->second;
		Span* nextSpan = ret;
		if (nextSpan->_isUse == true)
		{
			break;
		}
		//合并出超过128页的span无法进行管理，停止向后合并
		if (nextSpan->_n + span->_n > NPAGES - 1)
		{
			break;
		}
		//进行向后合并
		span->_n += nextSpan->_n;

		//将nextSpan从对应的双链表中移除
		_spanLists[nextSpan->_n].Erase(nextSpan);

		//delete nextSpan;
		_spanPool.Delete(nextSpan);
	}
	//将合并后的span挂到对应的双链表当中
	_spanLists[span->_n].PushFront(span);
	//建立该span与其首尾页的映射
	//_idSpanMap[span->_pageId] = span;
	//_idSpanMap[span->_pageId + span->_n - 1] = span;
	_idSpanMap.set(span->_pageId, span);
	_idSpanMap.set(span->_pageId + span->_n - 1, span);

	//将该span设置为未被使用的状态
	span->_isUse = false;
}
```



需要注意的是，在向前或向后进行合并的过程中：

1. 如果没有通过页号获取到其对应的span，说明对应到该页的内存块还未申请，此时需要停止合并。
2. 如果通过页号获取到了其对应的span，但该span处于被使用的状态，那我们也必须停止合并。
3. 如果合并后大于128页则不能进行本次合并，因为page cache无法对大于128页的span进行管理。

