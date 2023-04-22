---
title: Multithread 06) Lock-Based Stack & Queue
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **0. 개요**
 
앞서 배운 mutex, lock_guard, condition variable를 이용하여 멀티스레드 환경에서도 잘 동작할 수 있는 Lock-based Stack과 Lock-based Queue를 만든다. 

# **1. Lock-Based Stack**

**LockBasedStack.h**

```c++
#pragma once
#include <mutex>

template<typename T>
class LockBasedStack
{
public:
    LockBasedStack() {}
    LockBasedStack(const LockBasedStack&) = delete;
    LockBasedStack& operator=(const LockBasedStack&) = delete;

    void Push(T value)
    {
        lock_guard<mutex> lock(_mutex);
        _stack.push(std::move(value));
        _cv.nofity_one();
    }

    bool TryPop(T& value)
    {
        lock_guard<mutex> lock(_mutex);
        if(_stack.empty())
            return false; 

        value = std::move(_stack.top());
        _stack.pop();
        return true;
    }

    void WaitPop(T& value)
    {
        unique_guard<mutex> lock(_mutex);
        _cv.wait(lock, [this]{ return _stack.empty() == false; });
        value = std::move(_stack.top());
        _stack.pop();
    }

private:
    stack<T> _stack;
    mutex _mutex;
    condition_variable _cv;
}
```

# **2. Lock-Based Queue**

```c++
#pragma once
#include <mutex>

template<typename T>
class LockBasedQueue
{
public:
    LockBasedQueue() {}
    LockBasedQueue(const LockBasedQueue&) = delete;
    LockBasedQueue& operator=(const LockBasedQueue&) = delete;

    void Push(T value)
    {
        lock_guard<mutex> lock(_mutex);
        _queue.push(std::move(value));
        _cv.nofity_one();
    }

    bool TryPop(T& value)
    {
        lock_guard<mutex> lock(_mutex);
        if(_queue.empty())
            return false; 

        value = std::move(_queue.front());
        _queue.pop();
        return true;
    }

    void WaitPop(T& value)
    {
        unique_guard<mutex> lock(_mutex);
        _cv.wait(lock, [this]{ return _queue.empty() == false; });
        value = std::move(_queue.front());
        _queue.pop();
    }

private:
    queue<T> _queue;
    mutex _mutex;
    condition_variable _cv;
}
```

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
