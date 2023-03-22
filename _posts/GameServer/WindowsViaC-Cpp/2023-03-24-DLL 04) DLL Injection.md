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

<br/>

# **4. 원격 스레드를 이용한 DLL 인젝션**

<br/>

# **5. 트로얀 DLL을 이용한 DLL 인젝션**

<br/>

# **6. 디버거를 이용한 DLL 인젝션**

<br/>

# **7. CreateProcess를 이용한 코드 인젝션**

<br/>

# **8. API 후킹**

