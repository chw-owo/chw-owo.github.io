---
title: Memory 03) Memory Pool 
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Memory Pool**

[Allocator 포스트][1]에서 처럼 새로 메모리를 할당할 때마다 매번 페이지 단위로 할당하면 안전하게 작업할 순 있겠지만 메모리 낭비가 심해질 것이다. 따라서 우선 메모리를 할당 받은 다음, 할당 요청이 들어올 때마다 나누어 주면 안전하게 작업하는 동시에 메모리 낭비도 줄일 수 있다. 실제 메모리 할당도 이러한 원리로 이루어진다. 이 작업을 커스텀하여 구현한 예제는 아래와 같다. 

**MemoryPool.h**

```c++
struct MemoryHeader
{
    MemoryHeader(int32 size): allocSize(size) {}
    
    static void* AttachHeader(MemoryHeader* header, int32 size)
    {
        new(header)MemoryHeader(size);
        return reinterpret_case<void*>(++header);
    }

    static MemoryHeader* DetachHeader(MemoryHeader* header, int32 size)
    {
        MemoryHeader* header = reinterpret_cast<MemoryHeader*>(ptr) - 1;
        return header;
    }

    int32 allocSize;
};

class MemoryPool
{
public:
    MemoryPool(int32 allocSize);
    ~MemoryPool();

    void            Push(MemoryHeader* ptr);
    MemoryHeader*   Pop();

private:
    int32 _allocSize = 0;
    atomic<int32> _allocCount = 0;

    USE_LOCK;
    queue<MemoryHeader*> _queue;
};
```

**MemoryPool.cpp**

```c++

MemoryPool::MemoryPool(int32 allocSize) : _allocSize(allocSize)
{
}

MemoryPool::~MemoryPool()
{
	while (_queue.empty() == false)
	{
		MemoryHeader* header = _queue.front();
		_queue.pop();
		::free(header);
	}
}

void MemoryPool::Push(MemoryHeader* ptr)
{
	WRITE_LOCK;
	ptr->allocSize = 0;
	_queue.push(ptr);
	_allocCount.fetch_sub(1);
}

MemoryHeader* MemoryPool::Pop()
{
	MemoryHeader* header = nullptr;

	{
		WRITE_LOCK;
		if (_queue.empty() == false)
		{
			header = _queue.front();
			_queue.pop();
		}
	}

	if (header == nullptr)
		header = reinterpret_cast<MemoryHeader*>(::malloc(_allocSize));
	else
		ASSERT_CRASH(header->allocSize == 0);

	_allocCount.fetch_add(1);

	return header;
}
```

**Memory.h**
```c++
class MemoryPool;
class Memory
{
	enum
	{
		POOL_COUNT = (1024 / 32) + (1024 / 128) + (2048 / 256),
		MAX_ALLOC_SIZE = 4096
	};

public:
	Memory();
	~Memory();

	void*	Allocate(int32 size);
	void	Release(void* ptr);

private:
	vector<MemoryPool*> _pools;
	MemoryPool* _poolTable[MAX_ALLOC_SIZE + 1];
};
```

**Memory.cpp**

```c++

Memory::Memory()
{
	int32 size = 0;
	int32 tableIndex = 0;

	for (size = 32; size <= 1024; size += 32)
	{
		MemoryPool* pool = new MemoryPool(size);
		_pools.push_back(pool);

		while (tableIndex <= size)
		{
			_poolTable[tableIndex] = pool;
			tableIndex++;
		}
	}

	for (; size <= 2048; size += 128)
	{
		MemoryPool* pool = new MemoryPool(size);
		_pools.push_back(pool);

		while (tableIndex <= size)
		{
			_poolTable[tableIndex] = pool;
			tableIndex++;
		}
	}

	for (; size <= 4096; size += 256)
	{
		MemoryPool* pool = new MemoryPool(size);
		_pools.push_back(pool);

		while (tableIndex <= size)
		{
			_poolTable[tableIndex] = pool;
			tableIndex++;
		}
	}
}

Memory::~Memory()
{
	for (MemoryPool* pool : _pools)
		delete pool;
	_pools.clear();
}

void* Memory::Allocate(int32 size)
{
	MemoryHeader* header = nullptr;
	const int32 allocSize = size + sizeof(MemoryHeader);

	if (allocSize > MAX_ALLOC_SIZE)
	    header = reinterpret_cast<MemoryHeader*>(::malloc(allocSize));
	else
		header = _poolTable[allocSize]->Pop();

	return MemoryHeader::AttachHeader(header, allocSize);
}

void Memory::Release(void* ptr)
{
	MemoryHeader* header = MemoryHeader::DetachHeader(ptr);
	const int32 allocSize = header->allocSize;
	ASSERT_CRASH(allocSize > 0);

	if (allocSize > MAX_ALLOC_SIZE)
	    ::free(header);
	else
		_poolTable[allocSize]->Push(header);
}
```

**CoreGlobal.h**
```c++
extern class Memory*    GMemory;
```


**CoreGlobal.cpp**
```c++
ThreadManager*      GThreadManager = nullptr;
DeadLockProfiler*   GDeadLockProfiler = nullptr;
Memory*             GMemory = nullptr;
class CoreGlobal
{
public:
    CoreGlobal()
    {
        GThreadManager = new ThreadManager();
        GMemory = new Memory();
        GDeadLockProfiler = new DeadLockProfiler();
    }
    ~CoreGlobal()
    {
        delete GThreadManager;
        delete GMemory;
        delete GDeadLockProfiler;
    }
}GCoreGlobal;
```

**Allocator.h**
```c++
#pragma once
class StompAllocator
{
    enum { PAGE_SIZE = 0x1000 };

public:
    static void*    Alloc(int32 size);
    static void     Release(void* ptr);
}
```

**Allocator.cpp**

```c++
void* PoolAllocator::Alloc(int32 size)
{
    return GMemory -> Allocate(size);
    
}

void PoolAllocator::Release(void* ptr)
{
    GMemory -> Release(ptr);
}
```

그 외의 파일에서도 StompAllocator를 PoolAllocator로 변환해주면 new, delete를 할 때마다 Memory Pooling이 일어나게 된다. 

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss

[1]: https://chw-owo.github.io/iocpserver/Memory-02)-Allocator/