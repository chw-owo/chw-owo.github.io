---
title: Multithread 2) Atomic
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 



## **1. 스레드 간의 데이터 공유**

```c++
using namespace std;
#include <thread> 

int sum = 0;

void Add()
{
    for (int i = 0; i < 1000000; i++)
    {
        sum++;
    }
}

void Sub()
{
    for (int i = 0; i < 1000000; i++)
    {
        sum--;
    }
}

int main()
{
    thread t1(Add);
    thread t2(Sub);
    t1.join();
    t2.join();

    cout << sum << endl;
}
```
위 함수를 실행해보면 1000000번 더하고 1000000번 빼니까 0이 출력될 것 같지만 실제로는 이상하고 거대한 음수값이 출력된다. (ex -987663) 심지어 실행할 때마다 다른 음수값이 출력된다. 이것은 한 코어를 공유하는 스레드들이 서로 Heap 영역과 Data 공유하기 때문에 생기는 문제이다. 구체적으로 어떤 상황인지 어셈블리어로 살펴보자. 

```
mov     eax, dword ptr [sum]
inc     eax
mov     dword ptr [sum], eax
```
```
mov     eax, dword ptr [sum]
dec     eax
mov     dword ptr [sum], eax
```

sum++, sum--는 각각 위와 같이 실행된다. C++ 코드에서는 한줄로 보였지만 실제로는 위와 같이 세줄로 이루어진 것이다. 스레드는 하나의 코어가 빠른 속도로 번갈아가며 각 스레드의 명령들을 실행하는 것인데, 당연하게도 C++ 코드가 아니라 기계어 코드를 기본 단위로 실행하게 된다. 따라서 위의 명령어들이 어떤 순서로 실행될지 보장받지 못한다. 

```
mov     eax, dword ptr [sum]
inc     eax
mov     dword ptr [sum], eax
mov     eax, dword ptr [sum]
dec     eax
mov     dword ptr [sum], eax
```
위와 같이 순차적으로 실행된다면 예상대로 0이라는 결과가 나오겠지만

```
mov     eax, dword ptr [sum]
inc     eax
mov     eax, dword ptr [sum]
mov     dword ptr [sum], eax
dec     eax
mov     dword ptr [sum], eax
```
이렇게 순서가 뒤섞여서 실행된다면 예상하지 못한 값이 나오게 될 것이다. 그리고 스레드는 이를 고려하여 순서가 지정되지 않기 때문에 높은 확률로 순서가 뒤섞일 것이다. 이러한 이유로 스레드에서 공유된 데이터를 사용할 때는 별도의 처리가 필요하다.  

<br/>

## **2. Atomic**

atom(미소 존재)에서 유래한 atomic은 미소 존재처럼 더 이상 쪼개지지 않는 단위임을 나타낼 때 사용한다. 

(+ 화학에서는 atom이 원자(더 쪼개질 수 있는 존재)로 사용되지만 철학에서는 사물 구성 최후의 미소 존재를 나타내는 용어로 쓰인다.)

```c++
using namespace std;
#include <thread> 
#include <atomic>

atomic<int> sum = 0;

void Add()
{
    for (int i = 0; i < 1000000; i++)
    {
        sum++;
    }
}

void Sub()
{
    for (int i = 0; i < 1000000; i++)
    {
        sum--;
    }
}

int main()
{
    thread t1(Add);
    thread t2(Sub);
    t1.join();
    t2.join();

    cout << sum << endl;
}
```

위와 같이 sum을 선언할 때 atomic으로 자료형을 감싸주면 기존 의도대로 0이 출력되는 것을 볼 수 있다. atomic으로 인해 sum에 관련된 연산은 한번에 처리하기 때문이다. 

```c++
using namespace std;
#include <thread> 
#include <atomic>

atomic<int> sum = 0;

void Add()
{
    for (int i = 0; i < 1000000; i++)
    {
        sum.fetch_add(1);
    }
}

void Sub()
{
    for (int i = 0; i < 1000000; i++)
    {
        sum.fetch_add(-1);
    }
}

int main()
{
    thread t1(Add);
    thread t2(Sub);
    t1.join();
    t2.join();

    cout << sum << endl;
}
```

++, --의 경우 이미 오버로딩이 되어있어서 그대로 사용해도 상관 없지만, 위와 같이 값을 수정해줄 수도 있다. 어셈블리어로 확인해보면 아래와 같이 수정된 것을 확인할 수 있다. 

**Before**
```
mov     eax, dword ptr [sum]
inc     eax
mov     dword ptr [sum], eax
```
**After**
```
call    std::_Atomic_integral<int, 4>::fetch_add
```

기존에는 thread가 가장 빨리 처리할 수 있는 명령어 순서대로 처리를 해주었기 때문에 비교적 빠르게 동작했다. 그러나 atomic을 사용하면 전체 명령어를 실행할 수 있을 때까지 기다렸다가 동작하게 되므로 속도가 꽤 느려지고, 병목현상이 발생할 수 있다. 따라서 꼭 필요할 때만 사용하는 것이 좋다. 

<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
