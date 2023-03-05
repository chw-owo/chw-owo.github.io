---
title: Task 05) Thread Sync - User
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Interlocked 함수**

## **1) 원자적 접근**

원자적 접근이란 어떤 스레드가 특정 리소스에 접근할 때 다른 스레가 동일 시간에 동일 리소스에 접근할 수 없는 것을 의미한다. 멀티 스레드의 동작은 여러 스레드를 빠르게 돌아가며 실행함으로써 이루어지는데, 이때 실행 단위가 코드 라인이 아니라 기계어 단위이기 때문에 예상하지 못한 동작이 생길 수 있다. 인터락 계열 함수들은 최소 실행 단위를 개발자가 제어할 수 있게 함으로써 이를 예방한다. 

## **2) InterlockedExchange**

```c++
LONG InterlockedExchangeAdd(
    PLONG volatile plAddend,
    LONG lIncement
);

LONGLONG InterlockedExchangeAdd64(
    PLONGLONG volatile pllAddend,
    LONGLONG llIncement
);

LONG InterlockedIncrement(
    PLONG plAddend
);

LONG InterlockedDecrement(
    PLONG plAddend
);
```

더하기 연산을 할 때 위 함수에 값을 저장하고 있는 변수의 주소와 얼마나 증가시킬지에 대한 값을 인자로 전달하면 덧셈 연산이 원자적으로 동작한다. 공유하는 변수값에 접근하려는 스레드는 값 수정 연산 시 반드시 이러한 함수들을 사용해야 한다. 인터락 함수는 꽤 빠르게 동작하며, 내부 동작 방식은 CPU 플랫폼마다 상이하다.

```c++
LONG InterlockedExchange(
    PLONG volatile plTarget,
    LONG lValue
);

LONGLONG InterlockedExchange64(
    PLONGLONG volatile pllTarget,
    LONGLONG llValue
);

PVOID InterlockedExchangePointer(
    PVOID* volatile ppvTarget,
    PVOID pvValue
);
```

첫번째 매개변수로 전달되는 주소가 가진 값을 두번째 매개변수 값으로 변경한다. 이는 Spinlock을 구현해야 하는 경우에 유용하게 사용할 수 있다. 

```c++
PVOID InterlockedCompareExchange(
    PLONG plDestination,
    LONG lExchange,
    LONG lComparand
);

LONGLONG InterlockedCompareExchange64(
    LONGLONG pllDestination,
    LONGLONG llExchange,
    LONGLONG llComparand
);

PVOID InterlockedCompareExchangePointer(
    PVOID* ppvDestination,
    PVOID pvExchange,
    PVOID pvComparand
);
```

이 두 함수를 이용하면 원자적으로 비교와 할당을 수행할 수 있다. 이러한 인터락 함수들은 메모리 맵 파일과 같이 공유 메모리 영역 값에 동기적으로 접근하기 위해서 사용되기도 한다.

이 외에도 윈도우 XP 이후로는 정수값, BOOL 값을 원자적으로 다루는 함수와 더불어 Interlock Single Linked List 스택도 함께 제공하고 있다.  

## **3) Spinlock**

```c++
BOOL g_fResourceInUser = FALSE;
void Func1()
{
    while(InterlockedExchange (&g_fResourceInUse, TRUE) == TRUE)
        Sleep;
    
    // 리소스에 접근하는 코드

    // 리소스에 더 이상 접근할 필요가 없을 경우
    InterlockedExchange(&g_fResourceInUse, FALSE);
}
```

Spinlock은 CPU 시간을 많이 낭비할 수 있기 때문에 주의하여 사용해야 한다. 이때 CPU는 일관된 방법으로 두 값을 비교해야 하며 다른 스레드에 의해 전역 변수 값이 제대로 변경된 경우에만 비교를 중단해야 한다. Spinlock은 사용하는 모든 스레드가 동일 우선순위 레벨에 있는 것으로 가정하기 때문에 우선 순위 동적 상승 기능이 불가능하도록 설정해야 한다. 

추가적으로, 락 변수와 락을 통해 보호하려는 데이터는 서로 다른 캐시 라인에 있는 것이 좋다. 동일한 캐시라인에 있을 경우 리소스를 사용 중인 CPU가 동일 리소스에 접근하려는 타 CPU와 경쟁하게 되는데 이것이 성능에 악영향을 주기 때문이다. 만약 단일 CPU 머신이라면 루프 회전에 많은 시간을 허비하게 되기 때문에 Spinlock을 사용하지 않는 것이 좋다. 

Spinlock은 보호된 리소스가 짧은 시간 동안만 사용될 것이라 가정한다. 따라서 일차적으로 일정 회수 (일반적으로 4000회)만큼만 spin을 수행해보고, 그 이후에도 접근이 불가능하다면 커널 모드로 전환하여 리소스가 가용해질 때까지 대기상태를 유지하는 것이 CPU 낭비를 줄일 수 있다. 이는 크리티컬 섹션의 구현 방식이기도 하다.

<br/>

# **2. Cache Line**

## **1) Cache Line**

CPU가 메모리로부터 값을 가져올 때는 Cache Line 단위로 가져오게 된다. Cache Line은 CPU에 따라 32byte, 64byte, 128byte 단위로 구성되어 각 크기만큼을 경계로 정렬되어 있으며 GetLogicalProcessorInformation 함수로 크기를 얻어올 수 있다. 보통의 경우 인접한 바이트를 자주 사용하는 경향이 있으며, 자주 쓰는 값을 불러옴으로써 메모리 버스에 대해 CPU가 추가적으로 접근하는 시간을 줄일 수 있게 한다. 

그러나 멀티 프로세스 환경에서는 오히려 이것이 성능 저하를 유발할 수 있다. 멀티 프로세스 환경에서 동일한 캐시 라인에 접근하는 경우, 특정 캐시 라인에 있는 정보를 변경했을 때 타 CPU에서는 해당 캐시 라인의 정보를 무효화하게 된다. 이후 타 CPU에서 해당 캐시 라인의 정보를 읽을 경우 정보를 변경한 CPU에서 값을 RAM으로 저장하고, 타 CPU에서 다시 캐시라인을 긁어오는 과정을 거쳐야 한다. 이러한 과정이 반복될 경우 성능을 저해하게 된다.

데이터를 캐시 라인 크기와 경계 단위로 묶어서 다룰 경우 이러한 상황을 보완할 수 있다. 적어도 하나 이상의 캐시 라인 경계로 분리된 서로 다른 메모리 블록에 각각의 CPU가 독립적으로 접근하는 것을 보장할 수 있기 때문이다. 또, 읽기 전용 데이터와 읽고 쓰는 데이터를 분리하거나 동일한 시간에 접근하는 데이터들끼리 묶어서 구성하는 등의 방법을 활용할 수 있다. 

<br/>


# **3. 고급 스레드 동기화 기법**

## **1) Interlock 함수의 한계**

소수의 기본 자료형 값에 원자적으로 접근해야 하는 경우 Interlock 계열 함수가 효과적일 수 있다. 그러나 복잡한 자료구조에 대해 원자적 접근을 수행해야 할 경우 다른 기능을 이용하는 것이 권장된다. 또 CPU 시간 낭비 때문에 주의 깊게 사용해야만 하는 Spinlock 역시 대기하는 동안 CPU 시간을 낭비하지 않을 수 있는 추가적인 메커니즘이 존재한다. 

스레드가 공유 리소스 혹은 이벤트를 기다리는 경우, 대기하고자 하는 리소스 혹은 이벤트를 나타내는 값을 매개변수로 하여 운영체제가 제공하는 대기함수를 호출할 수 있다. 리소스가 가용 상태가 되거나 이벤트가 발생하면 대기 함수는 반환되고 스레드가 스케줄 가능 상태가 된다. 만약 모든 스레드가 몇 분 동안 계속 대기 상태를 유지하게 될 경우 시스템의 전원 관리 기능에 의해 절전모드로 전환된다. 

## **2) 회피 기술**

운영체제가 동기화 객체 및 이벤트 대기 기능을 제공하지 않는 경우 자체적으로 동기화를 수행할 수 있다. 다수의 스레드에 의해 공유되는 변수의 상태를 지속적으로 poling하여 다른 스레드가 작업을 완료했는지 확인하는 것이다. 그 예제는 아래와 같다. 

```c++
volatile BOOL g_fFinishedCalculation = FALSE;
int WINAPI _tWinMain(...){

    CreateThread(..., RecalcFunc, ...);

    // 재연산이 완료될 때까지 대기
    while( !g_fFinishedCalculation)
    {
        // ...
    }

    //...
}

DWORD WINAPI RecalcFunc(PVOID pvParam)
{
    // 재연산 수행

    // 수행 완료 후
    g_fFinishedCalculation = TRUE;
    return 0;
}
```

위 예제에서 보이다시피 주 스레드는 RecalcFunc 함수가 완료되기를 기다리기 위해 대기 상태로 전환되지 않고 계속 CPU 시간을 사용하게 된다. 만약 주 스레드가 RecalcFunc 보다 우선순위가 높을 경우 BOOL 변수가 영영 TRUE로 변경되지 않을 수도 있다. 이러한 Poling 방법은 매우 간편하지만 이처럼 여러 한계점이 있기 때문에 운영체제가 동기화 기법을 지원하는 경우 절대 사용하지 않기를 권장한다. 

## **3) Critical Section**

Critical Section은 공유 리소스에 대해 배타적으로 접근해야 하는 작은 코드의 집합을 의미한다. 이는 여러줄의 코드를 원자적으로 수행하기 위한 방법으로, 현재 스레드가 Critical Section을 벗어나기 전까지는 동일 리소스에 접근하려고 하는 다른 스레드를 스케줄하지 않는다. 이를 이용한 예제는 아래와 같다. 

```c++
const int COUNT = 10;
int g_nSum = 0;
CRITICAL_SECTION g_cs;

DWORD WINAPI FirstThread(PVOID pvParam){
    
    EnterCriticalSection(&g_cs);
    g_nSum = 0;
    for(int i = 1; i <= COUNT; i++)
    {
        g_nSum += i;
    }
    LeaveCriticalSection(&g_cs);
    
    return g_nSum;
}

DWORD WINAPI SecondThread(PVOID pvParam){
    
    EnterCriticalSection(&g_cs);
    g_nSum = 0;
    for(int i = 1; i <= COUNT; i++)
    {
        g_nSum += i;
    }
    LeaveCriticalSection(&g_cs);
    
    return g_nSum;
}
```

CRITICAL_SECTION 데이터 구조체로 g_sc 변수를 할당하고 공유 리소스를 사용하는 부분을 EnterCriticalSection(&g_cs), LeaveCriticalSection(&g_cs)로 둘러싸도록 코드를 작성한다. 이는 내부적으로 인터락 함수를 사용하기 때문에 매우 빠르게 사용하기도 간편하다. 하지만 서로 다른 프로세스에 존재하는 스레드 사이의 동기화에는 사용할 수 없다는 한계가 있다. 

CRITICAL_SECTION 구조체는 클래스 정의 시 주로 private 멤버로 선언하며, 모든 스레드에서 접근 가능하도록 전역 변수로 선언하는 것이 일반적이다. 하지만 지역 변수로 선언하거나 동적으로 할당하는 것도 가능하다. 이때 공유 리소스에 접근하고자 하는 모든 스레드들이 반드시 구조체의 주소를 알고 있어야 하며, 사용 이전에 초기화가 선행되어야 한다. 

구조체 내부 멤버는 문서화 되어있지 않은데, 단순 데이터 구조체이므로 윈도우 헤더파일을 통해 확인할 수 있다. 그러나 멤버를 직접 참조하는 것은 좋지 않으며 항상 적절한 윈도우 함수들을 사용해 접근해야 한다. CRITICAL_SECTION 구조체에 접근할 수 있는 함수들은 아래와 같다. 

```c++
VOID InitializeCriticalSection(PCRITICAL_SECTION pcs);
```

이는 구조체를 인자로 받아 해당 구조체의 멤버를 초기화한다. EnterCriticalSection 함수 호출 이전에 반드시 호출되어야 한다. 

```c++
VOID DeleteCriticalSection(PCRITICAL_SECTION pcs);
```

더 이상 공유 리소스를 사용할 필요가 없을 경우 위 함수를 호출하여 CRITICAL_SECTION을 삭제해야 한다. 이 함수는 구조체 내 모든 멤버 변수를 리셋한다. 다른 스레드가 이 구조체를 사용하고 있는 중에 위 함수가 호출되지 않도록 유의해야 한다. 

```c++
VOID EnterCriticalSection(PCRITICAL_SECTION pcs);
```
공유 리소스에 접근할 경우 해당 코드 앞에서 위 함수를 호출한다. 내부적으로는 아래와 같이 동작하며, 여기서 쓰이는 멤버 변수를 체크하고 수정하는 과정들은 모두 원자적으로 수행된다. 

우선 구조체 내 멤버 변수들을 통해 어떤 스레드가 현재 공유 리소스를 사용하는지 알아낸다. 만일 공유 리소스를 사용하는 스레드가 없다면 구조체 내 멤버 변수를 갱신하여 이 함수를 호출한 스레드가 공유 자원에 대한 접근 권한을 획득했음을 설정한 후, 스레드가 계속 수행될 수 있도록 반환한다. 

만일 LeaveCriticalSection 없이 EnterCriticalSection을 연속하여 두 번 호출하는 등의 상황으로 인해 해당 스레드가 이미 공유 자원에 대한 접근 권한을 획득한 상태라면, EnterCriticalSection을 호출한 스레드가 접근 권한 획득을 위해 이 함수를 몇번 호출하였는지 멤버 변수에 기록 후 반환한다.

반면 멤버 변수 확인 결과 다른 스레드에서 이미 접근 권한을 갖고 있는 상태라면 이 함수를 호출한 스레드를 이벤트 커널 오브젝트를 이용해 대기 상태, 즉 스케줄 불가능 상태로 전환한다. 대기 상태가 될 경우 CPU 시간을 낭비하지 않게 된다. 접근 권한을 갖고 있던 스레드가 LeaveCriticalSection을 호출하면 멤버 변수를 갱신하여 대기 중인 스레드를 스케줄 가능 상태로 변경한다. 

이때 코드를 잘못 작성하면 스레드가 아주 오랫동안 대기 상태로 방치되는 일이 생긴다. 이 경우를 대비하여 EnterCriticalSection은 지정 시간이 만료되면 예외를 발생시킨다. 이 함수의 만료시간은 레지스트리 키 이하의 CriticalSectionTimeout 값에 의해 결정되는데 기본 값으로는 대략 30일 정도가 설정되어 있다.

```c++
BOOL TryEnterCriticalSection(PCRITICAL_SECTION pcs);
```
TryEnterCriticalSection는 이 함수를 호출한 스레드를 절대 대기 상태로 진입시키지 않는다. 대신 함수의 반환 값으로 접근 권한을 얻었는지 여부를 가져오게 된다. 만약 TRUE를 반환하는 경우 CRITICAL_SECTION의 멤버변수를 현재 스레드가 접근 권한을 획득한 것으로 갱신하므로 반드시 이후에 LeaveCriticalSection을 호출해주어야 한다. 이 함수를 이용하면 접근 가능 여부만을 빠르게 확인한 뒤, 접근이 불가능할 경우 스레드를 대기 상태로 변경하는 대신 다른 작업을 수행시킬 수 있다. 

```c++
VOID LeaveCriticalSection(PCRITICAL_SECTION pcs);
```
위 함수는 CRITICAL_SECTION 구조체 내 멤버 변수를 확인하고 접근 권한 획득 횟수를 1 감소시킨다. 이 값이 0보다 크면 아무런 작업도 수행하지 않고 반환하고, 이 값이 0이 되면 접근 권한 획득 스레드가 없는 것으로 멤버를 갱신한다. 이후 대기 상태로 진입한 스레드가 있는지 확인하여 이러한 스레드가 하나 이상일 경우 이 중 하나를 선택해 스케줄 가능 상태로 변경한다. 

Critical Section의 한가지 단점은 대기 상태로 변경되는 과정에서 스레드는 유저 모드에서 커널 모드로의 전환을 겪는데 이 전환 과정에서 많은 시간이 소비되는 것이다. 이에 따라 마이크로 소프트는 Critical Section에 Spinlock 매커니즘을 도입함으로서 Critical Section의 성능을 개선을 시도했다. 이는 EnterCriticalSection이 호출되면 일정 횟수동안 Spinlock을 사용해 리소스 획득을 시도하다가 실패할 경우 대기 상태로 전환하는 방법을 이용한다.

```c++
BOOL InitializeCriticalSectionAndSpinCount(
    PCRITICAL_SECTION pcs,
    DWORD dwSpinCount    
);
```

이러한 방법을 사용하기 위해서는 Critical Section 초기화 시 위 함수를 사용해야 한다. 만일 이 함수를 단일 프로세서 머신에서 호출할 경우 dwSpinCount 값은 0으로 설정되며, 그 외의 경우에는 0 - 0x00FFFFFF 범위 내 값으로 지정할 수 있다. 

```c++
DWORD SetCriticalSectionSpinCount(
    PCRITICAL_SECTION pcs,
    DWORD dwSpinCount    
);
```
SpinCount는 위 함수를 호출하여 변경할 수 있다. 최상의 성능을 위해 어떤 값이 적절한지는 프로그램의 성격에 따라 달라지기 때문에 숫자를 변경해보며 직접 시도해보는 것이 좋다. 프로세스 힙에 대한 접근을 보호하기 위해 사용하는 경우에는 대략 4000이 일반적이다. 

## **4) Critical Section과 예외 처리**

마이크로소프트는 위 함수들을 설계할 때 에러 발생 가능성을 고려하지 않았기 때문에 대부분의 반환값을 VOID로 선언하였다. 그러나 위 함수들도 내부적으로 디버깅 정보 저장을 위한 메모리 블록을 할당하기에 실패할 가능성이 있다. 메모리 할당에 실패하면 STATUS_NO_MEMORY 예외가 발생하는데 이는 SEH를 통해 확인할 수 있다. InitializeCriticalSectionAndSpinCount를 사용할 경우 실패시 FALSE를 반환하기 때문에 에러 확인이 좀 더 쉽다. 

또 다른 문제는 내부적으로 다수의 스레드가 동일 시간에 진입하려고 경쟁하는 경우 이벤트 커널 오브젝트를 사용하게 되는데 이때 오브젝트 생성에 실패하여 발생한다. 이와 같은 경쟁 상황은 매우 드물게 발생하기 때문에 자주 문제가 되진 않지만 DeleteCriticalSection 호출 시까지 이벤트 커널 오브젝트가 삭제되지 않기 때문에 사용을 마친 뒤 DeleteCriticalSection 함수를 항상 호출해야 한다. 

XP 이전 운영체제에서는 가용 메모리가 부족한 상황에서 경쟁 상태가 되면 이벤트 커널 오브젝트를 생성할 수 없게 되면서 EXCEPTION_INVALID_HANDLE 예외가 발생했다. 발생 가능성이 매우 낮기 때문에 특별한 처리를 수행하지 않는 경우가 많지만, 만약 대비하고 싶다면 크게 두가지 방법이 있다. 

첫번째는 SEH를 사용하여 예외가 발생하면 일단 리소스에 접근하지 못하게 한 후 메모리가 가용해질 때 EnterCriticalSection을 재호출하도록 핸들링 하는 것이다. 두번째는 InitializeCriticalSectionAndSpinCount를 사용해 초기화 하되 dwSpinCount 매개변수의 최상위 비트를 설정하는 것이다. 그렇게 하면 이벤트 커널 오브젝트를 초기화 시점에 미리 만들어두고, 만들 수 없을 경우 FALSE를 반환하게 된다. 

그러나 이벤트 커널 오브젝트를 미리 생성해두는 것은 시스템 리소스를 낭비하는 것이므로 이러한 방법은 경쟁 상황 발생 가능성이 높거나, 가용 메모리가 매우 적은 상황에서 수행될 가능성이 높을 경우에 사용하는 것을 권장한다. XP 이후로는 키 이벤트 오브젝트를 이용해 이벤트 커널 오브젝트를 대체하는 방식으로 이러한 상황을 예방하고 있다. 

## **5) Slim Reader-Writer Lock**

Slim Reader-Writer Lock은 리소스의 값을 읽기만 하는 Reader 스레드와 수정하는 Writer 스레드가 구분되어 있는 경우에만 사용할 수 있다. Reader는 리소스 값을 손상시키지 않기 때문에 동시에 수행되어도 무방한 반면 Writer 스레드가 접근할 경우 어떠한 Reader, Writer 스레드도 접근하면 안되는 등 배타적인 접근이 이루어져야 한다. SRWLock을 사용하면 이러한 작업을 정확하게 수행할 수 있다. 

```c++
VOID InitializeSRWLock(PSRWLOCK SRWLock);
```
우선 InitializeSRWLock을 이용해 SRWLock 구조체를 할당, 초기화 해야 한다. 이 구조체는 문서화되어 있지 않기 때문에 멤버를 직접 사용할 수 없다. 

```c++
// For Writer
VOID AcquireSRWLockExclusive(PSRWLOCK SRWLock);
VOID ReleaseSRWLockExclusive(PSRWLOCK SRWLock);

// For Reader
VOID AcquireSRWLockShared(PSRWLOCK SRWLock);
VOID ReleaseSRWLockShared(PSRWLOCK SRWLock);
```
초기화 한 이후 Writer 스레드에서는 배타적 접근 권한을 얻기 위해 AcquireSRWLockExclusive 함수를, 공유 리소스를 모두 사용한 이후에는 해제를 위해 ReleaseSRWLockExclusive 사용한다. 반면 Reader 스레드에서는 Shared 함수를 사용한다. 

SRWLOCK 오브젝트를 삭제하기 위한 함수는 없으며 이는 시스템이 자동으로 수행해준다. CRITICAL_SECTION에서의 TryEnter 기능은 지원되지 않으며 SRWLOCK 오브젝트를 반복적으로 획득할 수 없다. 

## **6) ConditionVariable**

Reader가 값을 읽어오기 위해 lock을 걸었으나 읽어올 자료가 없는 경우 Reader는 lock을 해제하고 Writer가 자료를 쓸 때까지 기다려야 한다. 반대로 Writer가 계속 자료를 생산해서 공유 리소스 저장 공간이 가득 차게 되면 이 경우 Writer가 lock을 해제하고 Reader가 값을 읽어갈 때까지 기다려야 한다. 조건 변수를 이용하여 이러한 상황을 관리할 수 있다. 

```c++
BOOL SleepConditionVariableCS(
    PCONDITION_VARIABLE pConditionVariable,
    PCRITICAL_SECTION pCriticalSection,
    DWORD dwMilliseconds
);

BOOL SleepConditionVariableSRW(
    PCONDITION_VARIABLE pConditionVariable,
    PSRWLOCK pSRWLock,
    DWORD dwMilliseconds,
    ULONG Flags
);
```

pConditionVariable는 스레드가 대기할 수 있도록 초기화 된 조건 변수 포인터를 전달하고, dwMilliseconds는 언제까지 기다릴지 전달한다. 만약 시그널 상태가 되기 전 타임아웃이 되면 FALSE를, 그렇지 않으면 TRUE를 반환한다. FALSE가 반환되면 lock을 수행하지 않으며 Critical Section을 획득하지 못한다. 

Flags의 경우 시그널 상태가 되었을 때 어떻게 lock을 수행할지 전달하는데, 0을 전달할 경우 Writer를 위한 배타적 lock을 CONDITION_VARIABLE_LOCKMODE_SHARED를 전달할 경우 Reader를 위한 공유 가능 lock을 설정한다. 

```c++
VOID WakeConditionVariable(
    PCONDITION_VARIABLE ConditionVariable
);

VOID WakeAllConditionVariable(
    PCONDITION_VARIABLE ConditionVariable
);
```
reader가 읽어올 자료가 생기거나 writer가 자료를 저장할 공간이 생긴 경우 와 같이 적절한 상황이 된다면 위 함수들을 호출하여 블로킹 된 스레드를 깨울 수 있다. WakeConditionVariable를 호출했을 때 수행을 재개한 스레드가 lock을 삭제해버리면 다른 스레드들은 깨어나지 못하게 된다. 

반면 WakeConditionVariable를 호출하되 writer가 Flags로 0을 전달, reader들이 CONDITION_VARIABLE_LOCKMODE_SHARED를 전달한다면 다수의 스레드가 동시에 깨어나기를 기다려도 아무런 문제가 되지 않는다. 

## **7) Tips**

**- 원자적으로 관리되어야 하는 오브젝트 집합 당 하나의 lock을 사용하라**

여러개의 오브젝트들이 항상 같이 사용되어 논리적으로 하나의 리소스처럼 다루어야 하는 경우가 있다. 이런 리소스들을 읽거나 쓸 때는 하나의 락을 유지하는 것이 좋다. 논리적 단일 리소스 내의 일부 리소스에만 접근하는 경우에도 이러한 락을 이용해 동기화를 수행해야 한다. 만약 모든 리소스에 대해 락을 하나만 구성하게 되면 여러개의 스레드가 서로 다른 리소스에 동시에 접근할 수 없기 때문에 확장성이 결여될 수 있다. 

**- 다수의 논리적 리소스들에 동시 접근하는 방법**

다수의 논리적 리소스에 동시에 접근해야 할 경우, 이 다수의 리소스들에 대해 원자적으로 락을 설정해야 한다. 이때 리소스들에 락을 설정하는 순서, 즉 EnterCriticalSection 호출 순서를 항상 동일하게 유지해야 데드락을 피할 수 있다. LeaveCriticalSection은 스레드를 대기 상태로 변경하지 않기 때문에 이의 호출 순서는 문제되지 않는다.

**- 락을 장시간 점유하지 마라**

락을 너무 오래 점유하면 다른 스레드들이 대기 상태에 머물며 성능에 안좋은 영향을 끼칠 수 있다. 따라서 크리티컬 섹션 내에서는 최소한의 시간만 머무르도록 설계해야 한다. 예를 들어 SendMessage(..., WM_SOMEMSG, ...)와 같이 윈도우 프로시저가 처리하는데 얼마만큼의 시간이 소요될지 예측하기 어려운 것은 되도록 크리티컬 섹션 내에 포함하지 않는 것이 좋다. 

<br/>

# **4. 동기화 메커니즘에 따른 성능 비교**

## **1) volatile 변수 사용**

volatile 변수를 읽는 동작은 동기화를 필요로 하지 않기 때문에 가장 빠르게 동작한며 스레드 수가 늘어도 소요 시간이 거의 달라지지 않는다. 반면 쓰는 동작은 CPU들 사이 각 캐시를 일관되게 유지하기 위해 상호 통신을 수행해야 하기 때문에 스레드가 늘어날 수록 시간이 더 오래 걸리게 된다. 

## **2) Interlocked Increment**

이 작업의 경우 CPU가 배타적으로 메모리에 접근하도록 lock을 설정하기 때문에 volatile을 이용한 읽기/쓰기보다 느리게 동작한다. 캐시 일관성을 유지하기 위해 데이터들이 계속 상호 전달되어야 하기 때문이다. 

## **3) Critical Section**

연산 수행 구간 앞뒤로 Enter, Leave 과정이 추가 수행되기 때문에 앞선 방법보다 더욱 느리게 동작한다. 특히 Critical Section에 대한 경쟁이 발생하면 성능은 더욱 나빠지는데, 스레드 간에 경쟁 상태가 유발되며 컨텍스트 스위칭 횟수가 증가할 확률이 높아지기 때문이다. 

## **4) SRWLock**

읽기 동작이 쓰기 동작에 비해 상대적으로 더 좋은 결과가 나온다. 이 역시 필드 내 모든 값을 CPU 캐시에서 일관되게 유지해야 하기 때문에 성능이 좋지 않게 나온다. 하지만 읽기만 하는 코드가 많을 경우 Crtical Section을 쓰는 것보다 Reader - Writer를 분리하여 수행하는 것이 조금 더 좋은 성능을 보인다. 

## **5) Mutex**

뮤텍스를 사용하면 테스트를 반복할 때마다 뮤텍스의 소유와 해제가 반복된다. 그 과정에서 유저 모드 - 커널 모드의 전환이 계속 발생하며 경쟁을 유발할 수도 있기 때문에 가장 좋지 않은 성능 결과가 나온다. 

어플리케이션의 성능을 높이고 싶다면 가장 먼저 공유 리소스 사용을 최소화 해야 한다. 그 후 Interlock API > SRWLock > Critical Section 순으로 사용을 검토한다. 커널 오브젝트를 이용한 동기화는 이러한 방식이 구현하려는 상황에 전혀 적합하지 않은 경우, 마지막으로 사용하는 것이 좋다.  

<br/>

# **5. Queue 예제 애플리케이션**

## **1) 개요**
아래 예제는 한개의 SRWLock과 두개의 조건변수를 이용해 요청 Queue를 제어한다. 요청 Queue를 초기화하는 동안 4개의 Client Thread (Writer)와 2개의 Server Thread(Reader)를 생성한다. 

각 Client는 Queue에 요청을 삽입하고 일정 시간동안 Sleep한 후 다시 요청을 삽입하길 반복하며 그 과정에서 Client 리스트 박스의 내용이 갱신된다. 두 개의 Server는 요청 번호가 짝수인 경우 0번 스레드가, 홀수인 경우 1번 스레드가 각각 처리한다. 삽입된 요청이 없을 경우 Sleep 상태를 유지하다가 요청이 들어올 경우 수행을 재개하며, 요청을 처리한 이후엔 Client에게 새로운 요청을 삽입할 수 있음을 알려준다. 

## **2) Queue**

```c++
class CQueue
{
public: 
    struct ELEMENT {
        int m_nThreadNum;
        int m_nRequestNum;
    };
    typedef ELEMENT* PELEMENT;

private:
    struct INNER_ELEMENT {
        int     m_nStamp;
        ELEMENT m_element;
    };
    typedef INNER_ELEMENT* PINNER_ELEMENT;

private:
    PINNER_ELEMENT  m_pElements;
    int             m_nMaxElements;
    int             m_nCurrentStamp;

private:
    int GetFreeSlot();
    int GetNextSlot(int nThreadNum);

public:
    CQueue(int nMaxElements);
    ~CQueue;
    BOOL IsFull();
    BOOL IsEmpty(int nThreadNum);
    void AddElement(ELEMENT e);
    BOOL GetNewElement(int nThreadNum, ELEMENT& e);
};
```

ELEMENT 구조체는 큐에 삽입될 데이터 항목 형태를 정의한다. 이 예제에서는 Client가 자신의 스레드 번호와 요청 번호를 항목에 설정하여, 서버가 요청을 처리한 후 설정된 정보들을 리스트 박스에 출력하도록 할 것이다. INNER_ELEMENT의 m_nStamp는 항목의 입력 순서를 추적하는 필드로 큐에 항목이 추가될 때마다 매번 증가된다. 

m_pElements는 다수의 Client/Server의 접근으로부터 보호되어야 한다. m_nMaxElements는 CQueue 타입 객체가 생성될 때 배열 크기를 나타내는 값으로 초기화되고, m_nCurrentStamp는 새로운 항목이 큐에 추가될 때마다 증가한다. 

GetFreeSlot은 m_nStamp 값이 0인 배열의 인덱스 값을 가져오며 만일 해당하는 항목이 없을 경우 -1이 반환된다. 

```c++
int CQueue::GetFreeSlot()
{
    for(int cur = 0; cur < m_nMaxElements; cur++){
        if(m_pElements[cur].m_nStamp == 0)
            return(cur);
    }
    return (-1);
}
```

GetNextSlot은 m_pElement가 가리키는 INNER_ELEMENT 배열에서 m_nStamp값이 0이 아닌 가장 작은 수를 가진 INNER_ELEMENT의 배열 인덱스를 가져오고, 모든 항목이 이미 처리되었다면 -1을 반환한다. 이를 통해 가장 오래 전에 삽입된 요청을 가져오는 역할을 수행한다. 

```c++
int CQueue::GetNextSlot(int nThreadNum)
{
    int firstSlot = -1;
    int firstStamp = m_nCurrentStamp + 1;

    for(int cur = 0; cur < m_nMaxElements; cur++){

        if( (m_pElements[cur].m_nStamp != 0) &&
            ((m_pElements[cur].m_element.m_nRequestNum % 2) == nThreadNum) &&
            (m_pElements[cur].m_nStamp < firstStamp ) )
            {
                firstStamp = m_pElements[cur].m_nStamp;
                firstSlot = cur;
            }    
    }
    return (firstSlot);
}
```

AddElement는 Client가 Queue에 요청을 삽입할 떄 사용하는 함수이다. m_pElement가 가리키는 배열 내에 빈 slot이 있을 경우 매개변수로 전달된 ELEMENT를 저장하고 마지막 삽입된 요청의 Stamp 값을 1 증가시켜 m_nStamp에 할당한다. 

```c++
void CQueue::AddElement(ELEMENT e){
    
    int nFreeSlot = GetFreeSlot();
    if( nFreeSlot == -1)
        return;

    m_pElements[nFreeSlot].m_element = e;
    m_pElements[nFreeSlot].m_nStamp = ++m_nCurrentStamp;
}
```

GetNewElement 함수는 Server는 삽입된 요청을 처리할 때 사용하는 함수이다. Server측에서는 이 함수와 스레드 번호와 ELEMENT 변수를 인자로 전달하여 큐로부터 요청을 가져올 것이다. Queue에 처리되지 않은 요청이 있는 경우 GetNewElement는 처리되지 않은 요청을 e 변수에 복사하고 m_nStamp 값을 0으로 설정ㅎㅏ여 이 요청이 처리되었다고 표시한다. 

```c++
BOOL CQueue::GetNewElement(int nThreadNum, ELEMENT& e){

    int nNewSlot = GetNextSlot(nThreadNum);
    if(nNewSlot == -1)
        return(FALSE);

    e = m_pElements[nNewSlot].m_element;
    m_pElements[nFreeSlot].m_nStamp = 0;
    return(TRUE);
}
```

## **3) Client - Server**

**전역 변수**
```c++
// 동기화를 위한 전역 변수
CQueue g_q(10);
SRWLOCK g_srwLock;
CONDITION_VARIABLE g_cvReadyToConsume;
CONDITION_VARIABLE g_cvReadyToProduce;

// 윈도우가 닫히거나 Stop이 눌렸을 때 루프 종료를 위한 변수
BOOL g_fShutdown; 
```
**Client**
```c++
DWORD WINAPI WriterThread(PVOID pvParam){

    int nThreadNum = PtrToUlong(pvParam);
    HWND hWndLB = GetDlgItem(g_hWnd, IDC_CLIENTS);

    for(int nRequestNum = 1; !g_fShutdown; nRequestNum++){

        CQueue::ELEMENT e = { nThreadNum, nRequestNum };

        // 공유 리소스에 값을 쓰기 위해 exclusive lock을 요청한다.
        AcquireSRWLockExclusive(&g_srwLock);

        // 큐에 빈공간이 없다면 reader가 큐를 비워줄 때까지 기다린다.
        if(g_q.isFull() & !g_fShutdown)
        {

            AddText(hWndLB, TEST("[%d] Queue is full: impossible to add %d"),
                nThreadNum, nRequestNum);
            SleepConditionVariableSRW(&g_cvReadyToProduce,
                &g_srwLock, INFINITE, 0);
        }

        
        if(g_fShutdown)
        {
            // shutdown이 입력된 경우, 
            // lock을 해제하고 다른 writer에게 종료할 것을 알려준다.
            AddText(hWndLB, TEST("[%d] Bye bye!"), nThreadNum);
            ReleaseSRWLockExclusive(&g_srwLock);
            WakeAllConditionVariable(&g_cvReadyToProduce);
            return 0;
        } 
        else 
        {
            // queue에 element를 삽입하고 lock을 해제한다.
            g_q.AddElement(e);
            AddText(hWndLB, TEST("[%d] Adding %d"), nThreadNum, nRequestNum);
            ReleaseSRWLockExclusive(&g_srwLock);
            WakeAllConditionVariable(&g_cvReadyToConsume);
            Sleep(1500);
        }
    }

    AddText(hWndLB, TEST("[%d] Bye bye!"), nThreadNum);
    return 0;
}
```

**Server**

동일한 콜백함수를 이용하여 두개의 서버 스레드를 수행한다. 각각 ConsumeElement 함수를 호출하여 홀수 / 짝수 번째 요청을 처리한다. 

```c++
BOOL ConsumeElement(int nThreadNum, int nRequestNum, HWND hWndLB)
{
    AcquireSRWLockShared(&g_srwLock);

    // 큐가 비어있을 경우 조건 변수를 이용해 기다린다.
    while(g_q.IsEmpty(nThreadNum) && !g_fShutdown)
    {
        AddText(hWndLB, TEST("[%d] Nothing to process"), nThreadNum);
        SleepConditionVariableSRW(&g_cvReadyToConsume, &g_srwLock,
            INFINITE, CONDITION_VARIABLE_LOCKMODE_SHARED);
    }

    // 중지된 경우 lock을 반환한다.
    if(g_fShutdown)
    {
            AddText(hWndLB, TEST("[%d] Bye bye!"), nThreadNum);
            ReleaseSRWLockShared(&g_srwLock);
            WakeConditionVariable(&g_cvReadyToConsume);
            return FALSE;
    }

    // 새로운 요청을 가져온다
    CQueue::ELEMENT e;
    g_q.GetNewElement(nThreadNum, e);
    ReleaseSRWLockShared(&g_srwLock);
    AddText(hWndLB, TEST("[%d] Processing %d: %d"),
                nThreadNum, e.m_nThreadNum, e.m_nRequestNum);
    WakeConditionVariable(&g_cvReadyToProduce);
    return TRUE;
}

DWORD WINAPI ReaderThread(PVOID pvParam)
{
    int nThreadNum = PtrToUlong(pvParam);
    HWND hWndLB = GetDlgItem(g_hWnd, IDC_SERVERS);

    for(int nRequestNum = 1; !g_fShutdown; nRequestNum++){
        if(!ConsumeElement(nThreadNum, nRequestNum, hWndLB))
            return 0;
        Sleep(2500);
    }

    AddText(hWndLB, TEST("[%d] Bye bye!"), nThreadNum);
    return 0;
}
```

## **4) DeadLock**

프로세스를 정지시키는 코드는 아래와 같다. 

```c++
void StopProcessing(){

    if(!g_fShutdown)
    {
        InterlockedExchangePointer((PLONG*) &g_fShutdown, (LONG) TRUE);

        WakeAllConditionVariable(&g_cvReaderToConsume);
        WakeAllConditionVariable(&g_cvReaderToProduce);

        WaitForMultipleObjects(g_nNumThreads, g_hThreads, TRUE, INFINITE);

        while(g_nNumThreads--)
            CloseHandle(g_hTreads[g_nNumThreads]);

        AddText(GetDlgItem(g_hWnd, IDC_SERVERS), TEXT("====="));
        AddText(GetDlgItem(g_hWnd, IDC_CLIENTS), TEXT("====="));
    }
}
```

g_fShutdown이 TRUE로 설정되면 WakeAll 함수를 호출하여 두개의 조건 변수를 시그널 상태로 만들고, 이후 핸들값을 가진 배열을 인자로 WaitForObject 함수를 호출한다. 해당 함수가 반환되면 인자로 전달했던 핸들을 모두 삭제하고 종료 메세지를 띄운다. 

Client, Server 측면에서 보면 SleepConditionVariableSRW를 호출하여 블로킹 되었던 스레드들이 WakeAll 함수를 통해 수행을 재개하게 된다. 이 스레드들은 g_fShutdown이 TRUE이므로 bye bye를 띄우고 종료될 것이다. 그러나 스레드들이 메세지를 추가하려고 시도하는 부분에서 데드락이 발생한다. 

WM_COMMAND 메세지를 처리하는 과정에서 StopProcessing 함수를 호출하게 되면 사용자 인터페이스를 처리하는 스레드는 WaitForMultipleObjects 내부에서 블로킹 된다. 그런데 Client, Server는 이 상태에서 새로운 항목을 추가하기 위해 ListBox_SetCurSel, ListBox_AddString을 호출한다. 그럼 사용자 스레드는 블로킹 되어 있기 때문에 응답하지 못할테고 이로 인해 데드락이 발생할 것이다. 

이를 해결하기 위해서는 Stop 메시지 핸들러가 호출되었을 때 데드락 위험을 피하기 위해 StopProcessing 함수를 호출하는 새로운 스레드를 생성하여 이벤트 핸들러가 빨리 반환될 수 있도록 해야 한다. 그 예제는 아래와 같다. 

```c++
DWORD WINAPI StoppingThread(PVOID pvParam)
{
    StopProcessing();
    return 0;
}

void Dlg_OnCommand(HWND hWnd, int id, HWND hWndCtl, UINT codeNotify)
{
    switch(id){
        case IDCANCEL:
            EndDialog(hWnd, id);
            break;
        case IDC_BTN_STOP:
            //StopProcessing이 UI 함수에 의해 호출되지 않으므로 데드락 X
            DWORD dwThreadID;
            CloseHandle(chBEGINTHREADEX(NULL, 0, StoppingThread,
                NULL, 0, &dwThreadID));
            Button_Enable(hWndCtl, FALSE);
    }
    break;
}
```

이처럼 공유 리소스를 동기적으로 접근하는 것과 같이 내부적으로 블로킹을 유발할 수 있는 스레드들이 다수 존재한다. 사용자 인터페이스와 관련된 메시지의 동시 처리 역시 한 예시이며, 이런 기능을 사용할 경우 데드락에 쉽게 노출될 수 있음에 유의해야 한다. 