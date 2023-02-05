---
title: Basic 03) Kernel Object
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Kernel Object란?**

## **1) Kernel Object**

커널 오브젝트는 커널에 의해 할당된 간단한 메모리 블록이다. 이 메모리 블록은 커널에 의해서만 접근 가능한 구조체로 구성되어있으며, 사용자가 접근하기 위해서는 마이크로소프트에서 제공하는 함수를 사용해야 한다. 해당 메모리 블록은 오브젝트에 대한 정보값을 갖고 있다. 

커널 오브젝트 생성 함수를 호출하면 함수는 각 커널 오브젝트를 구분하기 위한 핸들 값을 반환한다. 핸들 값은 프로세스 내 모든 스레드에 의해 사용가능하며 32bit에서는 32bit 값을, 64bit에서는 64bit 값을 가진다. 이들은 다양한 윈도우 함수의 매개변수로 전달되며, 운영체제는 매개변수로 전달된 핸들 값을 통해 어떤 커널 오브젝트를 조작하고자 하는지 구분한다. 

<br/>

# **2. Kernel Object의 구성 요소**

## **1) Kernel Object의 구성 요소**

운영체제는 Access Token Object, File Object, Event Object, Job Object, Mutex Object, Pipe Object, Process Object, Thread Object 등 다양한 커널 오브젝트를 생성, 조작한다. 이들이 갖는 대부분의 값은 오브젝트 별로 독특하나, Security descriptor, Usage count 등은 모든 오브젝트 타입에 공통적으로 존재하는 값도 있다.  

## **2) Usage count**

커널 오브젝트는 프로세스가 아니라 커널에 의해 소유된다. 따라서 프로세스가 커널 오브젝트를 생성한 후 종료되어도, 생성된 커널 오브젝트가 그 프로세스와 함께 삭제되는 것은 아니다. 만일 다른 프로세스가 해당 커널 오브젝트를 사용하고 있다면, 더 이상 자신을 사용하는 프로세스가 없을 때까지 기다렸다가 삭제되게 된다. 각 커널 오브젝트는 Usage count에 얼마나 많은 프로세스가 자신을 사용하는지 기록해두었다가, 이 값이 0이 되었을 때 삭제된다.  

## **3) Security descriptor**

커널 오브젝트는 Security descriptor, 보안 디스크립터를 통해 보호된다. 이는 누가 커널 오브젝트를 소유하고 있는지, 어떤 사용자들에 의해 사용될 수 있는지, 어떤 사용자들에 대해 접근이 제한되어 있는지 등에 대한 정보를 갖고 있다. SECURITY_ATTRIBUTES의 lpSecurityDescriptor 값을 통해 해당 정보를 지정할 수 있다. 만약 lpSecurityDescriptor 상에서 접근 권한을 부여받지 못한 접근을 시도할 경우 GetLastError를 호출해보면 ERROR_ACCESS_DENIED (5)가 반환되게 된다. 

특정 오브젝트가 커널 오브젝트인지 여부를 판단하는 가장 간단한 방법은 해당 오브젝트를 생성하기 위한 함수가 무엇인지 찾아보고, 그 함수에 보안 특성을 지정하는 매개변수가 있는지 확인하는 것이다. 유저 오브젝트, GDI 오브젝트를 생성하는 함수 중 PSECURITY_ATTRIBUTES형 매개변수를 취하는 함수는 없기에 이런 경우 커널 오브젝트가 아님을 알 수 있게 된다. 

<br/>

# **3. 프로세스와 Kernel Object**

## **1) Process Initailize**

프로세스가 초기화 되면 운영체제는 프로세스를 위해 Kernel Object Handle Table을 할당한다. 이러한 핸들 테이블은 유저 오브젝트, GDI 오브젝트에 의해 사용되지 않고 유일하게 커널 오브젝트에 의해서만 사용된다. 프로세스의 Object Handle Table은 단순한 데이터 구조체의 배열로 이루어져 있으며, 각 데이터 구조체는 커널 오브젝트에 대한 포인터, 액세스 마스크, 플래그로 구성된다. 

## **2) Create Kernel Object**

커널 오브젝트 생성 함수를 사용하면 프로세스 별로 고유한 핸들 값을 반환하며, 이 값은 프로세스 내의 모든 스레드에 의해 공유된다. 반환된 핸들 값을 2 right shift 하면 프로세스 핸들 테이블의 인덱스 값을 얻을 수 있다. 그러나 이러한 핸들 값 자체의 의미는 문서화 되어있지 않으며 버전에 따라 변경될 수 있다. 커널 오브젝트 생성 함수가 실패하면 일반적으로 0 (예외 종류에 따라 -1)을 반환하기 때문에, 유효한 커널 오브젝트 핸들 값은 4부터 시작된다. 

## **3) Close Handle**

CloseHandle을 호출하면 더 이상 해당 커널 오브젝트를 사용하지 않을 것이라고 시스템에게 전달하게 된다. 그렇다고 커널 오브젝트가 바로 사라지는 것은 아니며, 커널 오브젝트를 사용하던 프로세스들이 모두 종료되어 Usage Count가 0이 될 때 사라지게 된다. CloseHandle을 했다면 해당 핸들을 갖고 있는 변수 값도 바로 NULL로 초기화 하는 것이 좋다. 만약 해당 위치에 새로운 커널 오브젝트가 생성된 상황이라면 의도하지 않은 예외가 발생할 수 있다. 참고로 프로세스가 종료되면 CloseHandle을 해주지 않았어도 자동으로 핸들을 반환해준다. 그럼에도 누수의 위험을 막기 위해 명시하는 것이 좋다. 

<br/>

# **4. 프로세스 간 Kernel Object 공유**

## **1) 프로세스 간 공유가 필요한 상황**

프로세스 간 Kernel Object 공유가 필요한 상황은 크게 세가지가 있다. 

**-** File-mapping 오브젝트는 단일 머신에서 수행되는 두 프로세스 사이에서 데이터 블록을 공유할 수 있게 해준다. 

**-** MailSlot, Named Pipe를 이용하면 네트워크로 연결된 서로 다른 머신 사이에서 데이터를 주고 받을 수 있다. 

**-** Mutex, Semaphore, Event는 서로 다른 프로세스에서 수행되는 스레드 간의 동기화를 수행하게 해준다. 

이러한 상황에서 프로세스 간 커널 오브젝트를 공유하는 방법으로는 오브젝트 핸들 상속을 이용하는 방법, 명명된 오브젝트를 사용하는 방법, 오브젝트 핸들 복사를 사용하는 방법으로 크게 세가지가 있다. 

## **2) 방법 1 - 오브젝트 핸들 상속**

이 방법은 공유하려는 프로세스들이 parent - child 관계일 때만 사용할 수 있다. SECURITY_ATTRIBUTES를 만들 때, bInheritHandle 을 TRUE로 설정하면 상속 가능한 핸들이 생성된다. 이때 상속이 되는 것은 오브젝트에 대한 핸들이지 오브젝트 그 자체가 아님에 유의해야 한다. 이렇게 설정한 상태에서 CreateProcess의 bInhertHandles 매개변수를 TRUE로 전달하게 되면 그 결과로 나온 자식 프로세스는 부모 프로세스와 bInheritHandle 을 TRUE인 모든 핸들을 공유하게 된다. 이때 child 프로세스 핸들 테이블 내의 복사 위치는 parent 프로세스 핸들 테이블에서의 위치와 정확히 일치한다. 따라서 동일한 핸들 값을 이용해 동일한 오브젝트에 접근할 수 있게 된다. 해당 커널 오브젝트의 Usage Count 역시 동일하게 증가된다. 

이때 핸들이 상속되는 순간은 CreateProcess가 호출되는 순간이다. 그 이후 부모 프로세스에서 상속 가능한 새로운 커널 오브젝트를 생성해도 이미 수행되고 있었던 자식 프로세스는 이 핸들을 상속 받지 못한다. 또, 독특한 특징 중 하나는 핸들 상속을 이용할 경우 자식 프로세스는 어떤 핸들이 상속된 것인지 알 수 없다. 이를 보완하는 가장 일반적인 방법은 자식 프로세스 수행 시 명령행 인자를 이용해 핸들 값을 전달하는 것이다. 이러면 자식 프로세스의 초기화 코드를 명령행 인자를 분석(보통 _stscanf_s를 사용)하여 핸들 값을 얻어낼 수 있으며, 이렇게 획득한 핸들 값은 부모 프로세스에서와 동일한 접근 권한을 갖게 된다. 핸들 상속은 공유 커널 오브젝트에 대한 핸들값이 동일하게 유지되는 유일한 공유 방법이기에, 부모 프로세스는 명령행 인자로 핸들값을 전달할 수 있다. 

만약 부모 프로세스가 두개의 자식을 갖는데 그 중 하나에게만 핸들을 상속하고 싶을 경우 SetHandleInformation 함수를 이용해 상속 플래그를 변경하면 된다. 그 예제는 아래와 같다. 

```c++
SetHandleInformation(hObj, HANDLE_FLAG_INHERIT, HANDLE_FLAG_INHERIT); // 상속 가능 (HANDLE_FLAG_INHERIT == 0x00000001)
SetHandleInformation(hObj, HANDLE_FLAG_INHERIT, 0); // 상속 불가 ( 0 = 0x00000000 )
```

이 외에도 HANDLE_FLAG_PROTECT_FROM_CLOSE 플래그도 존재한다. 이는 운영체제에게 이 플래그가 설정된 핸들은 삭제할 수 없음을 알려주기 위해 쓰인다.

```c++
SetHandleInformation(hObj, HANDLE_FLAG_INHERIT, HANDLE_FLAG_PROTECT_FROM_CLOSE); 
CloseHandle(hObj);
```

위와 같은 코드를 실행할 경우 예외가 발생한다. 이는 흔한 방법은 아니지만 자식 프로세스가 또 다른 자식 프로세스를 생성하는 구조에서 부모 프로세스를 생성할 때, 상속이 이루어지기 전에 핸들을 닫아서 통신이 실패하는 상황을 방지할 수 있는 방법이다. 

```c++
DWORD dwFlags;
GetHandleInformation(hObj, &dwFlags); 
BOOL bHandleIsInheritable = (0 != (dwFlags & HANDLE_FLAG_INHERIT));
```
역으로 위와 같은 방법을 사용하여 해당 핸들의 상속 여부를 확인할 수 있다. 

그 외에 부모-자식 프로세스 간에 핸들을 전달하는 방법으로는 프로세스 간 통신 방법을 이용해 커널 오브젝트의 핸들을 전달하는 방법, 부모 프로세스가 환경 변수 블록에 상속할 핸들 값을 가진 새로운 환경 변수를 추가하는 방법 등이 있다. 후자 방법의 경우 환경 변수는 계속해서 상속되기 때문에 자식 프로세스가 또 다른 자식 프로세스를 생성하는 경우 특히 유용하게 활용할 수 있다. 이 경우 간편하게 GetEnvironmentVariable 함수를 통해 상속된 오브젝트 핸들 값을 얻을 수 있다.

## **3) 방법 2 - 명명된 오브젝트 사용**

이 경우 부모 - 자식 관계가 아닌 프로세스 간에도 오브젝트 핸들을 공유할 수 있다. 대부분의 커널 오브젝트는 Create 시에 이름을 지정할 수 있는데, 이를 이용하여 공유하는 것이다. 이름은 최대 MAX_PATH(260) 길이로 지정할 수 있으며, 명명 규칙이 제공되지 않기 때문에 동일한 이름을 만들지 않도록 유의해야 한다. A 프로세스에서 만든 것과 동일한 타입, 동일한 이름의 오브젝트를 B 프로세스에서 Create 하게 될 경우, 해당 오브젝트에 접근할 수 있는 B 프로세스 고유의 핸들 값을 생성, 반환된다. 

이때 Create 대신 Open 함수를 사용할 수도 있다. Create를 사용하면 동일한 커널 오브젝트가 없을 경우 새로운 오브젝트를 생성하는 반면 Open은 함수 호출에 실패한다. 만약 이름은 일치하지만 타입이 일치하지 않거나 접근 권한이 없는 경우 두 경우 다 함수의 반환값으로 NULL을, GetLastError에서는 ERROR_INVALID_HANDLE을 반환한다. 반면 일치하는 이름이 없을 경우 Create는 해당 오브젝트를 생성, Open의 경우 함수 반환값으로 NULL을, GetLastError에서는 ERROR_FILE_NOT_FOUND를 반환한다. Create 함수를 호출하여 기존 오브젝트를 참조할 때는 이름, 타입만 사용되고 그 외의 보안 특성 정보 및 두번째 인자의 경우 무시된다. 

중요한 것은 B에서 새로운 오브젝트를 생성하는 것이 아니며, A와 B의 핸들은 동일한 오브젝트를 가리키지만 각각 다른 값을 갖고 있다는 것이다. 당연히 해당 오브젝트의 Usage Count는 증가하게 된다. 또, 유의해야 할 것은 윈도우 비스타 이전 버전에서는 다른 프로세스가 공유 오브젝트의 이름을 훔치는 것으로부터 모호할 수 없었다. 이러한 방식은 Dos 공격의 기본 매커니즘이 된다. 따라서 공유할 필요가 없는 오브젝트의 경우 명명하지 않고 사용하는 것이 보다 일반저이다. 

## **4) 방법 3 - 오브젝트 핸들 복사**

DuplicateHandle 함수를 호출하여 커널 오브젝트 핸들을 복사할 수 있다. DuplicateHandle 함수의 원형은 아래와 같다. 

```c++
BOOL DuplicateHandle (
    HANDLE  hSourceProcessHandle,
    HANDLE  hSourceHandle,
    HANDLE  hTargetProcessHandle,
    PHANDLE phTargetHandle,
    DWORD   dwDesiredAccess,
    BOOL    bInheritHandle,
    DWORD   dwOptions
)
```
hSourceProcessHandle, hTargetProcessHandle에 프로세스 커널 오브젝트의 핸들을 넘겨주어야 한다. 이때 두 매개변수에는 반드시 프로세스 커널 오브젝트에 대한 핸들을 전달해야 한다. 만일 다른 타입의 커널 오브젝트를 전달하면 이 함수는 실패하게 된다. 

hSourceHandle에는 어떤 타입의 커널 오브젝트라도 전달할 수 있다. 이는 DuplicateHandle 함수를 호출한 프로세스와 아무 연관성을 가지지 않으며 hSourceProcessHandle 이 가리키는 프로세스에서만 의미를 가지는 프로세스 고유 값이다. phTargetHandle로는 HANDLE 변수의 주소값을 전달하게 되며, 함수 호출 이후 hTargetProcessHandle가 가리키는 프로세스에서만 사용될 수 있는 고유의 핸들 값, 즉 소스 핸들의 복사본을 전달 받게 된다. 

dwDesiredAccess, bInheritHandle를 통해서는 액세스 마스크와 상속 플래그의 값을 지정한다. dwOptions에는 DUPLICATE_SAME_ACCESS, DUPLICATE_CLOSE_SOURCE를 사용할 수 있다. 전자는 동일한 액세스 마스크를 전달하고 싶을 때 지정하며, 후자는 소스 프로세스에서 해당 핸들을 삭제할 때 사용한다. 

커널 상속과 마찬가지로 이 함수의 단점 중 하나는 Target Process가 새로운 커널 오브젝트에 접근 가능하게 되었다는 사실을 통보받지 못한다는 것이다. 이를 명시해주고 싶다면 윈도우 메세지, IPC 등의 통신 방법을 이용해 핸들 값을 전달해야 할 것이다. 

<br/>

# **5. Namespace**

## **1) Terminal Service Namespace**

터미널 서비스를 수행하는 머신은 커널 오브젝트에 대해 다수의 Namespace를 가진다. 모든 Terminal Service Client Session에는 접근 가능한 커널 오브젝트를 위한 전역 NameSpcae가 있는데, 이는 주로 서비스 타입의 어플리케이션에 의해 사용된다. 이와는 별도로 각 Client Session은 고유 Namespace를 가지게 된다. 이로 인해 설령 오브젝트 이름이 갖더라도 다른 세션의 오브젝트에 접근할 수 없기 때문에, 다수의 세션에서 동일한 어플리케이션이 서로간에 영향을 미치지 않게 된다. 이는 서버 머신 뿐 아니라 리모트 데스크톱, 빠른 사용자 전환에서도 동일하게 사용된다. 

어떤 터미널 세션에서 특정 프로세스가 수행되는지 알고 싶다면 ProcessIdToSessionId(processID, &sessionID) 함수를 사용하면 된다. 또, 커널 오브젝트를 생성할 때 인자값을 통해 전역으로 만들 것인지 현재 세션의 Namespace 내에 만들 것인지 지정할 수 있다. 그 예시는 아래와 같다. 

```c++
Handle h_global = CreateEvent(NULL, FALSE, FALSE, TEXT("Global\\MyName")); 
Handle h_local = CreateEvent(NULL, FALSE, FALSE, TEXT("Local\\MyName")); 
```
마이크로소프트는 Global, Local을 예약된 키워드로 간주하여 오브젝트 이름에 사용하지 않도록 권고한다. Session 역시 예약된 키워드로 간주하며 "Session\\\<current session ID>" 와 같이 사용된다. 이때 Session 키워드를 사용한다고 해서 현재 수행 중인 세션이 아닌 다른 세션에 오브젝트를 생성하는 것은 불가능하다. 그 경우 GetLastError에서 ERROR_ACCESS_DENIED를 반환한다. 

## **2) Private Namespace**

커널 오브젝트가 다른 어플리케이션에서 사용하는 오브젝트 이름과 절대 충돌하지 않게끔 만들기 위해서, CreatePrivateNamespace를 이용해 사용자 고유의 Private Namespace를 만들 수 있다. 이는 커널 오브젝트를 담기 위한 일종의 디렉토리와 같다. 디렉토리처럼 private namespace는 보안 디스크립터를 갖고 있고, 이 값은 CreaetPrivateNamespace를 호출할 때 연계된다. 

파일 시스템의 디렉토리와 다른 점은 상위 디렉토리도, 이름도 가지지 않는다는 것이다. 프로세스 익스플로러로 Private Namespace에 속한 오브젝트를 살펴보면 이름 대신 "...\\"와 같이 나타난다. 이를 통해 정보를 숨기고 잠재적 해커로부터 안전하게 보호할 수 있다. Private namespace의 이름은 프로세스 내에서만 보이는 일종의 별칭이다. 동일한 프로세스 내에서도 동일한 Private namespace를 열어서 다른 별칭을 부여할 수 있다. 

일반적으로 Namespace를 생성할 때는 바운더리에 대한 테스트가 먼저 수행되는데, 이때 현재 스레드의 토큰은 반드시 바운더리 디스크립터가 갖고 있는 SID 값을 포함해야 한다. 

<br/>
