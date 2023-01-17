---
title: Multithread 3) Lock
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---
## **1. Lock**

**1) Lock이 필요한 이유**

바로 이전의 Atomic 포스트에서, 멀티스레드 환경에서 명령어의 순서가 보장되아 생기는 버그를 예방하기 위해 atomic을 사용했다. lock 역시 이러한 상황을 예방하기 위한 동기화 기법 중 하나이다. 예시 코드로 lock이 필요한 상황을 살펴보자.

```c++
using namespace std;
#include <thread> 
#include <atomic>

vector<int32> v;

void Push()
{
    for (int32 i = 0; i < 1000000; i++)
    {
        v.push_back(i);
    }
}

int main()
{
    thread t1(Push);
    thread t2(Push);
    t1.join();
    t2.join();

    cout << v.size() << endl;
}
```
위 코드를 보면 v.push_back이 2000000번 실행되니 결과로 2000000이 출력될 것이라 예상할 수 있다. 그러나 실제로 실행해보면 crash가 난다. vector는 데이터를 넣을 때 1) capacity를 잡고 2) 하나씩 값을 넣다가 3) capcity가 가득 차면 메모리 위치를 옮긴 뒤 4) 기존 영역을 delete하는 방식으로 동작한다. 이 과정이 한 명령어로 실행된다면 문제가 없겠지만, 실제로는 여러 줄의 기계어 코드로 실행되며 멀티스레드에서는 그 실행 순서가 보장되지 않는다. 따라서 메모리를 옮겨주는 과정에서 이미 delete한 영역을 두번 delete하는 것과 같은 크리티컬한 문제가 생길 수 있다. 

```c++
int main()
{
    v.reserve(2000000);

    thread t1(Push);
    thread t2(Push);
    t1.join();
    t2.join();

    cout << v.size() << endl;
}
```
위와 같이 reserve를 통해 이중 free를 예방해주면 crash는 해결되지만, 위와 동일한 이유로 일부 데이터를 분실하는 결과가 나올 수 있다.

기본적으로 STL 자료형들은 멀티스레드 환경에서 공유되는 데이터로 활용했을 때 문제가 생길 가능성이 높다. 이런 문제는 atomic으로 해결할 수 없다. atomic은 mov, inc와 같은 기본적인 명령어들을 하나로 묶어서 처리해주는 것이지, STL 자료형에서 사용되는 복잡한 명령어들까지 묶어서 처리해주지는 못한다. 이런 상황에서 사용할 수 있는 게 lock이다. 

<br/>

**2) Lock 사용법**

```c++
#include <mutex>

vector<int32> v;
mutex m;

void Push()
{
    for (int32 i = 0; i < 1000000; i++)
    {
        m.lock();
        v.push_back(i);
        m.unlock();
    }
}
```
mutex는 mutual exclusive(상호배타적인)의 약자이다. 위와 같이 방해 받으면 안되는 영역 앞뒤로 mutex.lock(), mutex.unlock()을 사용해주면 lock을 걸 수 있다. 그러면 해당 영역의 명령어들이 실행되는 동안은 다른 스레드들이 일시정지 하고 잠시 싱글스레드처럼 동작하게 된다. 단, 이 역시 대기가 걸리기 때문에 atomic에서와 마찬가지로 속도가 다소 느려지게 된다. 만약 unlock()을 제대로 해주지 않으면 계속 해당 스레드만 싱글스레드로 동작하게 되기 때문에 break, 분기문 등에 걸려서 unlock을 하지 않는 일이 생기지 않도록 유의해야 한다. 


```c++
m1.lock();
...
m2.lock();
v.push_back(i);
m2.unlock();
...
m1.unlock();
```

참고로 mutex는 일반적으로 위처럼 재귀적으로 (다중으로) 사용할 수 없다. 이렇게 사용하는 것은 별도의 문법이 존재한다. 

<br/>


## **2. DeadLock**

Deadlock은 잘못된 자원 관리로 인해 둘 이상의 프로세스가 함께 퍼지는 현상을 의미한다. 만약 lock을 걸고 unlock을 하지 않을 경우 해당 스레드가 다른 스레드 (main 스레드 포함)를 막아버리면서 Deadlock이 발생하게 된다. 코드가 복잡해질 경우, 수동으로 lock, unlock을 관리하면 실수가 생길 수 있으므로 아래와 같은 방법들을 사용하길 권장한다. 

**1) Lock Guard**

```c++
void Push()
{
    for (int32 i = 0; i < 1000000; i++)
    {
        std::lock_guard<std::mutex> lockGuard(m);
        v.push_back(i);
    }
}
```
lock_guard의 동작 원리를 알기 위해 아주 간단하게 class로 구현해보면, 그 구조는 아래와 같다. 

```c++
template<typename T>
class LockGuard
{
public:
    LockGuard(T& m)
    {
        _mutex = &m;
        _mutex -> lock();
    }

    LockGuard(T& m)
    {
        _mutex -> unlock();
    }

private:
    T* _mutex;
}

void Push()
{
    for (int32 i = 0; i < 1000000; i++)
    {
        LockGuard<std::mutex> lockGuard(m);
        v.push_back(i);
    }
}
```
이렇게 사용할 경우, Push() 함수가 종료되면 지역 변수로 선언되었던 lock_guard 역시 소멸하게 된다. 이때 소멸자가 자동으로 호출되며 mutex.unlock()이 호출된다. 위와 같이 사용할 경우 객체를 만드는 과정에서 성능에 약간 영향이 갈 순 있지만 더 안전한 프로그래밍이 가능해진다. 

<br/>

**2) Unique Lock**

lock_guard와 비슷한 역할을 하는 것 중에 Unique Lock이라는 게 존재한다. 

```c++
void Push()
{
    for (int32 i = 0; i < 1000000; i++)
    {
        std::unique_lock<std::mutex> uniqueLock(m);
        v.push_back(i);
    }
}
```
위와 같이 사용하면 lock_guard와 동일한 기능으로 사용할 수 있다. 

```c++
void Push()
{
    for (int32 i = 0; i < 1000000; i++)
    {
        std::unique_lock<std::mutex> uniqueLock(m, std::defer_lock);
        
        ...

        uniqueLock.lock();
        v.push_back(i);
    }
}
```
위와 같이 std::defer_lock을 사용할 경우, lock 시점을 지정해줄 수 있다. 이 기능 역시 성능에 영향을 주기 때문에 간단한 경우라면 lock_guard를, 꼭 lock() 시점을 분리해야 한다면 unique_lock을 사용하는 것이 좋다. 

<br/>

**3)**

<br/>

## **3. SpinLock**

<br/>

## **4. Lock 구현 이론**

<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
