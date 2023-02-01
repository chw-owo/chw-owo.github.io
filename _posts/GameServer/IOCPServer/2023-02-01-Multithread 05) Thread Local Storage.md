---
title: Multithread 05) Thread Local Storage
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Thread Local Storage, TLS란?**

TLS는 스레드별로 부여되는 공간을 의미하며, TLS 영역에서 일어나는 일은 각 스레드만이 접근할 수 있어서 경합이 발생하지 않는다. 반면 Heap 영역, data 영역은 공유되는 공간이기 때문에 경합이 발생할 수 있다. TLS와 stack의 차이점은, stack은 함수를 위해 존재하는 공간이기 때문에 함수 소멸 이후까지 장기적으로 쓰여야 하는 값을 보관하기엔 부적절하다. 반면 TLS는 전역으로도 사용할 수 있기에 장기적으로 쓰여야 하는 값을 보관할 수 있다. 

# **2. TLS 예제**

```c++
thread_local int32 LThreadId = 0;

void ThreadMiain(int32 threadId)
{
    LThreadId = threadId;

    while(true)
    {
        cout<< LThreadId << endl;
        this_thread::sleep_for(1s);
    }
}
int main()
{
    vector<thread> threads;
    for(int32 i = 0; i < 5; i++)
    {
        int32 threadId = i+1;
        threads.push_back(thread(ThreadMain, threadId));
    }

    for(thread& t : threads)
        t.join();
}
```

각 스레드에 고유한 숫자가 부여되어, 타 스레드에 의해 덮어씌워지지 않는 것을 볼 수 있다. LThreadId가 TLS 영역에 선언된 변수, 즉 스레드 별 전역 변수에 해당하기 때문이다. C++11 이전, windows API에서는 아래와 같이 선언했다. 

```c++
__declspec(thread) int32 value;
```

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
