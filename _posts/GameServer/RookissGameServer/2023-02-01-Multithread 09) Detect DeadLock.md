---
title: Multithread 09) Detect DeadLock
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1.Detect DeadLock**

스레드 간의 그래프 구조를 만들어서 DFS 알고리즘을 통해 순방향 간선으로만 이루어져있는지, 교차 간선 혹은 역방향 간선이 존재하지는 않는지, 그로 인해 사이클이 발생하지 않는지 살펴봄으로써 DeadLock이 발생하기 전에 DeadLock을 미리 감지할 수 있다. 코드 예제는 아래와 같다. 

**DeadLockProfiler.h**
```c++
class DeadLockProfiler
{
public:
    void PushLock(const char* name);
    void PopLock(const char* name);
    void CheckCycle();

private:
    void Dfs(int32 index);

private:
    unordered_map<const char*, int32>   _nameTold;
    unordered_map<int32, const char*>   _idToName;
    stack<int32>                        _lockStack;
    map<int32, set<int32>>              _idToName;

    Mutex           _lock;
    vector<int32>   _discoveredOrder;
    int32           _discoveredCount = 0;
    vector<bool>    _finished;
    vector<int32>   _parent;
}
```


**DeadLockProfiler.cpp**
```c++
void DeadLockProfiler::PushLock(const char* name)
{
    LockGuard guard(_lock);
    int32 lockId = 0;

    auto findIt = _nameToId.find(name);
    if(findIt == _nameToId.end())
    {
        lockId = static_cast<int32>(_nameToId.size());
        _nameToId[name] = lockId;
        _idToName[lockId] = name;
    }
    else
    {
        lockId = findIt -> second;
    }
    
    if(_lockStack.empty() == false)
    {
        const int32 prevId = _lockStack.top();
        if(lockId != prevId)
        {
            set<int32>& history = _lockHistory[prevId];
            if(history.find(lockId) == history.end())
            {
                history.insert(lockId);
                CheckCycle();
            }
        }
    }

    _lockStack.push(lockId);
}

void DeadLockProfiler::PopLock(const char* name)
{
    LockGuard guard(_lock);

    if(_lockStack.empty())
        CRASH("MULTIPLE_UNLOCK");

    int32 lockId = _nameToId[name];
    if(_lockStack.top() != lockId)
        CRASH("INVALID_UNLOCK");

    _lockStack.pop();
}

void DeadLockProfiler::CheckCycle()
{
    const int32 lockCount = static_cast<int32>(_nameToId.size());
    _discoveredOrder = vector<int32>(lockCount, -1);
    _discoveredCount = 0;
    _finished = vector<bool>(lockCount, false);
    _parent = vector<int32>(lockCount, -1);

    for(int32 lockId = 0; lockId < lockCount; lockId++)
        Dfs(lockId);

    _discoveredOrder.clear();
    _finished.clear();
    _parent.clear(); 
}

void DeadLockProfiler::Dfs(int32 index)
{
    if(_discoveredOrder[here] != -1)
        return;

    _discoveredOrder[here] = _discoveredCount++;

    auto findIt = _lockHistory.find(here);
    if(findIt == _lockHistory.end())
    {
        _finished[here] = true;
        return;
    }

    set<int32>& nextSet = findIt -> second;
    for( int32 there : nextSet )
    {
        // 아직 방문한 적이 없다면 방문한다.
        if( _discoveredOrder[there] == -1)
        {
            _parent[there] = here;
            Dfs(there);
            continue;
        }

        // here가 there보다 먼저 발견되었다.
        // there은 here의 후손, 즉 순방향 간선이다.
        if( _discoveredOrder[there] < _discoveredOrder[here])
            continue;

        // 순방향이 아니거나
        // Dfs(there)가 아직 종료하지 않았다
        if(_finished[there] == false)
        {
            printf("%s -> %s\n", _idToName[here], _idToName[there]);

            int32 now = here;
            while(true)
            {
                printf("%s -> %s\n", _idToName[_parent[now]], _idToName[now]);
                now = _parent[now];
                if(now == there)
                    break;
            }
            CRASH("DEADLOCK_DETECTED");
        }
    }
}
```

이를 [Reader-Writer Lock][1]에 적용시킨 예제는 아래와 같다. 아래와 같이 수정하면 Lock을 걸 때마다 Lock 사이에 cycle이 생기진 않는지 추적하게 된다.  

**CoreGlobal.h**
```c++
extern class ThreadManager* GThreadManager;
extern class DeadLockProfiler* GDeadLockProfiler;
```

**CoreGlobal.cpp**
```c++
ThreadManager* GThreadManager = nullptr;
DeadLockProfiler* GDeadLockProfiler = nullptr;

class CoreGlobal
{
public:
    CoreGlobal()
    {
        GThreadManager = new ThreadManager();
        GDeadLockProfiler = new DeadLockProfiler();
    }
    ~CoreGlobal()
    {
        delete GThreadManager;
        delete GDeadLockProfiler;
    }
}GCoreGlobal;
```

**CoreMacro.h**

```c++
#define USE_MANY_LOCKS(count)   Lock _locks[count];
#define USE_LOCK                USER_MANY_LOCKS(1)
#define READ_LOCK_IDX(idx)      ReadLockGuard readLockGuard_##idx(_locks[idx], typeof(this).name);
#define READ_LOCK               READ_LOCK_IDX(0)
#define WRITE_LOCK_IDX(idx)      WriteLockGuard writeLockGuard_##idx(_locks[idx], typeof(this).name);
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
    //...
public:
    void WriteLock(const char* name);
    void WriteUnlock(const char* name);
    void ReadLock(const char* name);
    void ReadUnlock(const char* name);

    //...
}

class ReadLockGuard
{
pubilc:
    ReadLockGuard(Lock& lock, const char* name) : _lock(lock), _name(name) { _lock.ReadLock(name); }
    ~ReadLockGuard() { _lock.ReadUnlock(_name); }

private:
    Lock&   _lock;
    const char* _name;
}

class WriteLockGuard
{
pubilc:
    WriteLockGuard(Lock& lock, const char* name) : _lock(lock), _name(name) { _lock.WriteLock(name); }
    ~WriteLockGuard() { _lock.WriteUnlock(_name); }

private:
    Lock&   _lock;
    const char* _name;
}
```

**Lock.cpp**

```c++
void Lock::WriteLock(const char* name)
{
#if _DEBUG
    GDeadLockProfiler -> PushLock(name);
#endif

    //...

}

void Lock::WriteUnlock(const char* name)
{
#if _DEBUG
    GDeadLockProfiler -> PopLock(name);
#endif

    //...

}

void Lock::ReadLock(const char* name)
{
#if _DEBUG
    GDeadLockProfiler -> PushLock(name);
#endif

    //...
}

void Lock::ReadUnlock(const char* name)
{
#if _DEBUG
    GDeadLockProfiler -> PopLock(name);
#endif

    //...
}
```

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss

[1]: https://chw-owo.github.io/RookissGameServer/Multithread-08)-Reader-Writer-Lock/