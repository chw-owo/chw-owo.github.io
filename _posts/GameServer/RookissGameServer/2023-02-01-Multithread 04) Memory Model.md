---
title: Multithread 04) Memory Model
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Atomic 연산**

## **1) Atomic 연산**

atomic 연산 (원자적으로 한번에 일어나는 연산)에 한해, 모든 스레드가 동일 객체에 대해 동일한 수정 순서 관찰을 보장한다. 그러나 이는 순서를 보장하는 것이지, 바뀐 시점에서 수정된 값을 보장하는 것은 아닐 수 있다. 또, 동일한 객체가 아닐 경우에는 동일한 수정 순서 관찰도 보장되지 않는다. 

<br/> 

# **2. Memory Model**

## **1) Sequentially Consistent**

가장 엄격한 메모리 정책으로, 컴파일러 최적화의 여지가 적어서 직관적으로 코드가 동작한다. atomic 연산 시, memory_order::seq_cst 으로 해당 설정을 해줄 수 있다. 이를 사용할 경우 컴파일러가 코드를 자동으로 재배치 하는 것을 막게 되며 가시성 문제도 해결된다. 

## **2) Relaxed**

가장 자유로운 메모리 정책으로, 컴파일러 최적화의 여지가 많아서 직관적이지 않게 코드가 동작한다. atomic 연산 시, memory_order::relaxed 으로 해당 설정을 해줄 수 있다. 이를 사용할 경우 컴파일러가 코드를 자동으로 재배치하게 되며, 동일 객체에 대한 동일 순서 관찰만 보장하게 된다. 멀티스레드 환경에서 의도하지 않은 방향으로 동작할 수 있기 때문에 많이 사용하지 않는다.


## **2) Acquire-Release**

Sequentially Consistent와 Relaxed 사이 중간 쯤 성격을 지닌 메모리 정책이다. memory_order::acquire, memory_order::release로 설정을 해줄 수 있으며 둘을 함께 사용하고 싶은 경우 memory_order::acq_rel를 사용할 수 있다. memory_order::acquire, memory_order::release은 짝지어서 사용할 수 있는데, memory_order::release를 사용할 경우 해당 지점 이전 메모리 명령들이 해당 지점 이후로 재배치되는 것을 막는다. 그 상태에서 memory_order::acquire를 사용할 경우 같은 변수를 읽는 스레드가 있을 때, release 이전 명령어들이 acquire 하는 순간에서 객체들 간의 수정 순서가 보장된 채로 관찰 가능하게 된다. 사용 예시는 아래와 같다. 

```c++
void Producer()
{
    value1 = 10;
    value2 = 20;
    ready.store(true, memory_order::memory_order_release);
}

void Consumer()
{
    while(ready.load(true, memory_order::memory_order_acquire) == false);
    cout << value1 << " " << value2 << endl;
}
```
위 예시에서 release, acquire로 인해 value1 = 10, value2 = 20임을 보장받을 수 있게 된다. 

```c++
void Producer()
{
    value1 = 10;
    value2 = 20;
    atomic_thread_fence(memory_order::memory_order_release);
}
```
절취선을 긋는 것만이 목적이라면 atomic_thread_fence로 대체할 수 있다. 

<br/>

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
