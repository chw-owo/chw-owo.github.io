---
title: DLL 04) DLL Injection
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. DLL 인젝션**

MS 윈도우는 기본적으로 프로세스들이 자신만의 주소 공간을 가진다. 그러나 디버깅에 필요한 경우, 다른 프로세스를 후킹하는 경우 다른 프로세스에 의해 생성된 윈도우를 Subclassing 하는 경우 등에는 다른 프로세스의 주소 공간에 접근해야 한다. 이때, 다른 프로세스 주소 공간에 DLL을 인젝션하면 해당 프로세스를 제한 없이 조작할 수 있게 된다. 

```c++
SetWindowLongPtr( hWnd, GWLP_WNDPROC, MySubclassProc);
```

Subclass로 예를 들어보자. Subclassing을 수행할 땐 SetWindowLongPtr로 윈도우 메모리 블록 내 윈도우 프로시저 포인터를 새로운 WndProc를 가리키는 값으로 변경한다. 특정 윈도우를 Subclassing 하기 위해 이를 호출한다면 시스템은 hWnd로 send, post 되는 모든 윈도우 메시지를 기본 윈도우 프로시저 대신 MySubclassProc으로 전달하게 될 것이다.

![Subclass](https://user-images.githubusercontent.com/96677719/226794772-6aed7a91-0834-407a-9fe2-cae32201aee0.png)

위 그림은 어떻게 윈도우 프로시저가 메시지를 수신하는지 보여준다. User32.dll은 A 공간에 매핑되었으므로 A의 스레드가 생성한 윈도우에 메시지를 send, post할 책임이 있다. User32.dll이 메시지를 발견하면 먼저 WndProc 주소를 찾고, 윈도우 핸들, 메시지, wParam, lParam을 인자로 해당 함수를 호출한 후, 다시 대기하게 될 것이다. 

이때 B에서 A의 윈도우에 SetWindowLongPtr를 시도한다면, WndProc 값을 변경할 것 같지만 실제로는 그냥 NULL값이 반환된다. 이 함수는 다른 프로세스가 생성한 윈도우의 주소를 변경하려고 하면 함수 호출 자체를 무시하기 때문이다. 무시하지 않는다면 MySubclassProc가 B에 있으므로, User32가 이 주소에 접근했을 때 메모리 접근 위반이 발생할 것이다.   

컨텍스트 전환 후 함수를 호출하도록 구현되어 있다면 예외 없이 작동할 수도 있겠지만 MS는 이를 구현하지 않았다. 우선 다른 프로세스를 Subclassing 하는 상황이 드물며, 프로세스 전환이 CPU에게 비싼 작업이고, MySubclassProc 코드를 수행할 스레드가 명확하지 않으며, User32.이 연계된 주소가 동일 프로세스 것인지 아닌지 알기 어렵기 때문이다. 

이런 이유들로 다른 프로세스의 윈도우를 Subclassing 하기 위해서는 다른 방법을 사용해야 한다. A의 주소 공간에 Subclassing을 위한 윈도우 프로시저를 포함시킬 수 있다면 SetWindowLongPtr으로 윈도우 프로시저를 변경할 수 있다. 이러한 기법을 두고 프로세스의 주소 공간으로 DLL을 인젝션 한다고 하며, 이를 위한 다양한 방법이 있다. 

<br/>

# **2. 레지스트리를 이용한 DLL 인젝션**

```
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\Windows
```
이 키 이하에는 AppInit_DLLs 항목이 있는데, 이는 한개 혹은 여러개의 DLL 파일 이름을 가질 수 있다. 이때 첫번째 DLL 이름만 경로를 포함할 수 있으므로 사용자 DLL 파일을 모두 윈도우 시스템 디렉토리에 두는 것이 좋다. 그리고 LoadAppInit_DLs 항목에 지정된 파일의 개수를 지정한다. 

DLL_PROCESS_ATTACH 통지를 받으면 AppInit_DLLs에 있던 DLL 파일들을 LoadLibrary로 읽어오게 되고, 각 라이브러리마다 자신의 DllMain을 호출하며 초기화 된다. 인젝션 되는 DLL은 초기에 로드되므로 내부적으로 함수를 호출할 때는 주의해야 한다. 

이는 DLL을 인젝션 하는 가장 간단한 방법이지만 User32.dll을 사용하는 프로세스에 대해서만 인젝션이 수행된다는 한계가 있다. GUI 기반 애플리케이션은 이 DLL을 포함하지만 CUI 기반 애플리케이션은 대부분 사용하지 않기 때문에 다른 방법을 사용해야 한다. 

또, 설정된 파일은 User32.dll을 쓰는 모든 곳에 매핑되지 하나씩 제한적으로 인젝션할 수 없다. 또, 종료될 때까지 계속해서 매핑 상태를 유지하게 된다. 이를 매핑하는 프로세스가 많아질수록 프로세스가 손상될 확률도 높아지므로 최대한 적게, 최대한 짧게 매핑을 유지하는 것이 좋다. 

<br/>

# **3. 윈도우 훅을 이용한 DLL 인젝션**

```c++
HHOOK hHook = SetWindowsHookEx(WH_GETMESSAGE, GetMsgProc, hInstDll, 0);
```

이를 통해 WH_GETMESSAGE 훅을 설치, 윈도우들이 처리하는 메시지를 볼 수 있다. WH_GETMESSAGE에는 훅 형태를, GetMsgProc에는 호출할 함수 주소를, hInstDll에는 DLL이 로드된 가상 메모리의 시작주소를 전달한다. 마지막 인자에는 후킹하려는 스레드의 ID를 전달하는데, 0을 전달할 경우 시스템 내 모든 GUI 스레드를 후킹하게 된다. 

시스템은 스레드에 WH_GETMESSAGE 훅이 설치되어 있는지, GetMsgProc 함수를 포함한 DLL이 프로세스 주소 공간에 매핑되어 있는지 확인하고, 매핑되어 있지 않다면 매핑한다. 그 후 DLL의 hInstaDLL 값이 대상 프로세스에 매핑했을 때와 동일한 값인지 확인한다. 일치할 경우 GetMsgProc 주소로 기존 프로세스 내 매핑된 GetMsgProc을 호출하고, 다른 값을 가진 경우 가상 메모리 주소를 계산하여 접근한다. 이 과정이 다 진행됐다면 기존 프로세스 내 DLL의 락 카운트를 증가시키고, 거기 매핑되어 있던 GetMsgProc 함수를 호출한다. 함수가 반환되면 다시 락 카운트를 감소시킬 것이다. 

훅 필터 함수를 가진 DLL을 인젝션 하기 때문에, 일단 DLL이 매핑되고 나면 훅 필터 함수 외의 DLL 내 모든 함수를 컨텍스트 내에서 호출할 수 있게 된다. 이를 위해서는 서브클래싱 할 윈도우 생성 스레드에 WH_GETMESSAGE 훅을 설치해야 한다. 그 다음 GetMsgProc 함수가 호출되면 SetWindowLongPtr 함수 호출로 윈도우를 서브클래싱 하게 된다. 물론 서브클래싱 할 윈도우 프로시저도 GetMsgProc 함수가 포함된 DLL 내에 같이 포함되어 있어야 한다. 

```c++
BOOL UnhookWindowsHookEx(HHOOK hHook);
```

레지스트리를 이용한 방법과 달리 해당 DLL이 더 이상 필요하지 않다면 위 함수를 통해 매핑을 해제할 수 있으며 이 경우 락 카운트가 감소된다. 이때 시스템은 GetMsgProc 함수 호출 직전에 DLL 락 카운트를 증가시키기 때문에, 윈도우 서브클래싱을 수행하자마자 바로 훅을 해제할 수는 없다. 훅은 서브클래싱이 진행 중인 동안에는 반드시 같이 살아있어야 한다. 

<br/>

# **4. 원격 스레드를 이용한 DLL 인젝션**

이는 유연성이 가장 뛰어난 방법이지만 이를 사용하기 위해서는 프로세스, 스레드, 스레드 동기화, 가상 메모리 관리, DLL, 유니코드와 같은 윈도우 구성 요소에 대해 이해하고 있어야 한다. 대부분의 윈도우 함수는 자신을 호출하는 프로세스에만 영향을 미치지만, 일부 함수는 다른 프로세스에 영향을 줄 수 있다. 이는 디버거 등의 도구 개발을 위해 제공되며 어떤 애플리케이션이든 제한 없이 사용할 수 있다. CreateRemoteThread가 그 예시이며, 이를 이용하면 다른 프로세스 내 새로운 스레드를 생성할 수 있다. 이 방법을 통해 DLL 인젝션을 수행할 수 있다. 

```c++
HANDLE CreateRemooteThread (
    HANDLE hProcess,
    PSECURITY_ATTRIBUTES psa,
    DWORD dwStackSize,
    PTHREAD_START_ROUTINE pfnStartAddr,
    PVOID pvParam,
    DWORD fdwCreate,
    PDWORD pdwThreadId
);
```

CreateRemoteThread는 hProcess 인자를 제외하면 CreateThread와 동일하다. pfnStartAddr는 스레드 함수의 메모리 주소를 나타내는데 이는 원격 프로세스와 연관된 값이다. 스레드 함수는 이 함수를 호출한 프로세스의 주소 공간에 있어서는 안된다. 기본적으로 DLL 인젝션은 DLL을 삽입하고자 하는 프로세스의 스레드가 인젝션할 DLL에 대해 LoadLibrary를 수행하도록 하는 것이다. 이렇게 생성한 스레드가 LoadLibrary를 호출하도록 하면 다른 프로세스의 스레드를 임의로 제어하는 것보다 훨씬 쉽게 DLL 인젝션을 수행할 수 있다. 

```c++
HANDLE hThread = CreateRemoteThread(hProcessRemote, NULL, 0,
                        LoadLibraryW, L"C:\\MyLib.dll", 0, NULL);
```

로드하고자 하는 라이브러리 파일명이 ANSI라면 LoadLibraryA를, 유니코드라면 LoadLibraryW를 호출한다. 이 함수의 원형은 스레드 함수의 원형과 유사하며 동일하게 WINAPI 규약을 따르기 때문에 위와 같이 사용하면 될 것 같다. 그러나 위와 같이 사용할 경우 두가지 문제가 생긴다. 

링크의 결과물인 바이너리는 import section을 가지며, 이 section은 함수의 thunk를 가리키는 값으로 구성되어 있다. 같은 함수를 사용하는 코드를 링크할 경우, 링커는 모듈의 import section에 thunk를 호출하도록 정보를 기록하고 thunk 다음으로 실제 함수가 호출된다. 따라서 위처럼 LoadLibrary를 직접 사용하면, 이 값은 LoadLibrary의 메모리 주소가 아니라 모듈 import section 내 LoadLibrary thunk의 주소로 해석된다. 이는 대부분의 경우 접근 위반을 발생시킬 것이다. 따라서 GetProcAddress 함수를 이용해 LoadLibrary의 정확한 메모리 주소를 가져와야 한다. 

CreateRemoteThread를 호출하려면 Kernel32.dll이 로컬과 원격 프로세스 주소 공간 상에서 동일 메모리 위치에 매핑될 것이라는 가정이 필요하다. 모든 애플리케이션은 Kernel32.dll을 필요로 하며, 시스템은 모든 프로세스에 대해 Kernel32.dll을 동일 주소로 매핑한다. 그러나 주소 자체는 ASRL, 주소 공간 랜덤 배치에 의해 임의로 변경될 수 있다. 따라서 다음과 같이 CreateRemoteThread를 호출해야 한다. 

```c++
PTHREAD_START_ROUTINE pfnThreadRtn = (PTHREAD_START_ROUTINE)
    GetProcAddress(GetModuleHandle(TEXT("Kernel32")), "LoadLibraryW");

HANDLE hThread = CreateRemoteThread(hProcessRemote, NULL, 0,
                        pfnThreadRtn, L"C:\\MyLib.dll", 0, NULL);
```

두번째 문제는 DLL 경로명 문자열과 관련된다. C:\\\\MyLib.dll는 

<br/>

# **5. 트로얀 DLL을 이용한 DLL 인젝션**

<br/>

# **6. 디버거를 이용한 DLL 인젝션**

<br/>

# **7. CreateProcess를 이용한 코드 인젝션**

<br/>

# **8. API 후킹**

