---
title: Task 06) Thread Sync - Kernel
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 대기 함수**

## **1) 커널 오브젝트 동기화**

유저 모드 동기화는 빠르다는 장점이 있지만 복잡한 작업을 수행하기에는 한계가 많다. 우선 단일 프로세스 내 스레드들 사이에서만 동기화를 수행할 수 있으며, 대기 시간을 설정할 수 없기 때문에 데드락 상태에 빠지기 쉽다. 이를 보완하기 위해 커널 오브젝트를 이용해 스레드 동기화를 진행할 수 있다. 이 경우 커널모드로의 전환이 필요하기 때문에 성능이 좋지 못하지만 보다 복잡한 작업을 수행할 수 있다. 

프로세스, 스레드, 잡과 같은 커널 오브젝트 뿐 아니라 대부분의 커널 오브젝트는 signal/non-signal 상태를 지원하기 때문에 동기화를 위해 사용될 수 있다. 예를 들어 프로세스와 시그널은 non-signal 상태로 생성되어서, 종료되는 시점에 운영체제에 의해 자동으로 signal 상태가 되는데 이를 체크할 경우 해당 오브젝트가 종료되었음을 알 수 있고 이를 바탕으로 대기 중인 스레드의 수행을 재개하도록 요청할 수 있다. 

signal/non-signal 상태에 대한 규칙은 오브젝트 타입 별로 상이하며, 마이크로소프트는 이러한 작업을 손쉽게 할 수 있는 함수들을 제공한다. 가장 대표적인 것이 대기함수이다. 대기함수를 호출하면 인자로 전달한 커널 오브젝트가 signal 상태가 될 때까지 해당 함수를 호출한 스레드를 대기 상태로 유지한다. 

## **2) WaitForObject 함수**

```c++
DWORD WaitForSingleObject(
    HANDLE hObject,
    DWORD dwMilliseconds
);
```

WaitForSingleObject는 가장 흔하게 쓰이는 대기 함수이다. 잘못된 인자를 전달할 경우 WAIT_FAILED를, 대기하던 오브젝트가 signal이 되었다면 WAIT_OBJECT_0을, 그 전에 타임아웃이 발생하였다면 WAIT_TIMEOUT을 반환한다. 

```c++
DWORD WaitForMultipleObjects(
    DWORD dwCount,
    CONST HANDLE* phObjects,
    BOOL bWaitAll,
    DWORD dwMilliseconds
);
```
WaitForMultipleObjects는 여러개의 커널 오브젝트를 기다릴 떄 사용하는 함수로, dwCount는 검사해야 하는 커널 오브젝트의 개수를 전달한다. bWaitAll로 TRUE를 전달할 경우 모든 오브젝트들이 signal 상태가 될 때까지 기다린다.

만일 bWaitAll을 FALSE로 전달할 경우 하나만이라도 signal이 되면 대기 상태에서 빠져나오게 된다. 이떄 반환 값은 WAIT_OBJECT_0과 WAIT_OBJECT + dwCount - 1 사이의 값이 된다. 즉, 반환 값에서 WAIT_OBJECT_0을 뺀 값이 signal된 오브젝트가 저장된 배열의 인덱스라고 볼 수 있다. 

이러한 WaitForMultipleObjects 함수는 모든 오브젝트 상태를 확인하는 작업부터 이후 설명할 성공적 대기로 인한 오브젝트 상태 변경 작업까지 모두 원자적으로 수행해주기 때문에 상당히 유용하다. 

## **3) 성공적 대기의 부가적인 영향**

대기 함수가 WAIT_OBJECT_0을 반환하는 경우를 두고 성공적 호출이라고 부르는데, 일부 커널 오브젝트는 대기 함수가 성공적으로 호출될 경우 오브젝트의 상태가 변경되기도 한다. 이를 두고 '성공적인 대기의 부가적인 영향'이라고 부른다. 예를 들어 자동 리셋 이벤트 커널 오브젝트 핸들을 매개변수로 대기 함수를 호출할 경우, signal이 되어서 WAIT_OBJECT_0을 반환하는 동시에, 반환 직전 이러한 영향으로 인해 non-signal 상태로 변경된다. 

이때 두개의 스레드가 동일한 방법으로 WaitForMultipleObjects의 phObjects에 두개의 자동 리셋 이벤트를, bWaitAll에 TRUE 를 인자로 넣어서 호출한다고 가정해보자. 만약 이 함수가 위에서 설명했듯이 원자적으로 수행되지 않는다면, 한 스레드가 이벤트 오브젝트가 signal임을 발견하고 이를 non-signal로 바꾸고 그 사이 다른 스레드가 또 다른 signal 상태의 이벤트 오브젝트를 바꾸는 일이 반복되면서 순환 대기로 인해 스레드들이 완전히 정지해버리게 될 것이다. 

하지만 이 함수는 원자적으로 동작하기 때문에 내부적으로 커널 오브젝트 상태를 확인하는 시점에서 다른 스레드가 오브젝트의 상태를 변경하지 못하게 된다. 따라서 실제로는 한 스레드에서 두 오브젝트 모두 signal로 변경된 것을 확인하고 스케줄 가능 상태가 됐을 때 다른 스레드는 해당 오브젝트들이 signal 상태가 되었던 것을 알고 있음에도 두 오브젝트가 다시 signal 상태가 될 때까지 대기 상태로 머물게 된다. 이렇게 함으로써 데드락을 미연에 방지할 수 있다.

다수의 스레드가 하나의 커널 오브젝트를 대기하고 있는 상황에서 어떤 순서로 스레드를 깨어나게 할 것인지에 대해서는 마이크로소프트에서 언급하지 않고 있다. 현재까지는 선입선출 방식을 대체로 따르지만 이것도 보장된 것은 아니며 추후 업데이트와 함께 바뀔 여지가 있다. 

<br/>

# **2. 커널 오브젝트**

## **1) 이벤트 커널 오브젝트**

이벤트는 Usage Count, 자동 리셋/ 수동 리셋을 판별하는 BOOL 값, Signal/Non-signal을 나타내는 BOOL 값으로 구성된다. 이는 작업이 완료되었음을 알리기 위해 사용되는 단순한 구조의 오브젝트이다. 수동 리셋의 경우 signal 상태가 되면 이 이벤트를 기대리던 모든 스레드가 동시에 스케줄 가능 상태가 되며, 자동 리셋의 경우 대기 중인 스레드 중 하나만이 스케줄 가능 상태가 된다. 

```c++
HANDLE CreateEvent(
    PSECURITY_ATTRIBUTES psa,
    BOOL bManualReset,
    BOOL bInitialState,
    PCTSTR pszName
);

HANDLE CreateEventEx(
    PSECURITY_ATTRIBUTES psa,
    PCTSTR pszName,
    DWORD dwFlags,
    DWORD dwDesiredAddress
);
```

위 함수를 통해 이벤트를 생성할 수 있으며 bManualReset에 TRUE를 전달할 경우 수동 리셋이, FALSE를 전달할 경우 자동 리셋이 된다. bInitialState은 TRUE를 전달하면 초기 상태가 시그널로, FALSE를 전달하면 논시그널로 설정이 된다.

윈도우 비스타에서는 CreateEventEx를 제공하는데 dwFlags 매개변수로 CREATE_EVENT_INITIAL_SET, CREATE_EVENT_MANUAL_SET 플래그를 통해 해당 정보를 대신 전달한다. dwDesiredAccess로는 이벤트 생성 시점에 함수 반환 값은 핸들을 통해 오브젝트에 접근하는 권한을 설정하게 된다. 

CreateEvent은 이벤트 커널 오브젝트에 대해 모든 권한을 갖는 핸들을 반환하는 반면 CreateEventEx는 접근이 적절히 제한된 핸들을 얻을 수 있기 때문에 둘 다 사용할 수 있는 환경에서는 주로 CreateEventEx를 사용하게 된다. 

```c++
HANDLE OpenEvent (
    DWORD dwDesiredAccess,
    BOOL bInherit,
    PCTSTR pszName
);
```
다른 프로세스에서 수행되는 스레드의 경우 OpenEvent를 통해 접근할 수 있으며, 이벤트 커널 오브젝트를 더 이상 사용하지 않을 때는 CloseHandle로 반환해주어야 한다. 

```c++
BOOL SetEvent(HANDLE hEvent);
BOOL ResetEvent(HANDLE hEvent);
```
이벤트가 생성되면 이벤트의 상태를 제어할 수 있는데, 이벤트를 signal 상태로 변경하기 위해서는 SetEvent를, non-signal 상태로 변경하기 위해서는 ResetEvent를 호출하면 된다. 자동으로 non-signal로 바뀌는 자동 리셋 이벤트와 달리 수동 리셋 이벤트는 ResetEvent을 통해 상태를 바꿔주어야 한다. 

```c++
BOOL PulseEvent(HANDLE hEvent);
```
PulseEvent는 이벤트를 signal 상태로 변경하였다가 곧바로 다시 non-signal로 변경한다. 수동 리셋 이벤트를 인자로 호출할 경우 이 이벤트가 시그널 상태가 되기를 기다리던 모든 스레드가 한꺼번에 스케줄 가능 상태가 되고, 자동 리셋 이벤트를 인자로 호출 할 경우 그 중 하나의 스레드가 스케줄 가능 상태가 된다. 

## **2) 대기 타이머 커널 오브젝트**

대기 타이머 커널 오브젝트는 특정 시간에 혹은 일정한 간격을 두고 자신을 signal 상태로 만드는 커널 오브젝트로서, 주로 시간에 맞추어 작업을 수행해야 되는 경우에 사용한다. 

```c++
HANDLE CreateWaitableTimer(
    PSECURITY_ATTRIBUTES psa,
    BOOL bManualReset,
    PCTSTR pszName
);

HANDLE OpenWaitableTimer(
    DWORD dwDesiredAccess,
    BOOL bInheritHandle,
    PCTSTR pszName
);
```

위 함수를 이용해 대기 타이머 커널 오브젝트를 생성하거나, 다른 프로세스에서 이미 생성한 핸들 값을 얻을 수 있다. bManualReset에 TRUE를 넣을 경우 수동 리셋 타이머가, FALSE를 넣을 경우 자동 리셋 타이머가 된다. 대기 타이머는 항상 non-signal 상태로 초기값이 설정되며, SetWaitableTimer 함수를 이용해 signal 상태가 되는 타이밍을 정할 수 있다. 

```c++
BOOL SetWaitableTimer(
    HANDLE hTimer,
    const LARGE_INTEGER *pDueTime,
    LONG lPeriod,
    PTIMERAPCROUTINE pfnCompletionRoutine,
    PVOID pvArgToCompletionRoutine,
    BOOL bResume
);

BOOL CancleWaitableTimer(HANDLE hTimer);
```
pDueTime은 시그널 상태가 되는 최초 시간을, lPeriod는 그 후 얼마의 주기로 시그널 상태를 반복할 것인지를 지정한다. 만약 SetWaitableTimer를 재호출할 때까지 타이머가 signal되지 못하게 하고 싶다면 CancleWaitableTimer를 호출하면 된다. 그러나 SetWaitableTimer만 호출해도 기존에 설정된 시그널 기준은 모두 취소되기 때문에 타이머 시그널 기준만 변경하고 싶은 경우 굳이 Cancle 함수까지 호출할 필요는 없다. 

SetWaitableTimer를 활용한 예제는 아래와 같다. 

```c++
HANDLE hTimer;
SYSTEMTIME st;
FILETIME ftLocal, ftUTC;
LARGE_INTEGET liUTC;

hTimer = CreateWaitableTimer(NULL, FALSE, NULL);
st.wYear = 2023;
st.wMonth = 1;
st.wDayOfWeek = 0;
st.wDay = 1;
st.wHour = 13;
st.wMinute = 0;
st.wSecond = 0;
st.wMillieseconds = 0;

SystemTimeToFileTime(&st, &ftLocal);
LocalFileTimeToFileTime(&ftLocal, &ftUTC);
liUTC.LowPart = ftUTC.dwLowDateTime;
liUTC.HighPart = ftUTC.dwHighDateTime;

SetWaitableTimer(hTimer, &liUTC, 6 * 60 * 60 * 1000, NULL, NULL, FALSE);
```
위 코드는 2023년 1월 1일 오후 1:00에 처음으로 시그널 상태가 되고 이후 매 6시간마다 시그널 상태가 반복되도록 대기타이머를 설정한 예제이다. 

이때 FILETIME과 LARGE_INTEGER 구조체는 구조적 모습이 동일하기 때문에 아래와 같이 전달하면 될 것처럼 보이기도 한다.
```c++
SetWaitableTimer(hTimer, (PLARGE_INTEGER) &ftUTC, 6 * 60 * 60 * 1000, NULL, NULL, FALSE);
```

그러나 FILETIME 구조체는 32bit 경계를 기준으로 정렬이 이루어지는 반면 LARGE_INTEGER는 64bit를 기준으로 정렬을 수행한다. 컴파일러는 해당 인자가 항상 64bit 기준으로 정렬되어 있을 것이라 가정하기 때문에 이렇게 넣으면 안되고 반드시 위와 같은 과정을 거쳐야 한다. 

타이머는 종종 통신 프로토콜을 구현하기 위해 사용되기도 한다. 요청별로 타이머 커널 오브젝트를 생성하는 것은 성능에 저하가 생길 수 있으니 하나의 타이머 오브젝트를 생성한 뒤 시그널 시간을 적절히 변경해가며 사용하는 것이 좋다. 그러나 일반적으로는 일일히 시간을 재설정하는 대신 Thread Pooling 함수의 일종인 CreateThreadpoolTimer와 같은 함수를 사용하게 된다. 

유저 타이머와 비교했을 때 이러한 대기 타이머는 커널 오브젝트이기 때문에 다수의 스레드에 의해 공유될 수 있고, 더 보안에 안정적이며, 더 정확하게 통지를 수신할 수 있다. 반면 유저 타이머는 SetTimer를 호출한 스레드나 윈도우를 생성한 스레드 단 하나만이 항상 유저 타이머를 처리하게 되며, 항상 가장 낮은 우선순위로 동작하기 때문에 부정확할 수 있다. 또, 유저 타이머는 비교적 리소스를 많이 사용하는 사용자 인터페이스 환경 하에서만 수행 가능하다. 

## **3) 대기 타이머와 APC 큐**

마이크로소프트는 SetWaitableTimer를 이용해 타이머가 signal 상태가 되었을 때 APC, Asynchronous Procedure Call 비동기 함수 호출의 요청을 스레드의 APC 큐에 삽입할 수 있는 방법을 제공한다. pfnCompletionRoutine, pvArgToCompletionRoutine 매개변수에 APC 루틴의 주소를 전달하면 타이머가 signal 상태가 되었을 때 APC 요청을 APC 큐에 삽입해주게 된다. 사용 예제는 아래와 같다. 

```c++
VOID APIENTRY TimerAPCRoutine (PVOID pvArgToCompletionRoutine,
    DWORD dwTimerLowValue, DWORD dwTimerHighValue) {

        // 수행하고자 하는 작업을 추가한다. 
    }
```

TimerAPCRoutine라는 이름은 변경되어도 상관 없다. 이 함수는 타이머가 signal 상태이며 SetWaitableTimer을 호출한 스레드가 alertable 상태에 있는 경우 그 스레드에 의해 호출된다. 스레드를 alertable 상태로 만들기 위해서는 SleepEX, WaitForSingleObjectEX, WaitForMultipleObjectsEX, MsgWaitForMultipleObjectsEX, SignalObjectAndWait와 가은 함수를 호출하면 된다. 

이때 스레드는 단일의 타이머 핸들에 대해 타이머 오브젝트에 대한 시그널 디개와 alertable 상태 대기를 동시에 수행해서는 안된다. 이 상황을 예제 코드로 보이자면 아래와 같다. 

```c++
HANDLE hTimer = CreateWaitableTimer(NULL, FALSE, NULL);
SetWaitableTimer(hTimer, ..., TimerAPCRoutine);
WaitForSingleObjectEX(hTimer, INFINITE, TRUE);
```

위와 같이 코드를 작성할 경우 WaitForSingleObjectEX가 signal 대기, alertable 대기와 같이 2번의 대기를 수행하는 꼴이 된다. 위와 같이 코드를 작성한 상태에서 타이머가 signal 상태가 되면 스레드가 깨어나게 되고 alertable 상태에서 벗어나기 때문에 APC 루틴이 호출되지 않는다.

실제로는 APC 요청을 처리할 때 타이머를 이용하는 경우보다는 IO Completion Port 메커니즘을 이용하는 경우가 더 일반적이다. 일정한 주기마다 Thread Pool로부터 스레드가 깨어나야 하는 기능을 구현할 때 대기 타이머는 이와 같은 기능을 제공하지 못해서, PoseQueuedCompletionStatus를 호출하는 역할의 스레드를 별도 생성해야하기 때문이다. 

## **4) 세마포어 커널 오브젝트**

세마포어는 리소스의 개수를 고려해야 하는 상황에서 주로 사용된다. 이 커널 오브젝트는 Usage Count, 최대 리소스 카운트 값, 현재 리소스 카운트 값을 갖고 있다. 세마포어는 현재 리소스 카운트가 0보다 크면 signal 상태가, 0이면 non-signal 상태가 된다. 이 값은 늘 0과 최대 리소스 카운트 사이의 값을 유지한다. 

만약 클라이언트의 요청마다 추가적인 버퍼를 할당하는 서버를 개발하고 있다고 했을 때, 동시에 최대 5명까지만 서비스를 제공하고 요청이 없을 때는 스케줄하지 않는 구조를 설계한다고 가정해보자. 이런 상황에서 최대 리소스 카운트가 5인 세마포어를 만들어서 효과적으로 사용할 수 있다. 

```c++
HANDLE CreateSemaphore (
    PSECURITY_ATTRIBUTE psa,
    LONG lInitialCount,
    LONG lMaximumCount,
    PCTSTR pszName
);

HANDLE CreateSemaphoreEx (
    PSECURITY_ATTRIBUTE psa,
    LONG lInitialCount,
    LONG lMaximumCount,
    PCTSTR pszName
    DWORD dwFlags,
    DWORD dwDesiredAccess
);

HANDLE OpenSemaphore (
    DWORD dwDesiredAccess,
    BOOL bInheritHandle,
    PCTSTR pszName
);
```

위 함수들을 이용해 세마포어를 만들거나 세마포어의 핸들을 얻을 수 있다. lMaximumCount는 부호있는 32bit 값이므로 최대 약 2147000000개의 리소스 개수를 지정할 수 있다. lInitialCount는 현재 사용가능한 리소스의 개수를 지정한다. 만약 이 값을 0으로 설정한다면 세마포어는 non-signal 상태가 되고 세마포어를 기다리는 모든 스레드들은 자동으로 대기 상태가 된다. 

스레드가 리소스 접근을 요청하기 위해 대기함수를 호출할 때는 세마포어의 핸들을 전달하면 된다. 대기 함수는 내부적으로 세마포어의 현재 리소스 카운트 값을 확인하여 이 값이 0보다 크면 값을 1 감소시키고 대기 함수를 호출한 스레드를 스케줄 가능 상태로 만든다. 만일 대기함수가 이 값이 0임을 확인하면 대기 함수를 호출한 스레드는 대기 상태로 유지된다. 

```c++
BOOL ReleaseSemaphore(
    HANDLE hSemaphore,
    LONG lReleaseCount,
    PLONG plPreviousCount
);
```
현재 리소스 카운트를 증가 시킬 때 ReleaseSemaphore를 호출한다. lReleaseCount에 값을 전달하면 그만큼 현재 리소스 카운트를 증가시키는데 일반적으로는 1을 전달한다. plPreviousCount 값을 통해서는 증가 이전의 현재 리소스 카운트 값을 반환한다. 이 값은 잘 사용되지 않으며 NULL을 대신 전달해도 무관하다. 리소스 카운트를 변경하지 않고 그 값을 가져오는 방법은 따로 존재하지 않는다.

## **5). 뮤텍스 커널 오브젝트**

뮤텍스는 스레드가 단일 리소스에 대해 배타적으로 접근할 수 있도록 해주는 오브젝트로 Usage Count, Thread ID, 반복 카운터를 가지고 있다. Thread ID는 어떤 스레드가 뮤텍스를 소유 중인지, 반복 카운터는 뮤텍스를 소유한 스레드가 몇 회 반복하여 뮤텍스를 소유하고자 했는지를 나타낸다. Thread ID가 0이면 뮤텍스는 소유되지 않은 상태를 의미하는 signal 상태가 되고, 특정 스레드에 의해 소유되면 non-signal 상태로 전환된다. 

뮤텍스는 "유저 모드 오브젝트"인 크리티컬 섹션과 동일한 방식으로 동작하는 "커널 오브젝트"이다. 따라서 크리티컬 섹션보다 느리지만, 서로 다른 프로세스에서도 접근 가능하며 리소스 접근 권한을 획득할 때 시간 제한을 지정할 수 있다는 장점이 있다. 그런 이유로 다수의 스레드가 동시에 접근하는 메모리 블록, 공유 데이터 등을 보호할 때 자주 사용된다. 

```c++
HANDLE CreateMutex(
    PSECURITY_ATTRIBUTES psa,
    BOOL bInitialOwner,
    PCTSTR pszName
);

HANDLE CreateMutexEx(
    PSECURITY_ATTRIBUTES psa,
    PCTSTR pszName
    DWORD dwFlags,
    DWORD dwDesiredAccess
);

HANDLE OpenMutex(
    DWORD dwDesiredAccess,
    BOOL bInheritHandle,
    PCTSTR pszName
);
```

위 함수들을 이용해 뮤텍스 오브젝트를 생성하고 핸들을 얻는다. bInitialOwner는 뮤텍스 초기 상태를 제어하는 용도로 사용되는데 FALSE로 설정할 경우 Thread ID와 반복 카운터 모두 0으로 설정되며 일반적으로 이 상태로 초기화한다. 만약 TRUE로 설정할 경우 함수를 호출한 스레드의 ID로 설정되며 반복 카운터는 1로 설정되며 자연히 non-signal 상태가 된다. 

공유 리소스에 접근하려는 스레드의 경우 뮤텍스 핸들을 이용해 대기함수를 호출하면 된다. 대기 함수는 내부적으로 스레드가 non-signal인 상태일 때는 쭉 대기 상태를 유지하다가, 뮤텍스를 소유한 스레드가 없어져서 signal 상태가 되면 Thread ID를 대기 함수를 호출한 스레드의 ID로 설정한 후 스레드를 스케줄 가능 상태로 만든다. 

독특한 것은 만일 Thread ID 값이 대기 함수를 호출한 스레드의 ID 값과 동일한 경우 뮤텍스 오브젝트가 non-signal 상태이더라도 이 스레드를 스케줄 가능 상태로 만들고, 뮤텍스의 반복 카운터 값을 증가시킨다. 즉, 이 값이 1을 초과하는 경우는 동일 스레드가 뮤텍스에 여러번 대기함수를 호출한 경우에만 예외적으로 발생하는 것이다. 

```c++
BOOL ReleaseMutex(HANDLE hMutex);
```
리소스에 대한 접근 권한을 획득한 스레드가 더 이상 리소스를 사용할 필요가 없어질 경우 반드시 ReleaseMutex를 호출하여 소유권을 해제해야 한다. 이는 반복 카운터 값을 1 감소시키며, 이 값이 0이 되어야 다른 스레드가 뮤텍스를 차지할 수 있게 된다. 해제할 때도 스레드의 ID를 비교하여 일치하지 않을 경우 아무런 작업을 수행하지 않고 FALSE를 반환한다. 

만일 소유권을 해제하지 않고 스레드가 갑작스레 종료될 경우, 시스템이 뮤텍스가 버려졌다고 판단하여 Thread ID와 반복 카운터를 0으로 변경한다. 이때 대기 함수는 WAIT_OBJECT_0 대신 WAIT_ABANDONED라는 값을 반환한다. 위와 같은 상황이 생긴다면 리소스가 손상되었을 가능성이 크기 때문에 이에 대한 처리를 해주는 것이 좋고, 애초에 뮤텍스를 가질 수 있는 스레드가 이처럼 종료되지 않도록 하는 것이 가장 좋다. 

뮤텍스와 크리티컬 섹션을 직접적으로 비교하자면 아래와 같다. 

|특성|뮤텍스|크리티컬 섹션|
|----|-----|------------|
|성능|느림|빠름|
|프로세스 간 사용 여부|가능|불가능|
|선언|HANDLE hmtx|CRITICAL_SECTION cs|
|초기화|CreateMutex|InitializeCriticalSection|
|삭제|CloseHandle|DeleteCriticalSection|
|무한 대기|WaitForSingleObject(hmtx, INFINITE)|EnterCriticalSection(&cs)|
|0 대기|WaitForSingleObject(hmtx, 0)|TryEnterCriticalSection(&cs)|
|임의 시간 대기|WaitForSingleObject(hmtx, dwMilliseconds)|불가능|
|해제|ReleaseMutex|LeaveCriticalSection|
|다른 커널 오브젝트와 함께 대기 가능 여부|가능(WaitForMultipleObjects 등)|불가능|

뮤텍스와 세마포어를 이용해 여러 스레드가 공유 가능한 큐를 만들 수 있다. 그 예제는 아래와 같다. 

**class CQueue**
```c++
class CQueue {
public:
    struct ELEMENT {
        int m_nThreadNum, m_nRequestNum;
    };
    typedef ELEMENT* PELEMENT;

private:
    PELEMENT m_pElements;
    int m_nMaxElements;
    HANDLE m_h[2];
    HANDLE &m_hmtxQ;
    HANDLE &m_hsemNumElements;

public:
    CQueue(int nMaxElements);
    ~CQueue();

    BOOL Append(PELEMENT pElement, DWORD dwMilliseconds);
    BOOL Remove(PELEMENT pElement, DWORD dwMilliseconds);
}
```
CQueue의 생성자에서 뮤텍스와 세마포어를 각각 생성한 뒤 그 핸들들을 m_h 배열에 저장하여 이후 WaitForMultipleObjects 함수의 인자로 사용한다. 

```c++
BOOL CQueue::Append(PELEMENT pElement, DWORD dwTimeout)
{
    BOOL fOk = FALSE;
    DWORD dw = WaitForSingleObject(m_hmtxA, dwTimeout);

    if(dw == WAIT_OBJECT_0)
    {
        LONG lPrevCount;
        fOk = ReleaseSemaphore(m_hsemNumElements, 1, &lPrevCount);
        if(fOk)
        {
            m_pElements[lPrevCount] = *pElement;
        }
        else
        {
            SetLastError(ERROR_DATABASE_FULL);
        }

        ReleaseMutex(m_hmtxQ);
    }
    else
    {
        SetLastError(ERROR_TIMEOUT);
    }

    return (fOk);
}
```
Append 함수에서는 메소드를 호출한 스레드가 큐에 대해 배타적 접근 권한을 획득할 수 있도록 m_hmtxQ 핸들을 인자로 WaitForSingleObject를 호출할 것이다. 다음으로 해당 메소드는 ReleaseSemaphore의 두번째 매개변수로 1을 전달하여 큐에 삽입된 요청의 개수를 증가시킨다. 큐에 요청이 추가되면 ReleaseMutex를 호출하여 다른 스레드도 큐에 접근할 수 있도록 할 것이다. 

```c++
BOOL CQueue::Remove(PELEMENT pElement, DWORD dwTimeout)
{
    BOOL fOk = (WaitForMultipleObjects(_countof(m_h), m_h, TRUE, dwTimeout) == WAIT_OBJECT_0);

    if(fOk)
    {
        *pElement = m_pElements[0];
        MoveMemory(&m_pElements[0], &m_pElements[1],
            sizeof(ELEMENT) * (m_nMaxElements - 1));

        ReleaseMutex(m_hmtxQ);
    }
    else
    {
        SetLastError(ERROR_TIMEOUT);
    }

    return (fOk);
}
```
Remove는 서버 스레드가 큐로부터 요청을 가져오기 위한 함수이다. 서버 스레드는 큐에 대한 배타적 접근 권한을 획득해야 하며 적어도 한개 이상의 요청이 큐에 존재해야 한다. 따라서 앞서 만든 배열을 인자로 WaitForMultipleObjects 를 호출한 후, 두 오브젝트 모두 signal 상태가 되면 서버가 수행을 재개하게 될 것이다. 마지막으로 ReleaseMutex를 호출하여 다른 스레드가 큐에 안전하게 접근할 수 있도록 한다. 

<br/>

# **3. 그 외 스레드 동기화 함수**

## **1) 비동기 장치 IO**

비동기 장치 IO란 스레드가 읽기/쓰기 동작을 수행할 때 요청한 동작을 완료할 때까지 대기하지 않고 다른 작업을 수행할 수 있게 해주는 것이다. 파일, 소켓, 통신 포트 등의 장치 오브젝트 역시 동기화 가능한 커널 오브젝트이므로 WaitForSingleObject 함수에 핸들을 전달할 수 있다. 이들은 시스템이 비동기 장치 IO를 수행하는 동안 non-signal을 유지하다가 작업이 완료되면 signal이 된다.

## **2) WaitForInputIdle**

```c++
DWORD WaitForInputIdle(
    HANDLE hProcess,
    DWORD dwMilliseconds
);
```
스레드는 WaitForInputIdle 함수를 호출하여 대기 상태로 진입할 수 있다. 이 함수를 호출하면 hProcess가 가리키는 프로세스의 첫번쨰 윈도우를 생성한 스레드가 대기 상태가 될 떄까지 함수 호출한 스레드를 대기 상태로 유지한다. 이 함수는 부모 프로세스가 자식 프로세스 초기화 전까지 대기하고 싶은 상황, 애플리케이션에 키 입력을 전송해야 할 필요가 있는 상황 등에서 유용하게 사용된다. 

## **3) MsgWaitForMultipleObjects(Ex)**

```c++
DWORD MsgWaitForMultipleObjects(
    DWORD dwCount,
    PHANDLE phObjects,
    BOOL bWaitAll,
    DWORD dwMilliseconds,
    DWORD dwWakeMask
);

DWORD MsgWaitForMultipleObjectsEx(
    DWORD dwCount,
    PHANDLE phObjects,
    DWORD dwMilliseconds,
    DWORD dwWakeMask
    DWORD dwFlags
);
```

이 함수들은 WaitForMultipleObjects와 유사하지만, 오브젝트가 signal 된 상황 외에 함수 호출 스레드가 생성한 윈도우로부터 메시지가 전달된 상황에서도 스케줄 가능 상태가 된다는 점에서 차이가 있다. 사용자 인터페이스와 관련된 작업을 수행하는 스레드라면 인터페이스가 응답하지 않는 상황을 피하기 위해 이 함수를 사용하는 것이 권장된다. 

## **4) WaitForDebugEvent**

```c++
BOOL WaitForDebugEvent(
    PDEBUG_EVENT pde,
    DWORD dwMilliseconds
);
```

디버거가 이 함수를 호출하면 디버거의 스레드는 대기 상태가 되며, 디버그 이벤트가 발생한 경우 WaitForDebugEvent가 반환된다. pde가 가리키는 구조체는 디버거의 스레드가 깨어나기 전에 시스템에 의해 채워지며 어떤 디버그 이벤트가 발생했는지에 대한 정보를 포함한다. 

## **5) SingleObjectAndWait**

```c++
DWORD SingleObjectAndWait(
    HANDLE hObjectToSignal,
    HANDLE hObjectToWaitOn,
    DWORD dwMilliseconds,
    BOOL bAlertable
);
```

이 함수는 hObjectToSignal로 뮤텍스, 세마포어, 이벤트 커널 오브젝트를 전달받으며, 각 오브젝트 타입에 맞춰 ReleaseMutex, ReleaseSemaphore, SetEvent를 호출한다. hObjectToWaitOn로는 뮤텍스, 세마포어, 이벤트, 타이머, 프로세스, 스레드, 잡, 콘솔 입력, 변경 통지 핸들을 전달할 수 있으며 다른 대기 함수와 마찬가지로 dwMilliseconds로 signal까지 얼마나 대기할지 지정한다. bAlertable은 대기 상태 동안 비동기 프로시저 호출 수행을 허락할지 여부를 전달한다. 

이 함수는 개발자들이 특정 오브젝트를 signal 상태로 만든 후 다른 오브젝트를 대기하는 식의 코드를 자주 작성하기 때문에 이를 한번에 처리해주기 위해 등장했다. 만약 이 작업을 여러개의 함수로 할 경우 각 함수 호출 시마다 유저모두 - 커널모드 전환을 수행해야 하므로 이렇게 하나의 함수로 해결하는 것이 더 효율적이다. 

또, 이 함수를 사용하면 이를 호출한 스레드가 대기 상태에 있음을 보장하기 때문에 PulseEvent와 같은 함수를 사용할 때 유용하게 활용할 수 있다. 이는 signal 상태 변경과 대기를 원자적으로 수행하기 때문에 PulseEvent가 일어났음에도 다른 방해로 인해 대기가 성공적으로 수행되지 못하는 일을 막을 수 있다. 즉, 이를 이용할 경우 PulseEvent 호출을 통한 이벤트 상태 변경이 항상 감지되게 된다. 

## **6) 대기 목록 순회 API를 이용한 데드락 감지**

윈도우 비스타에 추가된 대기 목록 순회 API를 이용하면 프로세스들 사이에서 발생한 락을 나열해주는데 이를 이용하면 무한락 및 데드락을 비교적 쉽게 발견, 분석할 수 있다. 