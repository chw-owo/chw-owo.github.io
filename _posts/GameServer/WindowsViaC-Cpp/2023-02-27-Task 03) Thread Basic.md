---
title: Task 03) Thread Basic
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 스레드**

## **1) 스레드란?**

프로세스는 스스로 어떤 것도 수행할 수 없으며, 단순하게 생각한다면 스레드들의 저장소로 볼 수도 있다. 스레드는 프로세스 주소 공간 내의 코드를 실제로 수행하는 주체가 된다. 스레드는 항상 프로세스의 컨텍스트 내에서 생성되어 프로세스 내에서만 살아있는다. 하나의 프로세스에 다수의 스레드가 존재할 경우 스레드들은 단일 주소 공간을 공유하며 커널 오브젝트 핸들 역시 공유하게 된다. 

프로세스의 초기화가 진행되는 동안 시스템은 주 스레드를 생성하며, 마이크로소프트 C/C++ 컴파일러로 작성한 경우 주 스레드가 C/C++ 런타임 라이브러리의 시작코드를 수행하게 된다. 이후 진입점 함수 (main 함수)를 호출하고 함수가 반환될 때까지 수행을 계속하다가 반환되면 C/C++ 런타임 라이브러리의 시작 코드가 ExitProcess를 호출하며 수행을 종료한다. 

프로세스는 자신만의 주소 공간을 가지므로 스레드에 비해 더 많은 시스템 리소스를 사용한다. 프로세스는 상당량의 정보를 시스템 내부에 저장하기 때문에 메모리를 많이 필요로 하며, .exe, .dll 파일이 주소 공간으로 로드되어야 하므로 파일 리소스 또한 필요하다. 반면 스레드는 상당히 적은 시스템 리소스를 필요로 하는데, 하나의 커널 오브젝트와 스레드 스택만을 요구한다.

스레드는 스레드는 운영체제가 스레드를 다루기 위해 사용하는 스레드 커널 오브젝트와 스레드가 코드를 수행할 때 매개변수 및 지역 변수를 저장하기 위해 사용하는 스레드 스택으로 구성된다. 이때 스레드 커널 오브젝트는 시스템이 스레드에 대한 통계 정보를 저장하는 공간이기도 하다. 따라서 시스템 내부에 저장해두는 내용도 적고 메모리도 덜 차지한다.

이처럼 스레드는 프로세스에 비해 부하가 적기 때문에 프로세스를 새로 생성하는 대신 추가적인 스레드를 생성하여 해결하는 것이 더 효율적인 경우가 많다. 반면, 반대로 여러개의 프로세스를 사용하는 것이 더 효율적인 상황 역시 존재한다. 각각의 예시는 아래와 같다.  

## **2) 스레드를 생성해야 하는 경우**

대다수의 애플리케이션은 주 스레드 하나면 충분하지만 작업 수행에 도움이 된다면 추가적인 스레드를 생성할 수 있다. 예를 들어 IO를 비롯하여 DB 정렬, 문서 출력, 파일 복사 등의 작업을 분리된 스레드로 수행하면 작업하는 도중 다른 작업을 수행할 수 있으므로 사용자 인터페이스가 더 즉각적인 응답을 보이도록 할 수 있다. 또 스레드별 전용 CPU를 할당할 수 있게 되면서 더 좋은 성능을 낼 수 있으며, 컴파일러에 버그가 있어서 한 스레드가 무한 루프에 빠지더라도 다른 스레드는 계속해서 사용할 수 있다. 

## **3) 스레드를 생성하지 말아야 하는 경우**

대부분의 어플리케이션에서 사용자 인터페이스를 위한 컴포넌트는 동일한 스레드를 사용해야 한다. 즉, 하나의 스레드를 이용해 모든 윈도우의 차일드 윈도우를 생성하는 것이 대부분의 상황에서 더 좋다. 계산 작업, IO 위주의 작업과 같이 윈도우를 만들지 않는 워커 스레드들은 별개의 스레드로 두고, 단일 사용자 인터페이스 스레드를 이러한 워커스레드보다 높은 우선순위에 둘 경우 사용자 인터페이스의 응답성을 개선할 수 있다. 

<br/>

# **2. 스레드 생성**

## **1) 스레드 함수**

모든 스레드는 수행을 시작할 진입점 함수를 반드시 가져야 한다. 주 스레드의 진입점 함수는 _tmain, _tWinMain이 되며, 그 이후의 스레드는 어떤 이름이라도 사용할 수 있다. 대신 각각의 스레드 함수는 서로 다른 이름으로 명명되어야 한다. 

```c++
DWORD WINAPI ThreadFunc(PVOID pvParam)
{
    DWORD dwResult = 0;
    // ...
    return(dwResult);
}
```
스레드 함수는 위와 같은 형태의 진입점 함수를 가져야 한다. 보이다시피 스레드 함수는 하나의 매개변수만 전달할 수 있으며, 각 매개변수의 의미는 운영체제가 아닌 사용자에 의해 정의되기 때문에 ANSI, 유니코드 버전을 각기 구성하지 않아도 된다. 

스레드 함수는 가능한 매개변수와 지역 변수만을 사용하는 것이 좋다. 정적 변수, 전역 변수에 다수의 스레드가 동시에 접근하게 되면 예상치 못한 문제가 발생할 수 있기 때문이다. 

스레드 함수는 반드시 값을 반환해야 하는데, 이는 스레드의 종료 코드가 된다. 스레드 함수가 끝나서 반환되면 스레드는 수행을 멈추고, 스레드 스택은 반환되며, 스레드 커널 오브젝트 Usage Count도 감소한다. 


## **2)  CreateThread 함수**

운영체제가 스레드 함수를 호출할 스레드를 생성하도록 할 때는 CreateThread 함수를 사용한다. CreateThread가 호출되면 시스템은 스레드 커널 오브젝트를 생성한다. 이는 운영체제가 스레드를 다루기 위한 데이터 구조체로, 스레드에 대한 통계 정보를 담는다. 그 이후 시스템은 스레드가 사용할 스택을 확보하게 된다. 

CreateThread의 구조과 각 매개변수 별 의미는 아래와 같다. 

```c++
HANDLE CreateThread(
    PSECURITY_ATTRIBUTES    psa,
    DWORD                   cbStackSize,
    PTHREAD_START_ROUTINE   pfnStartAddr,
    PVOID                   pvParam,
    DWORD                   dwCreateFlags,
    PDWORD                  pdwThreadID
);
```

**psa**: SECURITY_ATTRIBUTES 구조체를 가리키는 포인터로 보안 특성을 설정한다. 자식 프로세스에 스레드 커널 오브젝트 핸들을 상속하려면 이 구조체의 bInhertHandle 멤버를 TRUE로 초기화하면 된다. 

**cbStackSize**: 스레드 스택의 크기를 결정한다. 실행 파일 내에 저장된 스택 크기를 변경하기 위해서는 링커의 /STACK 스위치를 사용하면 된다. 기본값은 한페이지 크기이며, 스택 오버플로 예외가 발생하면 추가적인 페이지를 주소 공간 상에 커밋해주어 스택이 동적으로 커지게 된다. 매개변수를 0으로 전달할 경우 초기 크기를 따르게 된다. 

**pfnStartAddr**: 새로 생성되는 스레드가 호출할 스레드 함수의 주소를 가리킨다. 매개변수로 pvParam, 값이 전달된다. 

**pvParam**: 새로 생성되는 스레드 함수의 매개변수가 된다. 

**dwCreateFlags**: 스레드의 세부적인 제어를 위한 플래그를 전달한다. 0을 전달할 경우 CPU에 의해 스케줄 가능하게 되며, CREATE_SUSPENDED 플래그를 전달할 경우 스래드 생성 후 바로 스케줄 되지 않고 일시정지 상태를 유지하게 된다. 

**pdwThreadID**: 스레드 ID 값을 저장할 DWORD 변수 주소를 전달한다. 스레드의 ID 값이 필요 없다면 NULL을 전달하면 된다. 

<br/>

# **3. 스레드 종료**

## **1) 스레드 함수의 반환**

스레드를 종료하는 방법 중 가장 적절한 방법이다. 이 방법을 사용할 경우 스레드 함수 내 모든 C++ 오브젝트들이 파괴자를 통해 적절히 제거되며, 운영체제가 스레드 스택을 반환한다. 이후 시스템이 스레드 종료 코드를 스레드 함수의 반환 값으로 설정하고 스레드 커널 오브젝트의 Usage Count를 감소시킨다. 

## **2) ExitThread의 호출**

ExitThread를 호출한 스레드를 강제 종료되며, dwExitCode 매개변수로 스레드 종료 코들르 설정할 수 있다. 스레드에서 사용한 모든 운영체제 리소스는 정리되지만 C++ 클래스 오브젝트와 같은 C/C++ 리소스는 정리되지 않는다. 

## **3) TerminateThread의 호출**

TerminateThread의 매개변수로 전달한 핸들의 스레드를 강제 종료한다. 이는 ExitThread와 달리 스레드 스택을 정리하지 않으며, 강제적으로 종료된 스레드 스택을 다른 스레드가 참조하여도 접근 위반이 발생하지 않는다. 또, dll이 스레드 종료에 대한 통지도 받지 못하기 때문에 적절한 정리 작업을 수행하지 못할 수 있다.

이는 비동기 함수로, 이 함수를 호출한 시점에 해당 스레드가 종료되었음을 보장할 수 없다. 스레드가 종료되는 시점을 알아야 한다면 WaitForSingleObject 등을 사용해야 한다. 

## **4) 프로세스의 종료**

ExitProcess, TerminateProcess 등으로 프로세스가 종료될 경우 프로세스가 소유하고 있던 모든 스레드도 함께 종료된다. 프로세스를 강제 종료하면 남아있는 스레드에 대해 각각 TerminateThread 함수가 호출된다. 이 경우 프로세스가 사용하던 리소스들이 모두 정리되므로 스레드 스택들도 함께 정리되긴 하지만, C++ 파괴자는 적절하게 호출되지 못하여 디스크로 자료를 저장하는 등의 정리 작업도 수행되지 않는다. 

어플의 진입점 함수가 반환되면 C/C++ 런타임 라이브러리의 시작 코드는 ExitProcess를 호출한다. 따라서 주 스레드가 종료되기 전에 모든 스레드에 대해 적절한 정리 작업을 마쳐야 한다. 

## **5) 스레드 종료 시 발생하는 일**

**-** 스레드 커널 오브젝트 상태가 Signal로 변경되며 Usage Count가 1 감소한다. 

**-** 스레드가 소유하고 있던 유저 오브젝트 핸들이 삭제된다. 스레드에 의해 생성된 대부분의 오브젝트는 보통 프로세스에 의해 소유되지만, 윈도우와 윈도우 훅은 스레드에 의해 소유된다. 다른 형태의 오브젝트들은 프로세스 종료 시점에 파괴된다. 

**-** 스레드의 종료코드가 STILL_ACTIVE에서 ExitThread, TerminateThread가 지정한 종료 코드로 변경된다. 이때 타 스레드에서 GetExitCodeThread 함수로 종료된 스레드의 종료 코드를 획득할 수 있다. 

**-** 종료되는 스레드가 프로세스 내 마지막 활성 스레드일 경우 시스템이 프로세스도 같이 종료되어야 하는 것으로 간주한다.

<br/>

# **4. 스레드 내부 구조**

## **1) 스레드 구조**

![thread](https://user-images.githubusercontent.com/96677719/221468373-57a8b2b9-241b-499b-bf71-eab7e426612e.png)

CreateThread가 호출되면 시스템은 스레드 커널 오브젝트를 생성한다. Usage Count는 2로, 정지 Count는 1로, 종료 코드는 STILL_ACTIVE로, 오브젝트 상태는 non-signal로 각각 초기화된다. 또 스레드 컨텍스트 내의 SP는 스레드 스택의 주소를, IP는 NTDLL.dll 모듈이 export 하고 있는 RtlUserThreadStart 주소를 가리키게 된다. 

각 스레드는 자신만의 CPU 레지스터 세트를 가지는데 이를 스레드 컨텍스트라고 부른다. 이는 스레드가 마지막으로 수행되었을 당시 스레드의 CPU 레지스터 값을 가지며 이는 CONTEXT 구조체 형태로 스레드 커널 오브젝트 내에 저장된다. 스레드가 CPU 시간을 얻으면 시스템은 스레드 컨텍스트에 마지막으로 저장된 값을 CPU 레지스터로 로드하여 작업을 수행한다.

스레드 커널 오브젝트가 생성되고 나면 시스템은 스레드 스택 메모리 공간을 할당한다. 스레드는 자신만의 가상 메모리 공간을 가지지 않으므로 이는 프로세스 주소 공간으로부터 할당된다. 이후 시스템은 스레드 스택 가장 상위에 pvParam과 pfnStartAddr 값을 기록한다. 

스레드의 초기화가 완료되면 시스템은 CREATE_SUSPENDED 플래그를 확인한다. 이 플래그가 전달되지 않은 경우 스레드 정지 카운트를 0으로 감소시켜 스레드가 프로세서에 스케줄될 수 있도록 만들고, 전달된 경우 Resume 함수가 호출될 때까지 기다린다. 

스레드의 IP가 RtlUserThreadStart로 설정되어 있기 때문이 이 함수는 스레드가 실질적으로 수행하는 최초의 위치가 된다. RtlUserThreadStart의 매개변수는 사용자가 직접 스레드 스택을 통해 삽입한 것이 아니라 운영체제가 임의로 스레드 스택에 삽입한 값을 사용한다. 몇몇 아키텍쳐에서는 스택 대신 CPU 레지스터를 통해 매개변수를 전달하기도 한다. 

RtlUserThreadStart는 구조적 예외처리 프레임(SEH frame)을 설정하고 pvParam 매개변수로 스레드 함수를 호출한다. 스레드 함수가 반환되면 RtlUserThreadStart 함수가 스레드 함수 반환 값을 인자로 ExitThread 함수를 호출하여 스레드를 종료시킨다. 만약 스레드가 예외를 유발했는데 예외가 처리되지 않은 경우 RtlUserThreadStart의 SEH frame이 예외를 처리하게 된다. 

만약 RtlUserThreadStart의 SEH frame 출력한 메시지 박스에서 사용자가 프로그램 닫기를 선택할 경우 RtlUserThreadStart가 ExitProcess를 호출하여 예외 유발 스레드를 비롯한 전체 프로세스를 종료한다. 이 경우 스레드는 RtlUserThreadStart로부터 반환되지 못하고 내부적으로 종료되게 된다. 

RtlUserThreadStart 함수는 C/C++ 런타임 라이브러리의 시작 코드를 호출하여 각종 초기화를 진행하고 main 진입점 함수를 호출한 후, 진입점 함수가 반환되면 C/C++ 런타임 라이브러리 시작 코드가 ExitProcess를 호출한다. 따라서 C/C++ 애플리케이션의 주 스레드는 RtlUserThreadStart 함수로 반환되지 않는다. 

<br/>

# **5. C/C++ 런타임 라이브러리**

## **1) beginthreadex**

C/C++은 현재 멀티스레드를 제공하지만, 표준 C 런타임 라이브러리는 스레드 개념이 도입되기 전에 개발되었다. 따라서 errno, strtok, strerror, tmpfile, asctime, _ecvt 등과 같은 C/C++ 런타임 라이브러리의 각종 변수, 함수들은 멀티스레드 환경에서 문제를 발생시킨다. 따라서 C/C++ 런타임 라이브러리 함수들은 스레드별로 적절한 구조의 데이터 블록을 생성하여 다른 스레드로부터 영향을 받지 않도록, 자신을 호출한 스레드의 데이터 블록에서만 접근 가능하게끔 설계해야 한다. 

운영체제는 새로운 스레드가 생성되었을 때 어떻게 데이터 블록을 새로 할당해야 하는지 알 수 없다. 따라서 새로운 스레들르 생성할 때는 운영체제가 제공하는 CreateThread 함수를 호출하는 대신 C/C++ 런타임 라이브러리가 제공하는 _beginthreadex 함수를 호출하기를 권장한다. _beginthreadex 의 반환형 및 매개변수는 CreateThread와 동일한 의미를 갖고 있으나 이름과 형태에는 차이가 있으므로 형변환을 해주는 것이 좋다.

```c++
unsigned long _beginthreadex(
    void* security,
    unsigned stack_size,
    unsigned (*start_address)(void*),
    void* arglist,
    unsigned initflag,
    unsigned &thrdaddr
)
```

_beginthreadex의 소스 코드를 수도코드 형태로 간단하게 옮겨보면 아래와 같다. 

```c++
uintptr_t __cdecl _beginthreadex(
    void* psa,
    unsigned cbStackSize,
    unsigned (__stdcall * pfnStartAddr)(void*),
    void* pvParam,
    unsigned dwCreateFlags,
    unsigned *pdwThreadID)
{
    _ptiddata ptd;  // thread's data block pointer
    uintptr_t thdl; // thread handle

    if((ptd = (_ptiddata)_calloc_crt(1, sizeof(struct _tiddata))) == NULL)
        goto error_return;

    initptd(ptd);

    ptd -> _initaddr = (void* ) pfnStartAddr;
    ptd -> _initarg = pvParam;
    ptd -> _thandle = (uintptr_t)(-1);

    thdl = (uintptr_t) CreateThread((LPSECURITY_ATTRIBUTES)pas, cbStackSize, _threadstartex, (PVOID) ptd, dwCreateFlags, pdwThreadID);

    if(thdl == 0)
        goto error_return;
    
    return(thdl);

    error_return:

    _free_crt(ptd);
    return((uintptr_t)0L);
}
```

각 스레드는 C/C++ 런타임 라이브러리 힙에 _tiddata 메모리 블록을 가지며, _beginthreadex 함수에 전달된 스레드 함수의 주소와 매개변수를 _tiddata 메모리 블록 내에 저장한다. 그 후 _beginthread에서 내부적으로 CreateThread를 호출하여 운영체제로 하여금 새로운 스레드를 생성하도록 명령한다. CreateThread가 호출되면 _threadstartex 함수가 수행되며 매개변수 역시 pvParam을 직접 전달하는 대신 _tiddata 구조체의 주소를 통해 전달한다. 그 후 CreateThread와 동일하게 스레드 핸들을 반환한다. 

만일 _beginthreadex로 만든 스레드를 강제로 종료해야 할 경우 _endthreadex를 통해 종료해야 한다. 이는 내부적으로 _tiddata를 해제한 뒤 ExitThread를 호출한다. 만일 ExitThread 함수를 직접 호출할 경우 스레드의 _tiddata 메모리 블록이 삭제되지 않아서 메모리 누수로 이어지게 된다. 물론 _endthreadex도 스레드의 종료 파트에서 설명했던 이유로 되도록 사용하지 않는 것이 좋다. 

이와 같이 데이터 블록을 초기화 하고 스레드와 연계하게 되면, 스레드별로 저장된 데이터에 접근해야 하는 C/C++ 런타임 라이브러리 함수들이 호출하는 스레드의 데이터 블록 주소를 보다 손쉽게 획득할 수 있다. errno 역시 전역변수이긴 하지만 호출한 스레드와 연관된 데이터 블록으로부터 errno 데이터 멤버 주소를 반환하게 되기 때문에 정상적으로 사용할 수 있게 된다. 

C/C++ 런타임 라이브러리는 함수의 동기화 관련 문제와도 연관되어 있는데, 예를 들어 두개의 스레드가 동시에 malloc을 호출하면 힙이 깨진다. 따라서 한 스레드가 반환될 때까지 다른 스레드는 malloc을 수행하지 못하고 대기 상태가 된다. 이러한 상황은 성능에 영향을 미친다. 멀티 스레드를 설계할 때는 이러한 것도 고려하여 설계해야 한다. 

## **2) CreateThread를 호출한 상황**

beginthreadex 대신 CreateThread를 호출하게 되면 _tiddata 구조체가 생기지 않는다. C/C++ 런타임 라이브러리 함수는 스레드의 데이터 블록을 가져오려고 시도하는데, 이때 _tiddata 블록의 주소로 NULL이 반환되게 되면 C/C++ 런타임 함수가 해당 스레드에서 사용할 _tiddata 블록을 새로 할당하게 된다. 이 블록은 스레드가 수행되는 동안 계속 유지된다. 

그러나 이 상황은 몇가지 문제가 생길 수 있다. 첫째로 C/C++ 런타임 라이브러리가 제공하는 signal 함수를 사용하는 경우 SEH frame이 준비되지 않은 상황이므로 프로세스가 종료되어 버린다. 둘째로 endthreadex를 호출하지 않고 종료하게 되면 데이터 블록이 삭제되지 않아 메모리 누수가 발생하게 된다. 

## **3) 호출하지 말아야 할 C/C++ 라이브러리 함수**
```c++
unsigned long _beginthread
(
    void (__cdecl * start_address)(void*),
    unsigned stack_size,
    void* arglist
);
```
```c++
void _endthread(void);
```
beginthread는 매개변수가 부족하기 때문에 보안 특성을 가진 스레드도, 일시 정지 상태의 스레드도 생성할 수 없고 스레드 ID 값도 얻을 수 없다. _endthread 역시 항상 종료 코드가 0이 되며, ExitThread 호출 직전에 새로 생성된 스레드 핸들을 인자로 CloseHandle을 호출한다는 문제가 생긴다. 이 두 함수는 안정적이지 않은 함수이므로 사용하지 않는 것이 좋다. 

<br/>

# **6. 구분자 얻기**

## **1) 구분자 얻기**

스레드가 수행되면 우선순위와 같은 자신의 수행 환경을 변경하기 위해 윈도우 함수를 호출해야 할 떄가 있다. 이 경우 아래 함수를 이용하면 자신이 속한 프로세스 커널 오브젝트 및 스레드 커널 오브젝트를 손쉽게 얻을 수 있다. 

```c++
HANDLE GetCurrentProcess();
HANDLE GetCurrentThread();
```
이들은 새로운 핸들을 생성하는 대신 해당 커널 오브젝트의 허위 핸들을 반환하기 때문에 Usage Count에 영향을 미치지 않는다. 이렇게 반환받은 핸들로 CloseHandle을 할 경우 함수 호출 자체가 무시된다. 

또, 유사하게 GetProcessTimes, GetThreadTimes 함수를 호출하면 시간 사용 정보를 획득할 수 있으며, GetCurrentProcessID, GetCurrentThreadID 를 호출하면 ID 값을 획득할 수 있다. 

## **2) 실제 핸들 얻기**

실제 핸들이 필요한 상황에서는 DuplicateHandle 함수를 사용하여 허위핸들을 실제 핸들로 변경할 수 있다. 그 사용 예제는 아래와 같다. 

```c++
DWORD WINAPI ParentThread(PVOID pvParam)
{
    HANDLE hThreadParent;
    DuplicateHandle(
        GetCurrentProcess(),
        GetCurrentThread(),
        GetCurrentProcess(),
        &hThreadParent,
        0, FALSE, 
        DUPLICATE_SAME_ACCESS
        );

    CreateThread(NULL, 0, ChildThread, (PVOID) hThreadParent, 0, NULL);

    //...
}
```
이 경우 허위 핸들이 부모 스레드를 나타내는 실제 핸들로 변경되며, 이렇게 획득한 실제 핸들을 CreateThread의 매개변수로 전달할 경우 pvParam 매개변수가 실제 핸들을 가지게 된다. 따라서 이 핸들을 이용한 함수들은 자식 스레드 대신 부모 스레드에 영향을 미칠 수 있게 된다. 

DuplicateHandle은 Usage Count를 증가시키기 때문에 복사한 오브젝트 핸들이 더 이상 필요 없어지면 CloseHandle을 반드시 호출해주어야 한다.위 예제에서는 해당 핸들을 전달 받은 자식 스레드에서 작업을 마친 후 CloseHandle(hThreadParent)를 수행해야 한다. 
