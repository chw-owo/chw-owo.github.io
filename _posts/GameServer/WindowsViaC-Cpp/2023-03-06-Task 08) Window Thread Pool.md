---
title: Task 08) Window Thread Pool
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 스레드 풀의 기본적인 활용**

## **1) 비동기 함수 호출**

스레드 풀로 비동기 함수 호출을 하려면 다음과 같은 원형의 함수를 구현해야 한다. 

```c++
VOID CALLBACK SimpleCallback(
    PTP_CALLBACK_INSTANCE pInstance,
    PVOID pvContext
);
```

스레드 풀에 의해 관리되는 스레드가 사용자 정의 함수를 수행하도록 작업 요청을 전달하려면 TrySubmitThreadpoolCallback 함수를 호출하면 된다. 

```c++
BOOL TrySubmitThreadpoolCallback(
    PTP_SIMPLE_CALLBACK pfnCallback,
    PVOID pvContext,
    PTP_CALLBACK_ENVIRON pcbe
);
```
이 함수는 내부적으로 작업 항목을 생성하여 PostQueuedCompletionStatus를 통해 스레드 풀의 큐에 삽입하고, 성공 시 TRUE를 실패 시 FALSE를 반환한다. pfnCallbackdp에는 앞서 정의한 것 같은 사용자 정의 함수를, pvContext에는 콜백 함수의 매개변수로 전달할 값을 지정한다. pcbe는 NULL을 전달해도 된다. 

이때 CreateThread 함수는 호출하지 않아도 된다. 기본 스레드 풀과 그 안의 스레드는 자동으로 생성되고, 이렇게 생성된 스레드에 의해 콜백함수가 호출된다. 이 스레드는 요청을 처리한 후 종료되는 대신 스레드 풀로 돌아가서 다른 작업 항목이 삽입될 때까지 대기한다. 이를 통해 스레드 생성, 파괴에 소요되는 CPU 시간을 절약하여 성능을 향상시킨다. 

간혹 메모리 부족 및 할당 제한 등으로 TrySubmitThreadpoolCallback 호출에 실패하는 경우도 생기는데, 이 경우 이 함수로 전달하고자 했던 내용을 작업 항목 형태로 생성하고 스레드 풀에 삽입할 수 있게 될 때까지 기다려야 한다. 이 함수는 원래 작업 항목을 내부적으로 생성하는데, 이러한 상황을 대비해 작업 항목만 생성하기 위해서는 아래 함수를 사용한다. 

```c++
VOID CALLBACK WorkCallback (
    PTR_CALLBACK_INSTANCE pInstance,
    PVOID pvContext,
    PTP_WORK pWork
);
```
```c++
PTR_WORK CreateThreadpoolWork(
    PTR_WORK_CALLBACK pfnWorkHandler,
    PVOID pvContext,
    PTP_CALLBACK_ENVIRON pcbe
);
```
위와 같은 형태로 콜백함수를 만들어 준뒤 CreateThreadpoolWork의 pfnWorkHandler에는 콜백함수의 포인터를, pvContext는 매개변수를 전달하면 된다.

```c++
VOID SubmitThreadpoolWork(PTP_WORK pWork);
```
SubmitThreadpoolWork 함수로 작업 항목을 스레드 풀의 큐에 사빕할 수 있다. 이렇게 작업을 마치고 나면 나중에 스레드 풀 내의 스레드에 의해 콜백함수가 호출될 것이며, 작업 항목이 정상적으로 삽입되었다고 확신할 수 있기 때문에 반환형은 VOID이다. 

```c++
VOID WaitForThreadpoolWorkCallbacks(
    PTP_ 
    WORK pWork,
    BOOL bCancelPendingCallbacks
);
```

삽입된 작업 항목을 취소하거나 작업 항목이 처리될 때까지 특정 스레드를 대기 상태로 두고자 한다면 WaitForThreadpoolWorkCallbacks를 호출하면 된다. pWork에 삽입했던 작업 항목을 전달하는데, 만일 그 전에 이 함수를 호출한다면 아무 동작을 하지 않고 반환된다. 

bCancelPendingCallbacks에 TRUE를 전달하면 삽입했던 작업 항목을 취소하려 하고, 이미 처리 중에 있다면 완전히 처리될 떄까지 대기하게 된다. FALSE를 전달할 경우 작업 항목이 처리된 후 스레드 풀 스레드가 다른 항목을 처리하기 위해 반환될 때까지 대기한다.  

```c++
VOID CloseThreadpoolWork(PTP_WORK pwk);
```
생성한 작업 항목이 더 이상 필요하지 않다면 CloseThreadpoolWork를 통해 해당 작업 항목을 삭제해야 한다. 

## **2) 시간 간격을 두고 함수 호출**

애플리케이션이 제공하는 기능을 특정 시간에 수행하고 싶은 경우 대기 타이머 커널 오브젝트를 사용하게 된다. 이때 서로 다른 시간 주기를 필요로 하는 작업에 대해 각각 타이머를 생성하는 것은 리소스 낭비로 이어진다. 스레드 풀 함수를 이용해 단일의 대기 타이머를 생성, 시간을 재설정할 수 있다. 

```c++
VOID CALLBACK TimeoutCallback(
    PTP_CALLBACK_INSTANCE pInstance,
    PVOID pvContext,
    PTP_TIMER pTimer
);
```
```c++
PTP_TIMER CreateThreadpoolTimer (
    PTP_TIMER_CALLBACK pfnTimerCallback,
    PVOID pvContext,
    PTP_CALLBACK_ENVIRON pcbe
);
```
우선 TimeoutCallback과 동일한 원형을 가진 사용자 정의 함수를 만들어서 CreateThreadpoolTimer에게 포인터로 전달하면 스레드 풀 타이머가 생성된다. 

```c++
VOID SetThreadpoolTimer(
    PTP_TIMER pTimer,
    PFILETIME ptfDueTime,
    DWORD msPeriod,
    DWORD msWindowLength
);
```
스레드 풀 타이머를 스레드 풀에 등록할 때 SetThreadpoolTimer 함수를 사용한다. pTimer에 CreateThreadpoolTimer의 반환 값을, pftDueTime에 최초 호출 시간을 전달하는데 이때 -1을 전달하면  바로 호출하며 그 외의 음수값을 전달하면 상대적 소요 시간을 따르게 된다. 한번 호출을 원할 경우 msPeriod를 0으로 전달하며, msWindowLength를 지정하면 그 범위 내에서 콜백함수 호출 시간이 임의로 지정되어 동일 시간 여러개의 통지가 발생하는 것을 막을 수 있다. 

타이머 설정은 한번 정하고 가급적 변경하지 않는 것이 좋지만, SetThreadpoolTimer를 호출할 때 앞서 사용한 타이머 포인터를 pTimer에 전달하여 타이머 설정을 변경할 수 있다. 또, pftDueTime에 NULL을 넣으면 해당 콜백함수를 더 이상 호출하지 않기 때문에 타이머 오브젝트를 파괴하지 않고도 타이머를 정지시킬 수 있다. 

```c++
BOOL IsThreadpoolTimerSet(PTP_TIMER pti);
```
IsThreadpoolTimerSet 함수를 통해 타이머가 현재 동작중인지 확인할 수 있다. 또 앞선 예시와 마찬가지로 WaitForThreadpoolTimerCallbacks를 통해 타이머 작업이 완료될 때까지 특정 스레드를 대기 시킬 수 있으며 CloseThreadpoolTimer를 통해 메모리로부터 타이머를 삭제할 수 있다. 이들은 WaitForThreadpoolWork, CloseThreadpoolWork와 유사하게 동작한다. 

타이머에 특정 주기를 설정하면 주기에 맞춰 작업 항목을 스레드 풀 큐에 삽입한다. 이때 풀 내에 여러개의 스레드가 존재하면 콜백함수가 동기화 문제를 유발할 가능성이 있으며, 풀에 높은 부하가 걸리면 작업 항목의 처리가 지연될 수도 있다. 즉, 스레드의 최대 개수를 적게 설정할 경우 콜백함수 호출이 지연될 수 밖에 없는 것이다. 

이런 동작이 마음에 들지 않거나 매우 긴 시간을 요하는 작업 항목이 있을 경우, 이전 콜백함수가 수행 완료되고 일정 주기가 지난 뒤에 작업 항목을 삽입하고 싶을 수 있다. 이 경우 아래와 같으 ㄴ방법을 사용할 수 있다. 

**1)** CreateThreadpoolTimer 함수 호출 부분은 그대로 두고 SetTimerpoolTimer 호출 시 msPeriod를 0으로 전달하여 timeout을 한번만 발생시킨다. 

**2)** 콜백 함수가 호출되어 작업이 완료되면 msPeriod를 다시 0으로 설정해 타이머를 재시작하도록 한다. 

**3)** 타이머를 완전히 정리할 때는 WaitForThreadpoolTimerCallbacks 마지막 인자값을 TRUE로 호출하여 이 타이머에 의해 풀 내 스레드가 추가적으로 수행되지 않도록 한다. 그 후 CloseThreadpoolTimer를 호출한다. 

## **3) 커널 오브젝트가 시그널 되면 함수 호출**

상당히 많은 애플리케이션들이 커널 오브젝트 시그널을 기다리는 목적으로 새로운 스레드를 생성하여 사용한다. 이들은 오브젝트가 시그널 상태가 되면 다른 스레드에게 그 사실을 통지하고 다시 오브젝트가 시그널 될 때까지 대기한다. 그러나 이러한 방법은 리소스 낭비를 일으킬 수 있으므로 가능하다면 콜백함수를 이용해 보완하는 것을 권장한다. 

```c++
VOID CALLBACK WaitCallback(
    PTP_CALLBACK_INSTANCE pInstance,
    PVOID Context,
    PTP_WAIT Wait,
    TP_WAIT_RESULT WaitResult
);
```
```c++
PTP_WAIT CreateThreadpoolWait(
    PTP_WAIT_CALLBACK pfnWaitCallback,
    PVOID pvContext,
    PTP_CALLBACK_ENVIRON pcbe
);
```
WaitCallback과 같은 형식의 콜백 함수를 구현한 뒤 CreateThreadpoolWait 함수를 호출하여 스레드 풀 대기 오브젝트를 생성한다. 

```c++
VOID SetThreadpoolWait(
    PTP_WAIT pWaitItem,
    HANDLE hObject,
    PFILETIME pftTimeout
);
```
그 후 SetThreadpoolWait를 호출하여 커널 오브젝트와 풀 대기 오브젝트를 연계시킨다. hObject로 커널 오브젝트를 전달하면 해당 오브젝트가 signal 상태가 되었을 때 pWaitItem을 통해 전달한 콜백함수를 호출하게 된다. pftTimeout를 통해 얼마나 기다릴지 지정할 수 있으며, 0을 전달할 경우 기다리지 않고, 음수를 전달하면 상대 시간을, 양수를 전달하면 절대 시간을, NULL을 전달하면 무한 대기를 지정할 수 있다. 

스레드 풀은 내부적으로 WaitForMultipleObjects를 호출하므로 최대 64개의 핸들만 다룰 수 있다. SetThreadpoolWait 호출시마다 인자로 전달된 핸들을 WaitForMultipleObjects의 인자로 사용하며 bWaitAll 값은 늘 FALSE이기 때문에 하나라도 signal이 되면 스레드가 깨어난다. 이때 WaitForMultipleObjects과 달리 SetThreadpoolWait는 DuplicateHandle을 사용하여 동일 커널 오브젝트를 여러번 사용할 수 있다. 

커널 오브젝트가 signal되거나 대기 시간이 만료되면 스레드 풀은 정의된 WaitCallback 함수를 호출한다. 이때 WaitResult는 왜 Waitcallback이 호출되었는지를 나타내는데, 이는 WAIT_OBJECT_0, WAIT_TIMEOUT, WAIT_ABANDONED_0의 값을 가질 수 있다. 

스레드가 콜백함수를 호출하면 핸들은 비활성화된다. 이는 동일 오브젝트가 signal 되었을 때 계속 콜백함수가 호출되려면 SetThreadpoolWait를 호출하여 재등록해야 함을 의미한다. 이때 프로세스 커널 오브젝트는 한번 signal되면 영원히 signal되므로 이런 경우엔 다른 커널 오브젝트 핸들로 다시 등록하거나 NULL 값을 전달하여 해당 핸들을 제거해야 한다. 

마지막으로 WaitForThreadpoolWaitCallbacks 함수를 호출하여 특정 오브젝트가 signal 되어 콜백함수가 반환될 때까지 대기할 수 있으며, CloseThreadpoolWait 함수를 호출해 스레드 풀 대기 오브젝트를 삭제할 수도 있다. 이들은 WaitForThreadpoolWork, CloseThreadpoolWork와 유사하게 동작한다. 

## **4) 비동기 IO 요청이 완료되면 함수 호출**

IO 컴플리션 포트를 대기하는 스레드들을 포함해 스레드 풀을 만들 수 있다. 스레드 풀을 이용하면 스레드 생성, 파괴를 개발자가 직접 수행할 필요가 없다. 대신 내부적으로 생성된 스레드들이 컴플리션 포트를 사용하기 때문에, 장치를 컴플리션 포트와 연계시키고 비동기 작업을 완료했을 떄 스레드가 어떤 함수를 호출하게 할지 지정해주어야 한다. 

```c++
VOID CALLBVACK OverlappedCompletionRoutine (
    PTP_CALLBACK_INSTANCE pInstance,
    PVOID pvContext,
    PVOID pOverlapped,
    ULONG IoResult,
    ULONG_PTR NumberOfBytesTransferred,
    PTP_IO pIo
);
```

이를 위해 먼저 위와 같은 원형의 사용자 함수를 만든다. IO 작업이 완료되면 이 함수가 호출되면서 OVERLAPPED 구조체 포인터를 인자로 받아오고, 작업의 수행 결과를 IoResult 매개변수로 받아오게 될 것이다. 작업이 성공적으로 수행되었다면 이 매개변수에 NO_ERROR를, NumberOfBytesTransferred에 송수신 바이트 수를,  pIo에 스레드 풀 IO 오브젝트를 전달받게 될 것이다. 

```c++
PTP_IO CreateThreadpoolIo(
    HANDLE hDevice,
    PTP_WIN32_IO_CALLBACK pfnIoCallback,
    PVOID pvContext,
    PTP_CALLBACK_ENVIRON pcbe
);
```
그 후 CreateThreadpoolIo를 호출하여 스레드 풀 IO 오브젝트를 생성한다. CreateFile에서 FILE_FLAG_OVERLAPPED 플래그로 생성한 핸들을 이 함수의 인자로 전달하면 장치의 핸들 값이 스레드풀 내부의 IO 컴플리션 포트와 연계된다. 

```c++
VOID StartThreadpoolIo(PTP_IO pio);
```
마지막으로 StartThreadpoolIo를 호출하면 IO 오브젝트와 스레드 풀 내부 IO 컴플리션 포트가 연계된다. ReadFile, WriteFile 호출에 앞서 이를 먼저 호출해야 하며, IO 작업 요청 전에 이 함수에서 에러가 생긴다면 앞서 만든 OverlappedCompletionRoutine는 호출되지 않는다. 

```c++
VOID CancelThreadpoolIo(PTP_IO pio);
```
IO 작업 요청 이후 콜백함수 호출을 중단하려면 CancelThreadpoolIo를 호출한다. ReadFile, WriteFile을 통한 IO 작업 요청에 실패한 경우, 즉 두 함수가 FALSE를 호출하고 GetLastError가 ERROR_RO_PENDING이 아닌 다른 값을 반환한 경우에도 반드시 CancelThreadpoolIo를 호출해주어야 한다. 

```c++
VOID CloseThreadpoolIo(PTP_IO pio);
```
장치에 대한 사용을 마치려면 CloseHandle로 장치를 닫은 뒤, 핸들과 스레드 풀의 연계성을 끊기 위해 CloseThreadpoolIo를 호출한다. 

```c++
VOID WaitForThreadpoolIoCallbacks(
    PTP_IO pio,
    BOOL bCancelPendingCallbacks
);
```
만일 요청된 작업이 완료될 때까지 다른 스레드가 대기하기를 원한다면 WairForThreadpoolIoCallbacks를 호출한다. bCancelPendingCallbacks로 TRUE를 전달하면 처리되지 않은 요청들은 모두 취소되며 완료 통지는 발생하지 않는 등 CancelThreadpoolIo를 호출한 것과 유사하게 동작한다. 

<br/>

# **2. 스레드 풀의 디테일한 사용**

## **1) 콜백 종료 동작**

스레드풀은 사용자가 작성한 콜백함수가 반환될 떄 반드시 수행해야 하는 작업을 인자로 지정할 수 있어서 편리하다. 콜백함수는 pInstance 매개변수를 갖는데, 이는 스레드가 현재 처리하고 있는 작업, 타이머, 대기 또는 IO 인스턴스 등을 대표해준다. 

이 값으로 호출할 수 있는 종료 동작 지정 함수는 아래와 같다. 

```c++
VOID LeaveCriticalSectionWhenCallbackReturns(
    PTP_CALLBACK_INSTANCE pci,
    PCRITICAL_SECTION pcs
);
```
콜백함수가 반환되면 스레드 풀이 자동으로 CRITICAL_SECTION 구조체에 대해 LeaveCriticalSection 함수를 호출한다. 

```c++
VOID ReleaseMutexWhenCallbackReturns(
    PTP_CALLBACK_INSTANCE pci, 
    HANDLE mut
);
```
콜백함수가 반환되면 스레드 풀이 자동으로 전달된 핸들에 대해  ReleaseMutex 함수를 호출한다. 

```c++
VOID ReleaseSemaphoreWhenCallbackReturns(
    PTP_CALLBACK_INSTANCE pci, 
    HANDLE sem, 
    DWORD crel
);
```
콜백함수가 반환되면 스레드 풀이 자동으로 전달된 핸들에 대해  ReleaseSemaphore 함수를 호출한다. 

```c++
VOID SetEventWhenCallbackReturns(
    PTP_CALLBACK_INSTANCE pci, 
    HANDLE evt
);
```
콜백함수가 반환되면 스레드 풀이 자동으로 전달된 핸들에 대해 SetEvent 함수를 호출한다. 

```c++
VOID FreeLibraryWhenCallbackReturns(
    PTP_CALLBACK_INSTANCE pci, 
    HMODULE mod
);
```
콜백함수가 반환되면 스레드 풀이 자동으로 HMODULE에 대해 FreeLibrary 함수를 호출하여 DLL 파일을 언로드 할 수 있도록 해준다. 콜백함수 내부에서 FreeLibrary를 호출하면 접근 위반이 되기 때문에 이 함수가 유용하게 쓰일 수 있다.

이때 콜백 인스턴스 별로 하나의 종료 동작만을 지정할 수 있으며, 동시에 시그널링을 수행할 수는 없다. 이를 시도할 경우 마지막으로 호출된 함수에서 설정한 내용이 이전 설정을 덮어쓰게 된다. 

또 pInstance에 대해 아래 함수들을 지정할 수도 있다. 

```c++
BOOL CallbackMayRunLong(
    PTP_CALLBACK_INSTANCE pci
);
```
CallbackMayRunLong는 스레드 풀이 항목을 어떻게 처리할지 알려주며, 콜백함수 처리가 오래 걸릴 것으로 예상될 때 쓰일 수 있다. 처리가 오래 걸리는 스레드가 있으면 스레드 풀에 삽입된 다른 항목들이 적절히 처리되지 못할 수 있는데, 이때 이 함수가 TRUE를 반환하면 스레드 풀이 다른 항목들을 처리할 여분의 스레드를 가지고 있음을 나타낸다. 이 경우 처리 시간이 긴 작업을 여러 항목으로 나누고, 현재 콜백함수를 호출한 스레드로는 그 중 하나의 항목만 수행하도록 하면 더 효율적으로 처리할 수 있다. 

```c++
VOID DiassociateCurrentThreadFromCallback(
    PTP_CALLBACK_INSTANCE pci
);
```
DiassociateCurrentThreadFromCallback은 스레드 풀에게 작업이 논리적으로 종료되었음을 알려준다. 이를 통해 WaitForThreadpool~ 함수들을 호출하여 콜백함수가 반환되기만을 기다리던 스레드들을 즉각 깨울 수 있다. 

## **2) 스레드 풀 커스터마이징**

CreateThreadpool~ 함수를 호출할 때 PRP_CALLBACK_ENVIRON 형의 매개변수를 전달할 수 있는데 대부분의 경우 여기에 NULL을 전달하여 기본 설정으로 작업을 수행한다. 하지만 애플리케이션 특성에 맞추어 변경하고 싶은 경우 이 인자를 통해 스레드 최소, 최대 개수를 설정하거나 다수의 스레드 풀을 각각 독립적으로 생성, 파괴할 수 있다. 

```c++
PTP_POOL CreateThreadpool(PVOID reserved);
```
새로운 스레드 풀을 생성하고 싶을 때는 CreateThreadpool 함수를 호출한다. 매개변수는 예약되어 있으므로 NULL을 전달한다. 

```c++
BOOL SetThreadpoolThreadMinimum(PTP_POOL pTheradPool, DWORD cthrdMin);
BOOL SetThreadpoolThreadMaximum(PTP_POOL pTheradPool, DWORD cthrdMost);
```
위 함수들을 통해 스레드 풀의 스레드 개수를 설정할 수 있다. 기본 스레드 풀은 1-500의 범위를 가진다. 

```c++
VOID CloseThreadpool(PTP_POOL pThreadPool);
```
커스터마이징을 수행한 풀이 더 이상 필요하지 않은 경우 CloseThreadpool로 스레드 풀을 삭제해야 한다. 이를 호출하고 나면 더 이상 해당 풀에 작업 항목을 추가할 수 없게 되고,  큐에 삽입되었으나 시작되지 않았던 항목들은 모두 취소되며, 수행 중이던 스레드는 작업을 완료한 즉시 종료된다.

추가적인 스레드 풀을 생성하고 나면 작업 항목에 적용할 수 있는 설정 정보, 환경 정보를 저장하기 위해 콜백 환경을 구성할 수 있다. 이때 WinNT.h의 TP_CALLBACK_ENVIRON 구조체가 콜백 환경을 표현하기 위해 사용된다. 구조체 각 필드에 대해 직접 내용을 변경하기보다는 적절한 함수를 호출하여 변경하기를 권장한다. 

```c++
VOID InitializeThreadpoolEnvironment(PTP_CALLBACK_ENVIRON pcbe);
VOID DestroyThreadpoolEnvironment(PTP_CALLBACK_ENVIRON pcbe);
```

위 함수들을 통해 TP_CALLBACK_ENVIRON 구조체를 초기화 하고 정리할 수 있다.

```c++
VOID SetThreadpoolCallbackPool(
    PTP_CALLBACK_ENVIRON pcbe,
    PTP_POOL pThreadPool
);
```

사용자가 생성한 스레드 풀에 작업 항목을 삽입하기 위해서는 어떤 스레드 풀에서 작업을 수행할지 지정해야 한다. TP_CALLBACK_ENVIRON 구조체의 Pool 멤버에 작업을 수행할 스레드 풀을 지정하여 SetThreadpoolCallbackPool 함수의 두번째 매개변수로 전달하면 이러한 작업을 수행할 수 있다. 이 함수를 쓰지 않을 경우 기본 스레드 풀에 의해 처리된다. 

```c++
VOID SetThreadpoolCallbacksRunsLong(PTP_CALLBACK_ENVIRON pcbe);
```
SetThreadpoolCallbacksRunsLong는 작업 항목 처리에 일반적으로 오랜 시간이 걸릴 것임을 알려주도록 콜백 환경을 변경한다. 이렇게 되면 스레드 풀은 좀 더 빠른 시간 안에 여러개의 스레드를 생성하여 각 작업 항목을 더 효율적으로 서비스 할 수 있게 된다. 

```c++
VOID SetThreadpoolCallbackLibrary(
    PTP_CALLBACK_ENVIRON pcbe,
    PVOID mod
);
```
SetThreadpoolCallbackLibrary는 스레드 풀 내 작업 항목이 존재하는 동안 특정 DLL을 프로세스 주소 공간에 로드하고 있도록 한다. 이것은 기본적으로 데드락을 유발할 수 있는 경합 상태를 제거하기 위해 존재하는 함수다. 

## **3) 스레드 풀의 삭제 그룹**

스레드 풀을 삭제하기 위해서는 삽입된 작업 항목들이 언제 처리되었는지를 알아내야 하는데, 이를 깔끔하게 처리할 수 있도록 스레드 풀은 삭제 그룹이라는 기능을 제공한다. 기본 스레드 풀은 프로세스가 수행되는 동안 계속 유지되며 프로세스 종료 시점에 OS에 의해 삭제되기 때문에 이러한 기능을 적용할 수 없다. 

```c++
PTP_CLEANUP_GROUP CreateThreadpoolCleanupGroup();
```
CreateThreadpoolCleanupGroup를 호출하여 삭제 그룹을 생성할 수 있다. 

```c++
VOID CALLBACK CleanupGroupCancelCallback(
    PVOID pvObjectContext,
    PVOID pvCleanupContext
);
```
```c++
VOID SetThreadpoolCallbackCleanupGroup(
    PTP_CALLBACK_ENVIRON pcbe,
    PTP_CLEANUP_GROUP ptpcg,
    PTP_CLEARUP_GROUP_CANCEL_CALLBACK pfng
);
```
그리고 SetThreadpoolCallbackCleanupGroup를 통해 삭제 그룹을 TP_CALLBACK_ENVIRON 구조체와 연계한다. 내부적으로 이 함수는 TP_CALLBACK_ENVIRON 구조체의 CleanupGroup, CleanupGroupCancelCallback 필드들의 값을 설정한다. pfng에는 삭제 그룹이 취소될 때 호출할 콜백함수 주소를 지정하는데 이는 CleanupGroupCancelCallback와 같은 원형으로 구현하며 NULL값을 넣어도 된다. 

마지막으로 CreateThreadpool~ 함수의 마지막 매개변수로 NULL 대신 PTP_CALLBACK_ENVIRON을 전달한다. 그러면 생성된 작업 항목이 해당 스레드 풀이 속한 삭제 그룹에 삽입되며, 처리가 완료된 후 CloseThreadpool~ 함수를 호출하면 내부적으로 삭제 그룹으로부터 작업 항목을 삭제하게 된다. 

```c++
VOID CloseThreadpoolCleanupGroupMembers(
    PTP_CLEANUP_GROUP ptpcg,
    BOOL bCancelPendingCallbacks,
    PVOID pvCleanupContext
);
```
위 함수로 사용자가 생성한 스레드 풀을 삭제할 수 있다. 이는 WaitForThreadpool~ 함수들과 매우 유사한데, 작업 그룹 내에 항목이 남아있는 한 대기한다. bCancelPendingCallbacks에 TRUE를 전달하면 처리되지 않은 항목들을 취소할 수 있으며, 수행 중인 항목의 처리를 완료하면 함수가 반환된다. 이때 항목이 취소될 때마다 지정한 콜백함수가 호출된다. FALSE를 전달할 경우 모든 작업 항목이 처리된 후 함수가 반환되며, 이 경우 콜백함수가 호출되지 않기 때문에 해당 매개변수에 NULL을 전달해도 된다. 

```c++
VOID WINAPI CloseThreadpoolCleanupGroup(PTP_CLEANUP_GROUP ptpcg);
```
모든 작업 항목이 취소 혹은 처리 되고 나면 CloseThreadpoolCleanupGroup를 호출하여 삭제 그룹이 점유하고 있던 리소스를 반납해야 한다. 이후 DestroyThreadpoolEnvironment, CloseThreadpool을 호출하면 스레드 풀을 깔끔하고 안전하게 종료할 수 있다. 