---
title: Task 07) Sync-Async IO
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 장치 접근 함수**

## **1) 열기와 닫기**

스레드가 장치로부터의 응답을 대기하지 않으면서도 장치와 통신을 수행하는 것은 중요한 문제이다. 윈도우에서는 장치의 종류와 상관 없이 되도록 거의 동일한 방법으로 읽기, 쓰기를 수행할 수 있게끔 지원한다. 장치 별 장치를 열기 위한 함수의 목록은 아래와 같다. 

|장치|함수|
|----|----|
|파일|CreateFile(경로 이름)|
|디렉토리|CreateFile(디렉토리 이름) + FLAG_BACKUP_SEMANTICS|
|논리적 디스크 드라이브|CreateFile("\\\\.\\드라이브 문자")|
|물리적 디스크 드라이브|CreateFile("\\\\.\\PHYSUCALDRIVE 드라이브 숫자")|
|직렬 포트|CreateFile("COMx")|
|병렬 포트|CreateFile("LPTx")|
|메일슬롯 서버|CreateMailslot("\\\\.\\mailslot\\메일슬롯 이름")|
|메일슬롯 클라이언트|CreateFile("\\\서버 이름\\mailslot\\메일슬롯 이름")|
|네임드 파이프 서버|CreateNamedPipe("\\\\.\\pipe\\파이프 이름")|
|네임드 파이프 클라이언트|CreateFile("\\\\서버 이름\\pipe\\파이프 이름")|
|익명 파이프|CreatePipe|
|소켓|socket, accept, AcceptEx|
|콘솔|CreateConsoleScreenBuffer 혹은 GetStdHandle|

위 함수를 통해 얻은 핸들값을 인자로 특정 함수를 호출함으로서 장치와 통신을 수행할 수 있다. 대부분의 장치에 대해 핸들을 닫을 땐 CloseHandle을 사용한다. 단 소켓의 경우 closesocket을 사용한다. 

## **2) 그 외 장치 접근 함수**

```c++
DWORD GetFileType(HANDLE hDevice);
```
만약 핸들을 갖고 있는데 장치 타입을 모른다면 GetFileType로 넓은 범위의 타입을 알 수 있다. 위 함수는 FILE_TYPE_UNKNOWN, FILE_TYPE_DISK, FILE_TYPE_CHAR, FILE_TYPE_PIPE 중 하나를 반환한다.  

```c++
BOOL SetCommConfig(
    HANDLE          hCommDev,
    LPCOMMOCONFIG   pCC,
    DWORD           dwSize
);
```
시리얼 포트의 전송 속도를 설정하려면 SetCommConfig를 사용한다. 

```c++
BOOL SetMailslotInfo(
    HANDLE      hMailslot,
    DWORD       dwReadTimeout
)
```
데이터를 읽는 동안 대기 만료 시간을 설정하려면 SetMailslotInfo를 사용한다.

## **3) CreateFile**

Createfile을 ㅅ이용하면 새로운 파일을 생성하거나 파일을 포함한 기존 장치에 대해 열기 작업을 수행할 수 있다. 

```c++
HANDLE CreateFile(
    PCTSTR pszName, 
    DWORD dwDesiredAccess,
    DWORD dwShareMode,
    PSECURITY_ATTRIBUTES psa,
    DWORD dwCreationDisposition,
    DWORD dwFlagsAndAttributes,
    HANDLE hFileTemplate
);
```

우선 pszName을 통해서는 특정 장치의 인스턴스 혹은 장치의 타입을 나타내는 값을 전달한다. 

dwDesiredAccess는 장치와 어떻게 데이터를 주고 받을지 결정하는 값으로 0, GENERIC_READ, GENERIC_WRITE, GENERIC_READ | GENETIC_WRITE 이 네가지 값이 제일 많이 사용된다. 0의 경우 데이터를 읽고 쓰는 대신 파일의 타임스탬프를 변경하는 것과 같이 구성 설정만 변경하고자 하는 경우에 사용된다. 

dwShareMode은 장치의 공유 특성을 지정하는데 사용된다. 이미 열기 작업을 수행했는데 다시 CreateFile을 수행한 경우 어떻게 할지 제어하기 위해 사용된다. 일반적으로 0, FILE_SHARE_READ, FILE_SHARE_WRITE, FILE_SHARE_DELETE, FILE_SHARE_READ | FILE_SHARE_WRITE 네가지가 많이 사용된다. 

psa는 보안 정보를 설정하거나 반환되는 핸들을 상속 가능하도록 구성할지 여부를 결정하는 데에 사용된다. 상속 불가능한 기본 보안 특성의 핸들을 얻고자 하면 NULL을 전달하면 된다. 

dwCreationDisposition는 CreateFile을 파일 장치에 대해 사용할 때 큰 의미를 가진다. CREATE_NEW, CREATE_ALWAYS, OPEN_EXISTING, OPEN_ALWAYS, TRUNCATE_EXISTING이 사용될 수 있다. 

dwFlagsAndAttributes는 데이터 송수신 시 통신 플래그 설정 용도, 혹은 파일 특성 설정 용도로 사용된다. 통신 플래그 용도로 사용될 땐 어떤 방식으로 장치에 접근할 것인지에 대해 설정하는 용도로 사용되며, 이 값을 어떻게 설정하냐에 따라 캐시 알고리즘을 최적화 할 수 있다. 

## **3) CreateFile 플래그**

CreateFile로 파일을 다룰 때 사용할 수 있는 캐시 플래그는 아래와 같다. 메일슬롯과 같은 다른 커널 오브젝트에 대한 플래그 정보는 MSDN 문서를 참조하면 된다. 

**FILE_FLAG_NO_BUFFERING**

**FILE_FLAG_SEQUENTIAL_SCAN, FILE_FLAG_RANDOM_ACCESS**

**FILE_FLAG_WRITE_THROUGH**

통신 관련 플래그 중 캐시와 관련 없는 나머지 플래그는 아래와 같다. 


**FILE_FLAG_DELETE_ON_CLOSE**

**FILE_FLAG_BACKUP_SEMANTICS** 

**FILE_FLAG_POSIX_SEMANTICS**

**FILE_FLAG_OPEN_REPARSE_POINT**

**FILE_FLAG_OPEN_NO_RECALL**

**FILE_FLAG_OVERLAPPED**

파일 특성을 지정하는 플래그는 아래와 같다. 

<br/>

# **2. 파일 장치 이용**

## **1) 파일 크기 얻기**

<br/>

# **3. 동기 장치의 IO 수행**

## **1)**

<br/>

# **4. 비동기 장치의 IO 수행**

## **1)**

<br/>

# **5. IO 요청에 대한 완료 통지 수신**

## **1)**

<br/>

