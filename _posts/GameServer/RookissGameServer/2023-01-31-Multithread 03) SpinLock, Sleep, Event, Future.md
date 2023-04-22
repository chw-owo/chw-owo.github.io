---
title: Multithread 03) SpinLock, Sleep, Event, Future
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Spin Lock**

## **1) Spin Lock이란?**

Spin Lock은 임계 구역에 진입이 불가능할 경우 진입이 가능해질 때까지 계속 루프를 돌며 재시도 하는 방식으로 구현된 lock을 의미한다. 따라서 Busy waiting이 발생하게 되는데, 대신 컨텍스트 스위칭에 비용이 소요되지 않는다는 이점이 있다. Spin Lock은 유저 레벨의 명령어로, 운영체제의 스케줄링 지원을 받지 않으며 컨텍스트 스위칭도 일어나지 않는다.

## **2) Spin Lock의 구현**

SpinLock을 간단하게 구현해보면 아래와 같다. 

```c++
class SpinLock
{
public:
    void lock()
    {
        while(_locked)
        {

        }

        _locked = true;
    }
    void unlock()
    {
         _locked = false;
    }

private:
    volatile bool _locked = false;
}
```

소스 코드의 논리만 따지자면 위의 SpinLock이 제대로 작동할 것 같지만 실제로는 동작하지 않는다. SpinLock을 걸은 부분에 접근하고, _locked의 값을 확인하고, 아닐 경우 _locked를 true로 바꾸는 과정에 atomic 하게 봤을 때 한번에 이루어지지 않기 때문이다. 이를 올바르게 작동하도록 수정하면 아래와 같다. 
```c++
class SpinLock
{
public:
    void lock()
    {
        bool expected = false;
        bool desired = true;

        //CAS (Compare And Swap)
        while (_locked.compare_exchange_strong(expected, desired) == false)
        {
            expected = false;
        }
        
    }
    void unlock()
    {
         _locked.store(false);
    }

private:
    atomic<bool> _locked = false;
}
```
compare_exchange_strong과 같은 CAS 명령어를 사용하면 위의 과정을 atomic 하게 한번에 처리할 수 있게 되면서 의도대로 동작하게 된다. 

이 경우, 계속 대기 상태에 있음으로써 컨텍스트 스위칭에 드는 비용을 절약할 수 있지만 대기 상태에서 계속 CPU를 사용하여 상태를 체크, 비교 하기 때문에 CPU 성능상 낭비가 생긴다. 따라서 짧은 시간 안에 진입할 수 있을 것으로 예상되어 cpu 낭비가 심하지 않는 경우에 사용하길 권장하며, 컨텍스트 스위칭이 없기 때문에 멀티 프로세서 시스템에서 사용하길 권장한다. 

<br/> 

# **2. Sleep**

## **1) Sleep란?**

프로그램에서 Sleep 함수를 호출하면 운영체제는 해당 함수를 호출한 스레드를 대기 상태로 전환한 후, 지정된 시간만큼 중지하고 나서 다음번 스케줄링 때에 불러오게 된다. Spin Lock 때와 달리 CPU가 낭비되지 않는다는 장점이 있지만 대신 컨텍스트 스위칭에 비용이 소요된다. 

## **2) Sleep의 활용**

c++의 thread library에서는 sleep_for을 통해 sleep 기능을 지원한다. 

```c++
class Sleep
{
public:
    void lock()
    {
        bool expected = false;
        bool desired = true;

        //CAS (Compare And Swap)
        while (_locked.compare_exchange_strong(expected, desired) == false)
        {
            expected = false;
            this_thread::sleep_for(0ms);
        }
        
    }
    void unlock()
    {
         _locked.store(false);
    }

private:
    atomic<bool> _locked = false;
}
```
<br/> 

# **3. Event**

## **1) Event란?**

Event 객체는 Unsignaled / Signaled 상태를 기억할 수 있는 커널 객체이다. Event 객체를 이용하면 현재 돌아가고 있는 스레드가 종료되었을 때 해당 객체를 Unsignaled 상태에서 Signaled 상태로 바꾼다. 그러면 커널 차원에서 Event를 참조했던 Sleep 상태의 스레드를 깨워서 불러온다. 커널 차원에서 추가적인 리소스가 생긴다는 점에서 불필요하게 자주 사용하면 리소스 낭비가 발생할 수 있지만, 스레드 간 순서 동기화가 중요한 상황에서는 유용하게 사용할 수 있다. 자세한 설명은 ([이벤트 동기화][1] 포스트의 내용을 참조할 수 있다. 

## **2) Event의 활용**


```c++
mutex m;
queue<int32> q;
Handle handle;
void Producer()
{
    while(true)
    {
        {
            unique_lock<mutex> lock(m);
            q.push(100);
        }

        ::SetEvent(handle);
        this_thread::sleep_for(100000ms);
    }
}

void Consumer()
{
    while(true)
    {
        ::WaitForSingleObject(handle, INFINITE);
        unique_lock<mutex> lock(m);
        if(q.empty() == false)
        {
            int32 data = q.front();
            q.pop();
            cout << data << endl;
        }
    }
}

int main()
{
    handle = ::CreateEvent(NULL, FALSE, FALSE, NULL);

    thread t1(Producer);
    thread t2(Consumer);

    t1.join();
    t2.join();

    ::CloseHandle(handle);
}
```

Producer에서 SetEvent(handle)을 호출하면 handle이 Unsignaled 상태에서 Signaled 상태로 바뀐다. Consumer은 WaitForSingleObject(handle, INFINITE) 함수를 통해 handle의 Signal을 체크하게 둔 채로 Sleep 상태로 있다가, handle의 상태가 Signaled로 바뀌는 순간 커널에 의해 깨워져서 실행할 수 있는 상태가 된다. 이때, CreateEvent의 두번째 인자로 FALSE를 넣었을 경우 Auto Set 설정이 되기 때문에 자동으로 Unsignaled로 돌아가고, TRUE일 경우 Manual Set이기 때문에 자동으로 상태가 돌아가지 않는다. 

## **3) Condition Variable**

```c++
#include <mutex>
condition_variable cv;
```
```c++
#include <condition_variable>
condition_variable_any cva;
```
c++에는 mutex 라이브러리에 포함된 요소로서 mutex에만 적용할 수 있는 condition variable과, 좀 더 포괄적으로 사용할 수 있는 condition variable_any가 있다. 둘 다 User-Level Object이지 Kernel-Level Object는 아니다. Event와 유사하게 동작하지만 반드시 Lock과 짝지어서 동작하게 된다. Lock을 잡고, 공유 변수 값을 수정하고, Lock을 풀고, 조건 변수를 통해 다른 스레드에게 통지하는 과정을 거쳐 사용된다. 이를 활용한 예제는 아래와 같다.


```c++
mutex m;
queue<int32> q;
condition_variable cv;
void Producer()
{
    while(true)
    {
        {
            unique_lock<mutex> lock(m);
            q.push(100);
        }
        cv.nofity_one();
    }
}

void Consumer()
{
    while(true)
    {
        unique_lock<mutex> lock(m);
        cv.wait(lock, []() { return q.empty() == false; });
        
        int32 data = q.front();
        q.pop();
        cout << data << endl;
        
    }
}

int main()
{
    thread t1(Producer);
    thread t2(Consumer);

    t1.join();
    t2.join();
}
```
위와 같이 구현할 경우, Lock을 잡은 뒤 인자로 받은 조건을 확인, 조건이 false일 경우 Lock을 풀어주고 다시 대기상태로 들어간다. 만약 조건이 true일 경우 빠져나와서 이후 코드를 진행하게 된다. 이전 스레드가 논리적으로 조건을 만족하는 것처럼 보이더라도, lock을 해제하는 시점과 notify하는 시점이 일치하지 않기 때문에 그 사이에 다른 스레드에 의해 값이 바뀔 수 있다. 따라서 조건을 반드시 지켜야 하는 경우 위와 같이 사용해주는 게 안전할 수 있다. 이때 lock이 하나여야 위 동작이 가능하기 때문에 unique_lock을 사용해야 의도대로 동작한다. 

<br/>

# **4. Future**

## **1) future**

```c++
std::future<int64> future = std::async(std::launch::async, Calculate);
//...
int64 sum = future.get();
```
async의 경우 첫번째 인자로 deferred, async, deferred | async 세가지 값을 넣을 수 있다. 이때 deferred를 넣을 경우 lazy evaluation, 지연해서 CPU 여유가 있을 때 실행해달라는 의미가 되고, async를 넣을 경우 별도의 스레드를 만들어서 실행해달라는 의미가 된다. 마지막 경우는 둘 중 알아서 골라 실행해달라는 의미가 된다. 

```c++
std::future<int64> future = std::async(std::launch::async, &Knight::Attack, k1);
```
위와 같이 객체의 멤버 함수를 대상으로 사용할 수도 있다.


## **2) promise**

```c++

void PromiseWorker(std::promise<string>&& promise)
{
    promise.set_value("Secret Message");
}

void TestFuture()
{
    std::promise<string> promise;
    std::future<string> future = promise.get_future();

    thread t (PromiseWorker, std::move(promise));
    future.get();
    t.join();
}

```
위와 같이 move(promise)로 promise의 소유권을 넘겨줄 경우 promise는 다른 스레드에, future는 현 스레드에서 갖고 있게 된다. 그러다가 promise의 set_value를 호출하는 순간 future에서 해당 value 값을 받아올 수 있게 된다. 

## **3) packaged task**

packaged_task도 promise와 유사하게 사용할 수 있다. 이때, packaged_task의 경우 사용할 함수 타입을 맞춰주어야 한다. 

```c++
void TaskWorker(std::packaged_task<int64(void)>&& task)
{
    task();
}

void TestFuture()
{
    std:packaged_task<int64(void)> task(Calculate);
    std:future<int64> future = task.get_future();

    thread t (TaskWorker, std::move(task));
    int64 sum = future.get();
    t.join;
}
```
promise - future, packaged_task - future 자체가 많이 쓰이는 문법은 아니지만, 여러 스레드 간에 일회성의 간단한 통신이 필요할 때 유용하게 사용할 수 있다.

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss

https://yoongrammer.tistory.com/63

https://www.sysnet.pe.kr/2/0/11065

[1]: https://chw-owo.github.io/windowssystem/%EB%A9%80%ED%8B%B0%EC%8A%A4%EB%A0%88%EB%93%9C-5)-%EC%8A%A4%EB%A0%88%EB%93%9C-%EB%8F%99%EA%B8%B0%ED%99%94-%EA%B8%B0%EB%B2%95-(2)/#2-%EC%9D%B4%EB%B2%A4%ED%8A%B8-%EA%B8%B0%EB%B0%98-%EB%8F%99%EA%B8%B0%ED%99%94