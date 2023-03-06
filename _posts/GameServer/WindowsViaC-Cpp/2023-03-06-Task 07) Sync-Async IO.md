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
|물리적 디스크 드라이브|CreateFile("\\\\.\\PHYSUCALDRIVE드라이브 숫자")|
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

아래 나오는 플래그는 파일에 대한 플래그로 메일슬롯과 같은 다른 커널 오브젝트에 대한 플래그 정보는 MSDN 문서를 참조하면 된다. 

CreateFile로 파일을 다룰 때 사용할 수 있는 캐시 관련 플래그는 아래와 같다. 

**FILE_FLAG_NO_BUFFERING**

시스템은 성능 개선을 위해 항당 데이터를 캐시하는데 이 플래그를 사용하면 파일에 접근할 때 버퍼링을 수행하지 않는다. 버퍼를 복사하는 대신 파일에 직접 쓰기 때문에 수행하고자 하는 작업에 따라 속도와 메모리 사용량을 개선시킬 수도 있다. 

특수한 목적이 있는 게 아니라면 가능한 사용하지 않는 게 좋으며, 만약 사용한다면 파일에 접근할 때 항상 디스크 볼륨 섹터 크기의 배수 단위로 IO 수행 위치를 지정해야 한다. 매우 큰 파일을 다룰 때 캐시 매니저가 내부 자료구조 저장을 위해 충분한 공간을 할당받지 못할 때가 있는데 그런 상황에서 이 플래그를 사용하기도 한다. 

**FILE_FLAG_SEQUENTIAL_SCAN, FILE_FLAG_RANDOM_ACCESS**

시스템이 파일 데이터에 대한 버퍼링을 수행하는 경우 유용하다. 

FILE_FLAG_SEQUENTIAL_SCAN를 사용하면 사용자가 파일을 순차적으로 접근할 것이라 인지하여 필요한 크기보다 더 많은 데이터를 읽어들인다. 이를 통해 하드 디스크 접근 횟수를 줄여 성능을 향상시킨다. 

FILE_FLAG_RANDOM_ACCESS는 반대로 사용자가 파일에 임의 접근할 것이라 인지하여 파일 데이터를 미리 읽지 않도록 한다.  파일 접근 위치를 자주 이동한다면 이 플래그를 지정하는 것이 좋다. 

**FILE_FLAG_WRITE_THROUGH**

이 플래그를 사용하면 파일에 데이터를 쓸 때 데이터 손실 가능성을 줄이기 위해 중간 캐싱 기능을 사용하지 않고 변경사항을 디스크에 직접 쓰게 된다. 읽기는 여전히 내부적인 캐시를 통해 진행된다. 이 플래그를 네트워크 파일 서버의 파일을 열 때 사용할 경우 윈도우는 파일 서버의 디스크 드라이브 상에 데이터를 완전히 다 쓸 때까지 파일 쓰기 함수를 반환하지 않는다. 

통신 관련 플래그 중 캐시와 관련 없는 나머지 플래그는 아래와 같다. 

**FILE_FLAG_DELETE_ON_CLOSE**

이 플래그를 쓸 경우 파일과 관련된 모든 핸들이 닫히면 파일이 자동으로 삭제된다. FILE_ATTRIBUTE_TEMPORARY 플래그와 주로 같이 사용되는데 이 두 플래그를 동시에 사용할 경우 임시 파일을 생성하고 쓰고 읽고 닫을 수 있다. 

**FILE_FLAG_BACKUP_SEMANTICS** 

이 플래그는 주로 백업, 복원 소프트웨어에서 사용되는데 시스템이 호출자의 접근 토큰이 파일과 폴더에 대해 백업, 복원 권한을 갖고 있는지 확인한다.  

**FILE_FLAG_POSIX_SEMANTICS**

이 플래그를 이용하면 파일을 생성하거나 열 때 대소문자를 구분하도록 한다. 이때 이 플래그를 사용하여 생성한 파일은 다른 윈도우 애플리케이션에서 접근하지 못할 수도 있으므로 주의해야 한다. 

**FILE_FLAG_OPEN_REPARSE_POINT**

이 플래그를 사용하면 시스템에게 파일의 reparse 특성을 무시할 것을 요청한다. reparse 특성을 사용하면 파일 시스템 필터가 파일에 대한 열기, 읽기, 쓰기, 닫기 동작을 변경할 수 있는데 이는 반드시 필요한 부분이므로 되도록 해당 플래그는 사용하지 않는 것이 좋다. 

**FILE_FLAG_OPEN_NO_RECALL**

기본적으로는 파일이 오랫동안 사용되지 않는 경우 시스템이 파일 내용을 오프라인 저장 장치로 옮겨서 하드 디스크의 공간을 확보하는데, 이는 파일 정보가 하드 디스크에서 완전히 제거되는 것이 아니며 파일의 내용만 제거되는 것이다. 나중에 파일을 다시 열면 시스템이 오프라인 저장장치로부터 파일의 내용을 자동으로 복원한다.

그러나 이 플래그를 사용할 경우 테이프와 같은 오프라인 저장 장치로부터 하드 디스크와 같은 온라인 저장 장치로 파일을 복원하지 못하도록 하기 때문에 오프라인 저장소에 대해 IO 작업이 수행되지 않는다. 

**FILE_FLAG_OVERLAPPED**

장치 열기는 기본적으로 동기 IO로 수행되기 때문에 파일로부터 데이터가 모두 읽혀질 대까지 스레드가 대기 상태를 유지하다가, 읽기가 완료되면 비로소 수행을 계속 하게 된다. 그러나 장치 IO는 느린 작업에 속하기 때문에 이 플래그를 이용해 장치에게 비동기적으로 접근하길 원한다고 알려줄 수 있다. 그러면 운영체제가 사용자 스레드를 대신해 IO 작업을 수행하고 완료되었을 때 통보해준다. 이러한 비동기 IO 작업은 고성능 애플리케이션 작성의 핵심이 된다. 

dwFlagsAndAttribute 매개변수를 이용하면 파일 특성을 지정할 수 있는데 이때 사용되는 플래그는 아래와 같다. 

**FILE_ATTRIBUTE_ARCHIVE**

파일의 보관 특성을 지정하며 백업이나 제거를 위해 사용한다. CreateFile로 파일을 생성할 경우 이 플래그는 자동으로 설정된다. 

**FILE_ATTRIBUTE_ENCRYPTED**

파일이 암호화되었음을 나타낸다. 

**FILE_ATTRIBUTE_HIDDEN**

일반 파일 목록 나열 시 나타나지 않는 숨겨진 파일임을 나타낸다. 

**FILE_ATTRIBUTE_NORMAL**

특별한 설정이 없는 파일로, 다른 특성과 함께 사용할 수 없다. 

**FILE_ATTRIBUTE_NOT_CONTENT_INDEXED**

인덱싱 서비스에 의해 인덱싱 되지 않을 것임을 나타낸다. 

**FILE_ATTRIBUTE_OFFLINE**

파일이 존재하지만 오프라인 저장장치로 옮겨졌음을 나타낸다. 이 플래그는 계층적 저장 시스템에서 유용하게 사용될 수 있다. 

**FILE_ATTRIBUTE_READONLY**

읽기 전용 파일로 쓰거나 삭제할 수 없다. 

**FILE_ATTRIBUTE_SYSTEM**

운영체제를 구성하는 파일임을 나타내며 읽을 수는 있으나 쓰거나 삭제할 수 없다. 

**FILE_ATTRIBUTE_TEMPORARY**

단시간에 걸쳐 사용되는 임시 파일로 파일에 대한 접근 속도 개선을 위해 디스크 대신 램에 파일의 내용을 유지하기 위해 최대한 노력한다. 

<br/>

# **2. 파일 장치 이용**

## **1) 파일 크기 얻기**

```c++
BOOL GetFileSizeEx(
    HANDLE hFile,
    PLARGE_INTEGER pliFileSize
);

BOOL GetCompressedFileSize(
    PCTSTR pszFileName,
    PDWORD pdwFileSizeHigh
);
```
GetFileSizeEx는 파일의 논리적인 크기를, GetCompressedFileSize는 물리적인 크기를 반환한다. 예를 들어 100KB 파일이 85KB로 압축되어 있다면 GetFileSizeEx는 100KB를, GetCompressedFileSize는 85KB를 반환하는 것이다.

## **2) 파일 포인터 위치 지정**

파일 커널 오브젝트는 내부적으로 파일 포인터를 갖고 있다. 파일 포인터는 64bit 오프셋 값으로 동기적인 IO를 수행할 위치 정보를 가지고 있다. 동일한 파일을 여러번 CreateFile할 경우 각각의 파일 커널 오브젝트는 독립적인 파일 포인터를 가진다. 파일의 임의 위치에 접근하려는 경우 SetFilePointerEx를 호출한다. 

이때 몇가지 주의 사항이 있다. 파일 포인터를 파일 크기보다 더 크게 설정할 수도 있는데, 이동된 위치에 내용을 쓰거나 SetEndOfFile을 호출할 경우 파일의 크기가 수정된다. 또, FILE_FLAG_NO_BUFFERING으로 파일을 열었을 경우 섹터 크기로 정렬된 위치로만 파일 포인터를 이동할 수 있다. 마지막으로  SetFilePointerEx의 이동 크기를 0으로 설정하면 현재 파일 포인터 위치를 가져올 수 있게 된다. 

<br/>

# **3. 동기 장치의 IO 수행**

## **1) 데이터 읽고 쓰기**

ReadFile, WriteFile 함수로 데이터를 읽고 쓸 수 있다. 이때 장치를 여는 함수에서 FILE_FLAG_OVERLAPPED 를 사용하면 비동기 IO를 기대하기 때문에 이 플래그는 사용해서는 안된다. 두 함수는 성공적으로 수행되었을 때 TRUE를 반환한다. 

## **2) 데이터 flush**

캐시된 데이터를 장치로 flush 하기 위해 FlushFileBuffers 함수를 사용할 수 있다. 이 함수는 캐시된 데이터를 강제로 장치에 쓰도록 만들며 성공할 경우 TRUE를 반환한다. 이를 수행하기 위해서는 GENERIC_WRITE 플래그를 포함하여 장치를 열어야 한다.  

## **3) 동기 IO의 취소**

동기 IO 함수는 요청한 작업이 완료될 때까지 스레드와 스레드가 생성한 모든 윈도우가 같이 정지된다. 만약 동기적으로 함수를 호출했는데 IO에 너무 오랜 시간이 소요된다면 동기 IO를 강제로 취소하여 스레드가 계속 수행될 수 있도록 하는 것이 최상의 방법이 될 수 있다.

윈도우 비스타 기준 CancleSynchronousIo에 스레드의 핸들을 넣어 호출할 경우 동기 IO를 취소할 수 있다. 이때 스레드의 핸들은 THREAD_TERMINATE 접근 권한을 가진 형태로 생성되어야 한다. 그러나 이 함수 호출 시점에서 스레드가 실제로 동기 IO 작업을 진행하고 있는지 알아낼 방법이 없기 때문에 주의해야 한다. 이 경우 FALSE, ERROR_NOT_FOUND를 반환하게 된다. 

<br/>

# **4. 비동기 장치의 IO 수행**

## **1) 비동기 IO 기본**

IO는 상대적으로 가장 느리고 예측하기 어려운 작업 중 하나이기 때문에 적절하게 비동기적으로 사용할 경우 애플리케이션의 성능을 높일 수 있다. 비동기적으로 접근하기 위해서는 CreateFile을 호출할 때 FILE_FLAG_OVERLAPPED 플래그를 전달해야 한다. 

이후에는 동일하게 ReadFile, WriteFile을 이용하면 앞서 전달한 플래그를 확인하여 비동기 IO를 수행하게 된다. 이 경우 pdwNumBytes에는 일반적으로 NULL을 전달하는데 비동기 IO는 IO 작업 완료 이전에 함수가 반환되기 때문에 함수 반환 시점의 NumBytes는 의미가 없기 떄문이다. 

## **2) OVERLAPPED 구조체**

비동기 IO를 수행하려면 pOvelapped 매개변수로 OVERLAPPED 구조체 주소를 전달해야 한다. 해당 구조체는 에러코드, 전송 바이트 수, 파일 오프셋, 핸들을 포함한다. 비동기 IO에서는 커널 오브젝트 내 파일 포인터를 직접 사용할 수 없는데, 오프셋은 이를 대신하여 IO 작업의 시작 위치를 지정하기 위해 사용된다. 또 에러 코드와 전송된 바이트 수는 디바이스 드라이버에 의해 설정되어 IO 작업 완료 여부 확인을 위해 사용된다. 

## **3) 비동기 IO 유의사항**

첫째로, 디바이스 드라이버는 비동기 IO의 처리 순서에 있어서 선입 선출을 보장하지 않는다. 수행 성능 개선을 위해 순서에 따라 수행하는 대신 하드 디스크로부터 물리적으로 가장 가까운 위치에 대한 IO 요청을 먼저 수행할 수 있다. 

둘째로, 에러 확인 방법을 알고 있어야 한다. ReadFile, WriteFile은 IO 요청이 동기적으로 수행되는 경우 0이 아닌 값을 반환한다. 그러나 비동기적으로 수행되는 경우나 에러가 발생하면 둘 다 FALSE를 반환한다. 따라서 FALSE가 반환되었을 때 어떤 상황인지 확인하기 위해 GetLastError를 호출해야 한다. ERROR_IO_PENDING을 반환한다면 IO 요청이 성공적으로 전달된 것이다. 

만약 요청 리스트가 꽉 차서 에러가 난 경우 ERROR_INVALID_USER_BUFFER, ERROR_NOT_ENOUGH_MEMORY 중 하나를 반환한다. 또, 몇몇 장치는 버퍼가 잠긴 페이지 내에 존재하여 RAM 외부로 스와핑 되지 않을 것을 요구하는데, 잠글 수 있는 제한 크기를 넘어설 경우 ERROR_NOT_ENOUGH_QUOTA를 반환한다. 이 크기는 SetProcessWorkingSetSize로 변경할 수 있다. 위와 같은 에러들은 진행 중인 IO 요청이 일부 완료된 이후 재호출했을 때 해결되기도 한다.  

비동기 IO에서 쓰이는 버퍼, OVERLAPPED 구조체는 IO 요청이 완료될 때까지 옮기거나 삭제해선 안된다. 디바이스 드라이버로 IO 요청이 전달될 때는 주소가 전달되는 것이지 실제 블록이 전달되는 것은 아니다. 이를 제대로 인지하지 않을 경우 아래와 같은 실수가 생길 수 있다. 

```c++
VOID ReadData(HANELD hFile)
{
    OVERLAPPED o = { 0 };
    BYTE b[100];
    ReadFile(hFIle, b, 100, NULL, &o);
}
```
이 코드의 문제점은 비동기 요청 삽입 이후 함수가 즉각 반환된다는 것이다. 함수가 반환되면 스택에 있던 데이터 버퍼와 OVERLAPPED 구조체가 삭제되지만, 디바이스 드라이버는 이를 알지 못하기 때문에 삭제된 스택 내의 주소를 계속해서 사용하게 된다. 이는 메모리 오염의 원인이 되며, 메모리 수정 시점이 비동기적으로 이루어지기 때문에 원인을 발견하기 어려워진다. 

## **4) 장치 IO의 취소**

IO 요청을 처리하기 전에 작업을 취소하고 싶은 경우, 핸들이 IO 컴플리션 포트와 연결된 게 아니라면 CancelIo 함수를 호출할 수 있다. 삽입된 모든 IO 요청을 완전히 취소하고 싶다면 장치에 대한 핸들을 닫으면 되고, 특정 장치에 대해 하나의 IO 요청만 취소하고 싶은 경우 CancelIoEx를 사용하면 된다. CancelIoEx는 다른 스레드가 삽입한 IO 요청도 취소할 수 있으며, 만약 pOverlapped 매개변수로 NULL을 전달하면 hFile이 가리키는 장치에 대해 모든 IO를 취소할 수도 있다. 이때 요청이 취소되면 ERROR_OPERATION_ABORTED 에러코드를 반환 받는다. 

<br/>

# **5. IO 요청에 대한 완료 통지 수신**

## **1) 디바이스 커널 오브젝트의 시그널링**

디바이스 커널 오브젝트도 다른 커널 오브젝트와 마찬가지로 signal, non-signal 상태를 가지며, 동일하게 WaitFor 함수를 통해 이를 동기화에 사용할 수 있다. 이 방법은 간단하고 직관적이며 특정 스레드가 IO 요청을 삽입하고 다른 스레드가 완료 통지를 수신할 수 있다. 

그러나 이 방법은 단일 장치에 대해 다수의 IO 요청을 수행하는 경우에는 적합하지 않다. CreaetFile에서 반환한 하나의 파일 핸들을 이용해 Read, Write 작업을 실행하고 WaitFor 함수를 호출한다면 그 중 어떤 작업이 완료된 것인지 구분할 수 없기 때문이다.   

## **2) 이벤트 커널 오브젝트의 시그널링**

OVERLAPPED의 hEvent는 이벤트 핸들을 저장할 수 있다. IO 요청이 완료되면 디바이스 드라이버는 hEvent 멤버의 NULL 여부를 확인하여 NULL이 아닐 경우 이 값을 인자로 SetEvent를 호출하고 디바이스 오브젝트를 다시 signal로 만든다. 이 시그널을 통해 단일 장치에 대해 다수의 IO 요청을 수행할 수 있다. 만일 여러번의 비동기 장치 IO 요청을 동시에 수행하고자 한다면 각 요청마다 서로 다른 이벤트 커널 오브젝트를 생성해야 한다. 이 역시 특정 스레드가 IO 요청을 삽입하고 다른 스레드가 완료 통지를 수신할 수 있다. 

## **3) Alertable IO**

스레드가 생성되면 시스템은 스레드별로 APC Queue, 비동기 프로시저 콜 큐를 하나씩 생성한다. ReadFileEx, WriteFileEx 함수를 사용하면 디바이스 드라이버에게 IO 작업 완료 통지를 APC 큐에 삽입해줄 것을 요청할 수 있다. 

이 함수들은 ReadFile, WriteFile과 유사하지만 바이트 수를 돌려받는 매개변수가 없으며 이를 얻기 위해서는 콜백함수를 사용해야 한다. 대신 이 함수들은 Completion Routine이라는 콜백함수의 주소를 필요로 한다. 

```c++
VOID WINAPI CompletionRoutine(
    DWORD dwError,
    DWORD dwNumBytes,
    OVERLAPPED* po
);
```
컴플리션 루틴은 위와 같은 형태로 구성된다. ReadFileEx, WriteFileEx가 비동기 IO를 수행하면 디바이스 드라이버에게 컴플리션 루틴 주소 값을 전달하고, IO 요청을 마치면 APC 큐에 완료 통지 항목을 추가하는데 이 항목에 이러한 컴플리션 루틴의 주소와 최초 IO 요청시에 사용한 OVERLAPPED 구조체 주소가 포함되어 있다. 

만약 Alertable IO를 사용하면 디바이스 드라이버가 이벤트 커널 오브젝트 시그널링을 시도하지 않는다. 스레드가 Alertable 상태가 되면 시스템은 APC 큐를 확인하여 삽입된 모든 항목에 대해 컴플리션 루틴을 호출한다. 이때 IO 에러코드, 송수신 바이트 수, OVERLAPPED 구조체 주소가 전달된다. 

APC 큐에 항목이 삽입되었다고 해서 바로 컴플리션 루틴이 호출되지는 않는다. 우선 스레드가 자신을 Alertable 상태로 변경해야 하며, 이를 통해 스레드가 인터럽트 가능한 상태임을 알려야 한다. 이때 스레드를 Alertable 하게 변경할 수 있는 함수로는 SleepEx, WaitForSingleObjectEx, WaitForMultipleObjectsEx, SignalObjectAndWait, GetQueuedCompletionStatusEx, MsgWaitForMultipleObjectsEx가 있다. 

이 중 하나를 사용하면 그때 시스템이 APC 큐에 항목이 존재하는지 확인한다. 만일 큐에 여러개의 항목이 존재하면 스레드를 대기 상태로 전환하지 않고 APC 큐에 있는 항목들을 하나씩 빼내어 콜백 루틴을 호출하고, APC 큐가 완전히 비워지면 함수가 WAIT_IO_COMPLETION을 반환한다. 어떤 항목도 없는 상태에서 함수를 호출할 경우 스레드는 정지 상태가 되며 APC 큐에 항목이 삽입되었을 때 수행을 재개하게 된다. 

이러한 방법은 단일 장치에 대해 다수의 IO가 요청을 수행할 수 있다. 그러나 콜백함수를 반드시 필요로 한다는 점, 그로 인해 많은 전역 변수를 사용해야 된다는 점, IO 작업을 요청한 스레드가 반드시 완료 통지를 처리해야 한다는 점 등으로 인해 확장성 있는 애플리케이션을 만들기에 한계가 있다. 

## **4) IO Completion Port**

전통적으로 서비스 애플리케이션은 Serial Model과 Concurrent Model 중 하나의 형태로 설계되어왔다. Serial Model은 하나의 스레드가 사용자의 요청을 대기하다가, 요청이 들어오면 스레드가 깨어나 클라이언트의 요청을 처리하는 방식이다. Concurrent Model은 하나의 스레드가 사용자의 요청을 대기하다가, 요청이 들어오면 요청 처리를 위한 스레드를 새로 생성하고 다시 다른 사용자의 요청을 기다리는 방식이다.

시리얼 모델은 하나의 요청이 완전히 처리될 때까지 대기해야 한다는 한계 때문에 핑 서버와 같이 요청 수가 작은 간단한 애플리케이션에서 적절한 모델이다. 반면 컨커런트 모델은 요청을 기다리는 스레드의 경우 최소한의 작업만 수행하며 대부분을 대기 상태로 보내게 된다. 이 경우 요청을 처리하는 독립된 스레드가 확보되기 때문에 확장에도 용이하고 더 좋은 성능을 보일 수 있다. 

그러나 컨커런트 모델의 애플리케이션을 윈도우에서 구현할 때, 많은 수의 요청이 들어오게 되면 수많은 스레드들이 만들어지고 그것들이 스케줄 가능 상태에 있으면서 스레드 생성과 컨텍스트 전환에 너무 많은 시간이 소요된다는 문제점이 생겼다. 그리고 마이크로소프트에서는 이러한 상황을 개선하기 위해 IO Completion Port 커널 오브젝트를 만들었다. 

IO Completion Port의 이론적 배경은 동시에 수행할 수 있는 스레드 개수의 상한을 설정하고 스레드 생성에 드는 시간을 절약해야 한다는 것이다. 따라서 IO Completion Port는 스레드 풀을 이용하도록 설계되었다. IO Completion Port는 상당히 복잡한 커널 오브젝트로, CreateIoCompletionPort로 생성할 수 있다. 

```c++
HANDLE CreateIoCompletionPort(
    HANDLE hFile,
    HANDLE hExistingCompletionPort,
    ULONG_PTR CompletionKey,
    DWORD dwNumberOfConcurrentThreads
);
```
이 함수는 IO 컴플리션 포트 생성, 장치와 IO 컴플리션 포트의 연계 두가지 작업을 수행한다. 이 함수는 윈도우의 커널 오브젝트 생성 함수 중 유일하게 SECURITY_ATTRIBUTES 구조체의 포인터를 인자로 전달할 필요가 없는 함수인데, 이는 IO 컴플리션 포트가 단일 프로세스 내에서만 수행되도록 하기 위함이다. 

```c++
HANDLE CreateNewCompletionPort(DWORD dwNumberOfConcurrentThreads )
{
    return (CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, dwNumberOfConcurrentThreads));
}
```

만약 포트 생성 작업만 하고 싶다면 위와 같이 래핑할 수 있다.  dwNumberOfConcurrentThreads를 0으로 전달할 경우 그 값은 머신에 설치된 CPU의 개수로 설정된다. 이와 같이 CPU 개수만큼의 스레드를 활용하게 되면 스레드 간의 컨텍스트 전환을 막을 수 있다.


단일 장치에 대해 다수의 IO가 요청을 수행할 수 있다. 특정 스레드가 IO 요청을 삽입하고 다른 스레드가 완료 통지를 수신할 수 있다. 유연성과 확장성이 가장 뛰어난 방법이다. 

## **5) IO Completion Port와 스레드 풀 관리**