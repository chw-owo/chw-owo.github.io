---
title: Memory 03) Memory Pool 
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Memory Pool ver 1**

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

# **2. Memory Pool ver 2**

LockFreeStack을 이용하여 Memory Pool을 구현할 수 있다. 아래 예제는 LockFreeStack을 이용하여 c++의 메모리 풀 라이브러리를 비슷하게 간략화하여 구현한 것으로, 싱글 스레드에서는 문제 없이 동작하지만 다수의 스레드가 동시에 pop 하게 되면 문제가 생길 수 있다. 이는 동작 방식을 이해하기 위해 작성한 것이지 실제로 사용하기엔 안전하지 않은 코드임을 참고해야 한다.

**LockFreeStack.h**
```c++
DECLSPEC_ALIGN(16)
struct sListEntry
{
	sListEntry* next;
}

DECLSPEC_ALIGN(16)
struct sListHeader
{
	SListHeader()
	{
		alignment = 0;
		region = 0;
	}

	union
	{
		struct
		{
			uint64 alignment;
			uint64 region;
		} DUMMYSTRUCTNAME;

		struct
		{
			uint64 depth : 16;
			uint64 sequence : 48;
			uint64 reservec : 4;
			uint64 next : 60;
		} HeaderX64;
	};
};

void InitializeHead(sListHeader* header);
void PushEntrySList(sListHeader* header, sListEntry* entry);
sListEntry* PopEntrySList(SListHeader* header);
```

**LockFreeStack.cpp**
```c++
void InitializeHead(sListHeader* header)
{
	header -> alignment = 0;
	header -> region = 0;	
}

void PushEntrySList(sListHeader* header, sListEntry* entry)
{
	sListHeader expected = {};
	sListHeader desired = {};

	// 16 bytes 정렬
	desired HeaderX64.next = (((uint64)entry) >> 4);
	while(true)
	{
		expected = *header;
		entry -> next = (SListEntry*)(((uint64)expected.HeaderX64.next) << 4);
		desired.HeaderX64.depth = expected.HeaderX64.depth + 1;
		desired.HeaderX64.sequence = expected.HeaderX64.sequence + 1;

		if(::InterlockedComparedExchage128((int64*)header, desired.region, desired.alignment, (int64*)&expected) == 1 )
			break;
	}

}

sListEntry* PopEntrySList(SListHeader* header)
{
	sListHeader expected = {};
	sListHeader desired = {};
	sListEntry* entry = nullptr;

	// 16 bytes 정렬
	desired HeaderX64.next = (((uint64)entry) >> 4);
	while(true)
	{
		expected = *header;
		entry= (SListEntry*)(((uint64)expected.HeaderX64.next) << 4);
		if(entry == nullptr)
			break;

		desired.HeaderX64.next = ((uint64)expected.HeaderX64.next) >> 4;
		desired.HeaderX64.depth = expected.HeaderX64.depth - 1;
		desired.HeaderX64.sequence = expected.HeaderX64.sequence + 1;

		if(::InterlockedComparedExchage128((int64*)header, desired.region, desired.alignment, (int64*)&expected) == 1 )
			break;
	}

	return entry;
}
```

**Main.cpp**
```c++
DECLSPEC_ALIGN(16)
class Data
{
public: 
	SListEntry _entry;
	int64 _rand = rand() % 1000;
}

SListHeader* GHeader;
int main()
{
	GHeader = new SListHeader();
	ASSERT_CRASH(((uint64)GHeader % 16) == 0);
	InitializeHead(GHeader);

	for(int32 i = 0; i < 3; i++)
	{
		GThreadManger -> Launch([]()
		{
			while(true)
			{
				Data* data = new Data();
				ASSERT_CRASH(((uint64)data % 16) == 0);
				PushEntrySList(GHeader, (SListEntry*)data);
				this_thread: sleep_for(10ms);
			}
		});
	}

	for(int32 i = 0; i < 3; i++)
	{
		GThreadManger -> Launch([]()
		{
			while(true)
			{
				Data* pop = nullptr;
				pop = (Data*)PopEntrySList(GHeader);

				if(pop)
				{
					cout << pop -> _rand << endl;
					delete pop; 
				}
				else
				{
					cout << "None" << endl;
				}
			}
		});
	}
}
```
이때 4만큼 shift 연산을 할 수 있는 이유는 16bytes 정렬을 하고 있어서 하위 4it가 0으로 고정되어있기 때문이다. 만약 16bytes 정렬이 아니거나 shift 연산을 다른 수로 해주게 되면 잘못된 주소로 접근하여 위험한 상황이 생길 수 있다. 

<br/>

# **2. C++ Memory Pool Library**

위 예제에서 메모리 풀을 위해 만든 부분들을 C++ Library로 바꾼 예제는 아래와 같다. 

```c++
DECLSPEC_ALIGN(16)
class Data
{
public: 
	SLIST_ENTRY _entry;
	int64 _rand = rand() % 1000;
}

SLIST_HEADER* GHeader;
int main()
{
	GHeader = new SLIST_HEADER();
	ASSERT_CRASH(((uint64)GHeader % 16) == 0);
	::InitializeSListHead(GHeader);

	for(int32 i = 0; i < 3; i++)
	{
		GThreadManger -> Launch([]()
		{
			while(true)
			{
				Data* data = new Data();
				ASSERT_CRASH(((uint64)data % 16) == 0);
				::InterlockedPushEntrySList(GHeader, (PSLIST_ENTRY*)data);
				this_thread: sleep_for(10ms);
			}
		});
	}

	for(int32 i = 0; i < 3; i++)
	{
		GThreadManger -> Launch([]()
		{
			while(true)
			{
				Data* pop = nullptr;
				pop = (Data*)::InterlockedPopEntrySList(GHeader);

				if(pop)
				{
					cout << pop -> _rand << endl;
					delete pop; 
				}
				else
				{
					cout << "None" << endl;
				}
			}
		});
	}
}
```

혹은 MemoryHeader를 만들 때 SLIST_ENTRY를 상속 받아 만들어서 아래와 같이 구현할 수도 있다. 

**MemoryPool.h**
```c++
#pragma once

enum
{
	SLIST_ALIGNMENT = 16
};

DECLSPEC_ALIGN(SLIST_ALIGNMENT)
struct MemoryHeader : public SLIST_ENTRY
{
	// [MemoryHeader][Data]
	MemoryHeader(int32 size) : allocSize(size) { }

	static void* AttachHeader(MemoryHeader* header, int32 size)
	{
		new(header)MemoryHeader(size); // placement new
		return reinterpret_cast<void*>(++header);
	}

	static MemoryHeader* DetachHeader(void* ptr)
	{
		MemoryHeader* header = reinterpret_cast<MemoryHeader*>(ptr) - 1;
		return header;
	}

	int32 allocSize;
};

DECLSPEC_ALIGN(SLIST_ALIGNMENT)
class MemoryPool
{
public:
	MemoryPool(int32 allocSize);
	~MemoryPool();

	void			Push(MemoryHeader* ptr);
	MemoryHeader*	Pop();

private:
	SLIST_HEADER	_header;
	int32			_allocSize = 0;
	atomic<int32>	_allocCount = 0;
};
```

**MemoryPool.cpp**
```c++

MemoryPool::MemoryPool(int32 allocSize) : _allocSize(allocSize)
{
	::InitializeSListHead(&_header);
}

MemoryPool::~MemoryPool()
{
	while (MemoryHeader* memory = static_cast<MemoryHeader*>(::InterlockedPopEntrySList(&_header)))
		::_aligned_free(memory);
}

void MemoryPool::Push(MemoryHeader* ptr)
{
	ptr->allocSize = 0;

	::InterlockedPushEntrySList(&_header, static_cast<PSLIST_ENTRY>(ptr));

	_allocCount.fetch_sub(1);
}

MemoryHeader* MemoryPool::Pop()
{
	MemoryHeader* memory = static_cast<MemoryHeader*>(::InterlockedPopEntrySList(&_header));

	if (memory == nullptr)
	{
		memory = reinterpret_cast<MemoryHeader*>(::_aligned_malloc(_allocSize, SLIST_ALIGNMENT));
	}
	else
	{
		ASSERT_CRASH(memory->allocSize == 0);
	}

	_allocCount.fetch_add(1);

	return memory;
}
```

**Main.cpp**
```c++

class Knight
{
public:
	int32 _hp = rand() % 1000;
};

int main()
{
	for (int32 i = 0; i < 5; i++)
	{
		GThreadManager->Launch([]()
		{
			while (true)
			{
				Knight* knight = xnew<Knight>();
				cout << knight->_hp << endl;
				this_thread::sleep_for(10ms);
				xdelete(knight);
			}
		});
	}
	GThreadManager->Join();
}
```

이때 동일한 클래스끼리 모아서 관리하게 되면 메모리 누수가 일어난 상황 등에서 더 편하게 관리할 수 있다. 

<br/>

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss

[1]: https://chw-owo.github.io/iocpserver/Memory-02)-Allocator/