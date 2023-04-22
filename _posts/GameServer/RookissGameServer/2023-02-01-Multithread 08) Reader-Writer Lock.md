---
title: Multithread 08) Reader-Writer Lock
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. C++ mutex의 한계**

mutex는 완전히 상호 배타적인 lock이기 때문에 굳이 배타적으로 동작하지 않아도 되는 스레드끼리도 배타성을 유지하게 된다. 예를 들어 대체로 Read로만 동작하고 아주 가끔 Write 하게 되는 데이터에 대해서도 그 가끔 Write할 때 에러가 나지 않게 하기 위해 계속 배타적으로 동작하게 되는 것이다. 이는 불필요한 성능 상의 저하로 이어질 수 있다. 

<br/> 

# **2. Reader-Writer Lock 구현**

이를 보안하기 위해 Read와 Write를 분리해서 lock을 걸 수 있다. C++의 라이브러리를 사용할 수도 있지만 의도대로 최적화 하기 위해 Reader-Writer lock을 직접 구현하여 사용할 수 있다. 그 예제는 아래와 같다. 

**CoreMacro.h**

```c++
#define USE_MANY_LOCKS(count)   Lock _locks[count];
#define USE_LOCK                USER_MANY_LOCKS(1)
#define READ_LOCK_IDX(idx)      ReadLockGuard readLockGuard_##idx(_locks[idx]);
#define READ_LOCK               READ_LOCK_IDX(0)
#define WRITE_LOCK_IDX(idx)      WriteLockGuard writeLockGuard_##idx(_locks[idx]);
#define WRITE__LOCK               WRITE__LOCK_IDX(0)
```

**Lock.h**

```c++
/*
[WWWWWWWW][WWWWWWWW][RRRRRRRR][RRRRRRRR]
W: Write Flag, Exclusive Lock Owner ThreadId
R: Read Flag, Shared Lock Count
*/

class Lock
{
    enum: uint32
    {
        ACQUIRE_TIMEOUT_TICK = 10000,
        MAX_SPIN_COUNT = 5000,
        WRITE_THREAD_MASK = 0xFFFF0000
        READ_COUNT_MASK = 0x0000FFFF
        EMPTY_FLAG = 0x00000000
    }

public:
    void WriteLock();
    void WriteUnlock();
    void ReadLock();
    void ReadUnlock();

private:
    Atomic<uint32> _lockFlag = EMPTY_FLAG;
    uint16  _writeCount = 0;
}

class ReadLockGuard
{
pubilc:
    ReadLockGuard(Lock& lock) : _lock(lock) { _lock.ReadLock(); }
    ~ReadLockGuard() { _lock.ReadUnlock(); }

private:
    Lock&   _lock;
}

class WriteLockGuard
{
pubilc:
    WriteLockGuard(Lock& lock) : _lock(lock) { _lock.WriteLock(); }
    ~WriteLockGuard() { _lock.WriteUnlock(); }

private:
    Lock&   _lock;
}
```

**Lock.cpp**

```c++
void Lock::WriteLock()
{
    // 동일한 스레드가 소유하고 있다면 성공
    const uint32 lockThreadId = (_lockFlag.load() & WRITE_THREAD_MASK) >> 16;
    if(LThreadId == lockThreadId)
    {
        _writeCount++;
        return;
    }
    
    // 아무도 소유하지 않을 때, 소유권을 얻고 성공
    const int64 beginTick = ::GetTickCount64();
    const uint32 desired = ((LThreadId << 16) & WRITE_THREAD_MASK);
    while(true)
    {
        for(uint32 spinCount = 0; spinCount < MAX_SPIN_COUNT; spinCount++ )
        {
            uint32 expected = EMPTY_FLAG;
            if(_lockFlag.compare_exchange_strong(OUT expected, desired))
            {
                _writeCount++;
                return;
            }
        }

        if(::GetTickCount64() - beginTick >= ACQUIRE_TIMEOUT_TICK)
            CRASH("LOCK_TIMEOUT");
        this_thread::yield();
    }
}

void Lock::WriteUnlock()
{
    //ReadLock을 다 풀기 전에는 WriteUnlock이 불가능
    if((_lockFlag.load() & READ_COUNT_MASK)!= 0)
        CRASH("INVALID_UNLOCK_ORDER");
        
    const int32 lockCount = --_writeCount;
    if(lockCount == 0)
        _lockFlag.store(EMPTY_FLAG);
}

void Lock::ReadLock()
{
    // 동일한 스레드가 소유하고 있다면 성공
    const uint32 lockThreadId = (_lockFlag.load() & WRITE_THREAD_MASK) >> 16;
    if(LThreadId == lockThreadId)
    {
        _lockFlag.fetch_add(1);
        return;
    }

    while(true)
    {
        for(uint32 spinCount = 0; spinCount < MAX_SPIN_COUNT; spinCount++)
        {
            uint expected = (_lockFlag.load()&READ_COUNT_MASK);
            if(_lockFlag.compare_exchange_strong(OUT expected, expected + 1))
                return;    
        }
        if(::GetTickCount64() - beginTick >= ACQUIRE_TIMEOUT_TICK)
            CRASH("LOCK_TIMEOUT");
        this_thread::yield();
    }
}

void Lock::ReadUnlock()
{
    if(( _lockFlag.fetch_sub(1) & READ_COUNT_MASK) == 0)
        CRASH("MULTIPLE_LOCK");
}
```

**Main.cpp**

```c++
class TestLock
{
    USE_LOCK;

public: 
    int32 TestRead()
    {
        READ_LOCK;

        if(_queue.empty())
            return -1;

        return _queue.front();
    }

    void TestPush()
    {
        WRITE_LOCK;
        _queue.push(rand()%100);
    }

    void TestPop()
    {
        WRITE_LOCK;

        if(_queue.empty() == false)
            _queue.pop();
    }

private:
    queue<int32> _queue;
}

TestLock testLock;

void ThreadWrite()
{
    while(true)
    {
        testLock.TestPush();
        this_thread::sleep_for(1ms);
        testLock.TestPop();
    }
}

void ThreadRead()
{
    while(true)
    {
        int32 value = testLock.TestRead();
        cout <<value <<endl; 
        this_thread::sleep_for(1ms);
    }
}

int main()
{
    for(int32 i = 0; i < 2; i++)
    {
        GThreadManager -> Launch(ThreadWrite);
    }

    for(int32 i = 0; i < 5; i++)
    {
        GThreadManager -> Launch(ThreadRead);
    }

    GThreadManager -> Join();
}
```
<br/>

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
