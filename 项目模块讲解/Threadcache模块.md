# Threadcache模块

### threadcache整体设计
  定长内存池只支持固定大小内存块的申请释放，因此定长内存池中只需要一个自由链表管理释放回来的内存块。现在我们要支持申请和释放不同大小的内存块，那么我们就需要多个自由链表来管理释放回来的内存块，因此thread cache实际上一个哈希桶结构，每个桶中存放的都是一个自由链表。

  thread cache支持小于等于256KB内存的申请，如果我们将每种字节数的内存块都用一个自由链表进行管理的话，那么此时我们就需要20多万个自由链表，光是存储这些自由链表的头指针就需要消耗大量内存，这显然是得不偿失的。

  这时我们可以选择做一些平衡的牺牲，让这些字节数按照某种规则进行对齐，例如我们让这些字节数都按照8字节进行向上对齐，那么thread cache的结构就是下面这样的，此时当线程申请1-8字节的内存时会直接给出8字节，而当线程申请9~16字节的内存时会直接给出16字节，以此类推。

![2.png](https://img1.imgtp.com/2023/08/28/dhs3yO0w.png)



因此当线程要申请某一大小的内存块时，就需要经过某种计算得到对齐后的字节数，进而找到对应的哈希桶，如果该哈希桶中的自由链表中有内存块，那就从自由链表中头删一个内存块进行返回；如果该自由链表已经为空了，那么就需要向下一层的central cache进行获取了。

### threadcache哈希桶映射规则

由于对齐的原因，就可能会产生一些碎片化的内存无法被利用，比如线程只申请了6字节的内存，而thread cache却直接给了8字节的内存，这多给出的2字节就无法被利用，导致了一定程度的空间浪费，这些因为某些对齐原因导致无法被利用的内存，就是内存碎片中的内部碎片。

#### 我们设计出如下映射规则：

|        字节数        | 对齐数 | 哈希桶下标 |
| :------------------: | :----: | :--------: |
|       [1,128]        |   8    |   [0,16)   |
|     [128+1,1024]     |   16   |  [16,72)   |
|   [1024+1,8x1024]    |  128   |  [72,128)  |
|  [8x1024+1,64x1024]  |  1024  | [128,184)  |
| [64x1024+1,256x1024] | 8x1024 | [184,208)  |

#### 空间浪费率计算

虽然对齐产生的内碎片会引起一定程度的空间浪费，但按照上面的对齐规则，我们可以将浪费率控制到百分之十左右。需要说明的是，1~128这个区间我们不做讨论，因为1字节就算是对齐到2字节也有百分之五十的浪费率，这里我们就从第二个区间开始进行计算。


浪 费 率 = 浪 费 的 字 节 数 /对 齐 后 的 字 节 数

根据上面的公式，我们要得到某个区间的最大浪费率，就应该让分子取到最大，让分母取到最小。比如129~1024这个区间，该区域的对齐数是16，那么最大浪费的字节数就是15，而最小对齐后的字节数就是这个区间内的前16个数所对齐到的字节数，也就是144，那么该区间的最大浪费率也就是15 ÷ 144 ≈ 10.42 %。同样的道理，后面两个区间的最大浪费率分别是127 ÷ 1152 ≈ 11.02 % 和1023 ÷ 9216 ≈ 11.10 % 。

#### 映射和对齐部分

```cpp

//管理对齐和映射等关系
class SizeClass
{
public:
	//整体控制在最多10%左右的内碎片浪费
	//[1,128]              8byte对齐       freelist[0,16)
	//[128+1,1024]         16byte对齐      freelist[16,72)
	//[1024+1,8*1024]      128byte对齐     freelist[72,128)
	//[8*1024+1,64*1024]   1024byte对齐    freelist[128,184)
	//[64*1024+1,256*1024] 8*1024byte对齐  freelist[184,208)

	//一般写法
	//static inline size_t _RoundUp(size_t bytes, size_t alignNum)
	//{
	//	size_t alignSize = 0;
	//	if (bytes%alignNum != 0)
	//	{
	//		alignSize = (bytes / alignNum + 1)*alignNum;
	//	}
	//	else
	//	{
	//		alignSize = bytes;
	//	}
	//	return alignSize;
	//}

	//位运算写法
	static inline size_t _RoundUp(size_t bytes, size_t alignNum)
	{
		return ((bytes + alignNum - 1)&~(alignNum - 1));
	}

	//获取向上对齐后的字节数
	static inline size_t RoundUp(size_t bytes)
	{
		if (bytes <= 128)
		{
			return _RoundUp(bytes, 8);
		}
		else if (bytes <= 1024)
		{
			return _RoundUp(bytes, 16);
		}
		else if (bytes <= 8 * 1024)
		{
			return _RoundUp(bytes, 128);
		}
		else if (bytes <= 64 * 1024)
		{
			return _RoundUp(bytes, 1024);
		}
		else if (bytes <= 256 * 1024)
		{
			return _RoundUp(bytes, 8 * 1024);
		}
		else
		{
			//assert(false);
			//return -1;
			//大于256KB的按页对齐
			return _RoundUp(bytes, 1 << PAGE_SHIFT);
		}
	}

	//一般写法
	//static inline size_t _Index(size_t bytes, size_t alignNum)
	//{
	//	size_t index = 0;
	//	if (bytes%alignNum != 0)
	//	{
	//		index = bytes / alignNum;
	//	}
	//	else
	//	{
	//		index = bytes / alignNum - 1;
	//	}
	//	return index;
	//}

	//位运算写法
	static inline size_t _Index(size_t bytes, size_t alignShift)
	{
		return ((bytes + (1 << alignShift) - 1) >> alignShift) - 1;
	}

```



###  设计思想

##### Threadcache TLS无锁访问

每个线程都有一个自己独享的thread cache，那应该如何创建这个thread cache呢？我们不能将这个thread cache创建为全局的，因为全局变量是所有线程共享的，这样就不可避免的需要锁来控制，增加了控制成本和代码复杂度。

要实现每个线程无锁的访问属于自己的thread cache，我们需要用到线程局部存储TLS(Thread Local Storage)，这是一种变量的存储方法，使用该存储方法的变量在它所在的线程是全局可访问的，但是不能被其他线程访问到，这样就保持了数据的线程独立性。

```cpp
class ThreadCache
{
public:
	//申请内存对象
	void* Allocate(size_t size);

	//释放内存对象
	void Deallocate(void* ptr, size_t size);

	//从中心缓存获取对象
	void* FetchFromCentralCache(size_t index, size_t size);

	//释放对象导致链表过长，回收内存到中心缓存
	void ListTooLong(FreeList& list, size_t size);
private:
	FreeList _freeLists[NFREELISTS]; //哈希桶
};

//TLS - Thread Local Storage
static _declspec(thread) ThreadCache* pTLSThreadCache = nullptr;
```



```
//申请内存对象
void* ThreadCache::Allocate(size_t size)
{
	assert(size <= MAX_BYTES); //thread cache只处理小于等于MAX_BYTES的内存申请
	size_t alignSize = SizeClass::RoundUp(size);
	size_t index = SizeClass::Index(size);
	if (!_freeLists[index].Empty())
	{
		return _freeLists[index].Pop();
	}
	else
	{
		return FetchFromCentralCache(index, alignSize);
	}
}

//释放内存对象
void ThreadCache::Deallocate(void* ptr, size_t size)
{
	assert(ptr);
	assert(size <= MAX_BYTES);

	//找出对应的自由链表桶将对象插入
	size_t index = SizeClass::Index(size);
	_freeLists[index].Push(ptr);

	//当自由链表长度大于一次批量申请的对象个数时就开始还一段list给central cache
	if (_freeLists[index].Size() >= _freeLists[index].MaxSize())
	{
		ListTooLong(_freeLists[index], size);
	}
}
```



### 慢反馈调节算法

当thread cache向central cache申请内存时，central cache应该给出多少个对象呢？这是一个值得思考的问题，如果central cache给的太少，那么thread cache在短时间内用完了又会来申请；但如果一次性给的太多了，可能thread cache用不完也就浪费了。

鉴于此，我们这里采用了一个慢开始反馈调节算法。当thread cache向central cache申请内存时，如果申请的是较小的对象，那么可以多给一点，但如果申请的是较大的对象，就可以少给一点。
```cpp
//从中心缓存获取对象
void* ThreadCache::FetchFromCentralCache(size_t index, size_t size)
{
	//慢开始反馈调节算法
	//1、最开始不会一次向central cache一次批量要太多，因为要太多了可能用不完
	//2、如果你不断有size大小的内存需求，那么batchNum就会不断增长，直到上限
	size_t batchNum = min(_freeLists[index].MaxSize(), SizeClass::NumMoveSize(size));
	if (batchNum == _freeLists[index].MaxSize())
	{
		_freeLists[index].MaxSize() += 1;       // 反馈调节
	}
	void* start = nullptr;
	void* end = nullptr;
	size_t actualNum = CentralCache::GetInstance()->FetchRangeObj(start, end, batchNum, size);
	assert(actualNum >= 1); //至少有一个

	if (actualNum == 1) //申请到对象的个数是一个，则直接将这一个对象返回即可
	{
		assert(start == end);
		return start;
	}
	else //申请到对象的个数是多个，还需要将剩下的对象挂到thread cache中对应的哈希桶中
	{
		_freeLists[index].PushRange(NextObj(start), end, actualNum - 1);
		return start;
	}
}
```





此时当thread cache申请对象时，我们会比较_maxSize和计算得出的值，取出其中的较小值作为本次申请对象的个数。此外，如果本次采用的是_maxSize的值，那么还会将thread cache中该自由链表的_maxSize的值进行加一。

因此，thread cache第一次向central cache申请某大小的对象时，申请到的都是一个，但下一次thread cache再向central cache申请同样大小的对象时，因为该自由链表中的_maxSize增加了，最终就会申请到两个。直到该自由链表中_maxSize的值，增长到超过计算出的值后就不会继续增长了，此后申请到的对象个数就是计算出的个数。（这有点像网络中拥塞控制的机制）



#### 从中心缓存获取对象

```cpp
//从中心缓存获取对象
void* ThreadCache::FetchFromCentralCache(size_t index, size_t size)
{
	size_t batchNum = min(_freeLists[index].MaxSize(), SizeClass::NumMoveSize(size));
	if (batchNum == _freeLists[index].MaxSize())
	{
		_freeLists[index].MaxSize() += 1;
	}
	void* start = nullptr;
	void* end = nullptr;
	size_t actualNum = CentralCache::GetInstance()->FetchRangeObj(start, end, batchNum, size);
	assert(actualNum >= 1); //至少有一个

	if (actualNum == 1) //申请到对象的个数是一个，则直接将这一个对象返回即可
	{
		assert(start == end);
		return start;
	}
	else //申请到对象的个数是多个，还需要将剩下的对象挂到thread cache中对应的哈希桶中
	{
		_freeLists[index].PushRange(NextObj(start), end, actualNum - 1);
		return start;
	}
}
```



#### threadcache回收内存

当某个线程申请的对象不用了，可以将其释放给thread cache，然后thread cache将该对象插入到对应哈希桶的自由链表当中即可。

但是随着线程不断的释放，对应自由链表的长度也会越来越长，这些内存堆积在一个thread cache中就是一种浪费，我们应该将这些内存还给central cache，这样一来，这些内存对其他线程来说也是可申请的，因此当thread cache某个桶当中的自由链表太长时我们可以进行一些处理。

如果thread cache某个桶当中自由链表的长度超过它一次批量向central cache申请的对象个数，那么此时我们就要把该自由链表当中的这些对象还给central cache。



```cpp
//释放内存对象
void ThreadCache::Deallocate(void* ptr, size_t size)
{
	assert(ptr);
	assert(size <= MAX_BYTES);

	//找出对应的自由链表桶将对象插入
	size_t index = SizeClass::Index(size);
	_freeLists[index].Push(ptr);

	//当自由链表长度大于一次批量申请的对象个数时就开始还一段list给central cache
	if (_freeLists[index].Size() >= _freeLists[index].MaxSize())
	{
		ListTooLong(_freeLists[index], size);
	}
}

```



当自由链表的长度大于一次批量申请的对象时，我们具体的做法就是，从该自由链表中取出一次批量个数的对象，然后将取出的这些对象还给central cache中对应的span即可。

```cpp
//释放对象导致链表过长，回收内存到中心缓存
void ThreadCache::ListTooLong(FreeList& list, size_t size)
{
	void* start = nullptr;
	void* end = nullptr;
	//从list中取出一次批量个数的对象
	list.PopRange(start, end, list.MaxSize());
	
	//将取出的对象还给central cache中对应的span
	CentralCache::GetInstance()->ReleaseListToSpans(start, size);
}

```



当thread cache的某个自由链表过长时，我们实际就是把这个自由链表当中全部的对象都还给central cache了，但这里在设计PopRange接口时还是设计的是取出指定个数的对象，因为在某些情况下当自由链表过长时，我们可能并不一定想把链表中全部的对象都取出来还给central cache，这样设计就是为了增加代码的可修改性。

其次，当我们判断thread cache是否应该还对象给central cache时，还可以综合考虑每个thread cache整体的大小。比如当某个thread cache的总占用大小超过一定阈值时，我们就将该thread cache当中的对象还一些给central cache，这样就尽量避免了某个线程的thread cache占用太多的内存。对于这一点，在tcmalloc当中就是考虑到了的。
