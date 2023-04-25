---
title: Socket Model 2) Overlapped, IOCP Model
categories: TCPIPWindowSocket
tags: 
toc: true
toc_sticky: true
---

# **1. 소켓 IO 모델**

## **1) 동기와 비동기**

동기 IO는 IO 함수 호출 시 작업이 끝날 때까지 대기하다가 반환하는 처리 방식을 의미한다. Select, WSAAsyncSelect, WSAEventSelect 모두 동기 IO로 소켓을 처리하되 OS가 함수 반환 시점을 알려줘서 편하게 IO를 처리할 수 있도록 한 모델이다. 즉, 동기 IO와 비동기 통지를 결합한 것이다. 

반면 비동기 IO는 IO 함수 호출 후 작업 완료 여부와 무관하게 다른 작업을 할 수 있다. 작업이 끝나면 OS가 이를 알려주며, 그때 작업을 중지하고 결과를 처리한다. Overlapped, IOCP가 이 방식에 해당하며 이는 비동기 IO와 비동기 통지를 결합했다고 볼 수 있다. 

## **2) 모델들의 비교**

Select, AsyncSelect, EventSelect, Overlapped, IOCP 여섯가지 모델 모두 소켓 함수 호출 시 블로킹을 최소화하며, 스레드를 일정 수준으로 유지할 수 있도록 돕는다. 모델 별 장단점은 아래와 같다. 

|모델|장점|단점|
|---|----|----|
|Select|윈도우, 리눅스에서 모두 사용할 수 있어서 이식성이 높다.|모델 중 성능이 가장 떨어지며 스레드 당 최대 64개의 소켓만 처리할 수 있다.|
|AsyncSelect|소켓 이벤트를 윈도우 메시지로 처리하므로 GUI 응용 프로그램과 잘 결합한다. MFC 소켓 클래스는 내부적으로 이 모델을 사용한다.|단일 윈도우 프로시저에서 일반 윈도우 메시지와 소켓 메시지를 처리하다보니 성능 저하가 발생한다|
|EventSelect|Select, AsynSelect의 특성을 혼합한 모델로, 윈도우를 필요로 하지 않으면서도 비교적 뛰어난 성능을 제공한다.|스레드 당 최대 64개의 소켓만 처리할 수 있다.|
|Overlapped|비동기 IO를 통해 뛰어난 성능을 제공한다.|Event 방식의 경우 스레드 당 최대 64개의 소켓만 처리할 수 있고, Callback 방식의 경우 Callback을 지원하지 않는 함수에서 사용할 수 없다.|
|IOCP|IOCP와 비동기 IO를 통해 가장 뛰어난 성능을 제공한다.|성능 면에서는 특별한 단점이 없지만 비교적 코딩이 복잡하다.|

일반적으로 비동기 IO가 동기 IO보다 좋은 성능을 보인다. 이는 비동기 IO일 때 CPU 명령 수행과 IO 작업을 병행할 수 있고, 유저 모드와 커널 모드 전환을 최소화할 수 있기 때문이다. 비동기 IO는 버퍼가 가득 찼을 때 응용 프로그램 버퍼를 잠근 후 해당 메모리에 직접 데이터를 쓴다. 따라서 커널 영역에서의 유저 영역 복사가 일어나지 않고, 모드 전환 없이 입출력 전환이 이루어질 수 있다.

<br/>

# **2. Overlapped 모델**

## **1) 작동 방식**

Overlapped 모델은 기존에 윈도우에서 파일 IO를 위해 제공하던 Overlapped IO를 소켓 IO에서도 사용할 수 있게끔 만든 것이다. 이 모델은 기본적으로 비동기 IO를 지원하는 소켓 생성, 비동기 IO를 지원하는 소켓 함수 호출, OS가 작업 완료 통지 시 IO 결과 처리 세 단계를 따른다. 

완료 통지는 이벤트 객체 혹은 Callback 함수를 통해 이루어진다. 전자의 경우 소켓 IO 작업이 완료되면 OS가 등록된 이벤트 객체를 신호 상태로 바꾸며, 이를 관찰하여 완료를 통지 받는다. 후자는 작업이 완료되면 OS가 등록된 함수를 호출하여 통지하는데, 이 함수를 Callback 함수 혹은 완료 루틴이라 부른다.

## **2) 구성 요소**

```c++
int WSASend
(
    SOCKET s,
    LPWSABUF lpBuffers,
    DWORD dwBufferCount,
    LPDWORD lpNumberOfBytesSent,
    DWORD dwFlags, 
    LPWSAOVERLAPPED lpOverlapped,
    LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine
)

int WSARecv
(
    SOCKET s,
    LPWSABUF lpBuffers,
    DWORD dwBufferCount,
    LPDWORD lpNumberOfBytesRecvd,
    DWORD dwFlags, 
    LPWSAOVERLAPPED lpOverlapped,
    LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine
)
```

이 모델에서 사용되는 핵심 함수는 Send, Recv를 비동기적으로 처리하는 WSASend(), WSARecv()이다. WSASend, WSARecv는 WSABUF를 버퍼로 사용하며 Scatter, Gather 입출력을 지원한다. WSABUF 구조체를 송신 측에서 사용하면 여러 버퍼에 저장된 데이터를 모아서 전송해주며, 수신 측에서 사용하면 받은 데이터를 여러 버퍼에 흩뜨려 저장한다. 

순서대로 소켓, 송수신 버퍼의 시작 주소와 길이, 성공 시 송수신 바이트 수를 적을 공간, 옵션 플래그 그리고 이벤트 객체를 전달할 수 있는 OVERLAPPED 인자와 Callback 함수 포인터를 전달할 수 있는 COMPLETION_ROUTINE 인자가 있다. 이때 완료 루틴 인자의 우선순위가 더 높기 때문에, 그 값에 NULL이 들어가야 OVERLAPPED 인자로 전달한 이벤트 객체가 사용된다. 

```c++
typedef struct _WSAOVERLAPPED
{
    DWORD Internal;
    DWORD InternalHigh;
    DWORD Offset;
    DWORD OffsetHigh;
    WSAEVENT hEvent;

} WSAOVERLAPPED, *LPWSAOVERLAPPED;
```
OVERLAPPED 구조체의 멤버는 위와 같다. 이는 비동기 IO를 위한 정보를 전달할 때 사용하며 앞의 네가지는 OS가 내부적으로 사용한다. hEvent는 이벤트 방식으로 통지 받고 싶을 때 이벤트 객체의 핸들을 설정하기 위해 사용한다. 

## **3) Event 방식**

소켓 생성 시 WSACreateEvent로 소켓에 대응하는 이벤트 객체를 생성한다. 이후 비동기 IO 함수를 호출할 때 이벤트 객체의 핸들 값을 전달한다. 작업이 완료되지 않으면 WSA_IO_PENDING 오류 코드가 반환된다. 작업이 완료되면 OS가 이벤트를 신호 상태로 만들고 WSAWaitForMultipleEvents가 반환되는데, 이때 WSAGetOverlappedResult()를 통해 IO 결과를 확인할 수 있다. 

이를 바탕으로 구현한 예제는 아래와 같다. 

```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <stdio.h>

#define SERVERPORT 9000
#define BUFSIZE 512

void HandleError(int line)
{
	int err = ::WSAGetLastError();
	wprintf(L"Error! Line %d: %d", line, err);
}
#define _HandleError HandleError(__LINE__)

// Socket Info
struct SOCKETINFO
{
	WSAOVERLAPPED overlapped;
	SOCKET sock;
	char buf[BUFSIZE + 1];
	int recvbytes;
	int sendbytes;
	WSABUF wsabuf;
};

int sockCnt = 0;
SOCKETINFO* SocketInfoArray[WSA_MAXIMUM_WAIT_EVENTS];
WSAEVENT EventArray[WSA_MAXIMUM_WAIT_EVENTS];
CRITICAL_SECTION cs;

DWORD WINAPI WorkerThread(LPVOID arg)
{
	int ret;

	while (1)
	{
		DWORD index = WSAWaitForMultipleEvents(
			sockCnt, EventArray, FALSE, WSA_INFINITE, FALSE);
		if (index == WSA_WAIT_FAILED) continue;
		index -= WSA_WAIT_EVENT_0;
		WSAResetEvent(EventArray[index]);
		if (index == 0) continue;

		SOCKADDR_IN clientaddr;
		WCHAR szClientIP[16] = { 0 };
		int addrlen = sizeof(clientaddr);
		SOCKETINFO* ptr = SocketInfoArray[index];
		getpeername(ptr->sock, (SOCKADDR*)&clientaddr, &addrlen);

		DWORD cbTransferred, flags;
		ret = WSAGetOverlappedResult(ptr->sock,
			&ptr->overlapped, &cbTransferred, FALSE, &flags);

		if (ret == FALSE || cbTransferred == 0)
		{
			RemoveSocketInfo(index);
			wprintf(L"\n[TCP Server] Disconnect Client: IP=%s, port=%d\n",
				szClientIP, ntohs(clientaddr.sin_port));
			continue;
		}

		if (ptr->recvbytes == 0)
		{
			ptr->recvbytes = cbTransferred;
			ptr->sendbytes = 0;
			ptr->buf[ptr->recvbytes] = '\0';
			wprintf(L"\n[TCP Server] IP=%s, port=%d: ",
				szClientIP, ntohs(clientaddr.sin_port));
			printf("%s\n", ptr->buf);
		}
		else
		{
			ptr->sendbytes += cbTransferred;
		}

		if (ptr->recvbytes > ptr->sendbytes)
		{
			ZeroMemory(&ptr->overlapped, sizeof(ptr->overlapped));
			ptr->overlapped.hEvent = EventArray[index];
			ptr->wsabuf.buf = ptr->buf + ptr->sendbytes;
			ptr->wsabuf.len = ptr->recvbytes - ptr->sendbytes;

			DWORD sendbytes;

			// Send Data to Client
			ret = WSASend(ptr->sock, &ptr->wsabuf, 1, 
					&sendbytes, 0, &ptr->overlapped, NULL);

			if (ret == SOCKET_ERROR)
			{
				if (WSAGetLastError() != WSA_IO_PENDING)
					_HandleError;	
				continue;
			}
		}
		else
		{
			ptr->recvbytes = 0;
			ZeroMemory(&ptr->overlapped, sizeof(ptr->overlapped));
			ptr->overlapped.hEvent = EventArray[index];
			ptr->wsabuf.buf = ptr->buf;
			ptr->wsabuf.len = BUFSIZE;

			DWORD recvbytes;
			flags = 0;

			// Send Data to Client
			ret = WSARecv(ptr->sock, &ptr->wsabuf, 1,
				&recvbytes, &flags, &ptr->overlapped, NULL);

			if (ret == SOCKET_ERROR)
			{
				if (WSAGetLastError() != WSA_IO_PENDING)
					_HandleError;
				continue;
			}
		}
	}

}

BOOL AddSocketInfo(SOCKET sock)
{
	EnterCriticalSection(&cs);
	if (sockCnt >= WSA_MAXIMUM_WAIT_EVENTS) return FALSE;

	SOCKETINFO* ptr = new SOCKETINFO;
	if (ptr == NULL) return FALSE;

	WSAEVENT hEvent = WSACreateEvent();
	if (hEvent == WSA_INVALID_EVENT) return FALSE;
	
	ZeroMemory(&ptr->overlapped, sizeof(ptr->overlapped));
	ptr->overlapped.hEvent = hEvent;
	ptr->sock = sock;
	ptr->recvbytes = ptr->sendbytes = 0;
	ptr->wsabuf.buf = ptr->buf;
	ptr->wsabuf.len = BUFSIZE;
	SocketInfoArray[sockCnt] = ptr;
	EventArray[sockCnt] = hEvent;
	sockCnt++;

	LeaveCriticalSection(&cs);
	return TRUE;
}

void RemoveSocketInfo(int idx)
{
	EnterCriticalSection(&cs);
	SOCKETINFO* ptr = SocketInfoArray[idx];
	closesocket(ptr->sock);
	delete ptr;
	WSACloseEvent(EventArray[idx]);

	if (idx != (sockCnt - 1))
	{
		SocketInfoArray[idx] = SocketInfoArray[sockCnt - 1];
		EventArray[idx] = EventArray[sockCnt - 1];
	}

	sockCnt--;

	LeaveCriticalSection(&cs);
}

int main(int argc, char* argv[])
{
	int ret;
	int err;

	// Initialize
	InitializeCriticalSection(&cs);
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create Socket
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET)
	{
		_HandleError;
		return 0;
	}

	// Bind Socket
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, L"0.0.0.0", &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);
	ret = bind(listen_sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Socket to Listen
	ret = listen(listen_sock, SOMAXCONN);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Network Event
	WSAEVENT hEvent = WSACreateEvent();
	if (hEvent == WSA_INVALID_EVENT)
	{
		_HandleError;
		return 0;
	}
	EventArray[sockCnt] = hEvent;
	sockCnt++;

	HANDLE hThread = CreateThread(NULL, 0, WorkerThread, NULL, 0, NULL);
	if (hThread == NULL) return 1;
	CloseHandle(hThread);
	
	int addrlen;
	SOCKET client_sock;
	SOCKADDR_IN clientaddr;
	DWORD recvbytes, flags;

	while (1)
	{
		// Accept Client Socket
		addrlen = sizeof(clientaddr);
		client_sock = accept(listen_sock, (SOCKADDR*)&clientaddr, &addrlen);
		if (client_sock == INVALID_SOCKET)
		{
			_HandleError;
			break;
		}

		WCHAR szClientIP[16] = { 0 };
		InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
		wprintf(L"\n[TCP Server] Accept Client: IP=%s, port=%d\n",
			szClientIP, ntohs(clientaddr.sin_port));

		if (AddSocketInfo(client_sock) == FALSE)
		{
			closesocket(client_sock);
			InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
			wprintf(L"\n[TCP Server] Disconnect Client: IP=%s, port=%d\n",
				szClientIP, ntohs(clientaddr.sin_port));
			continue;
		}

		SOCKETINFO* ptr = SocketInfoArray[sockCnt - 1];
		flags = 0;

		ret = WSARecv(ptr->sock, &ptr->wsabuf, 1, &recvbytes,
			&flags, &ptr->overlapped, NULL);

		if (ret == SOCKET_ERROR)
		{
			if (WSAGetLastError() != WSA_IO_PENDING)
			{
				_HandleError;
				RemoveSocketInfo(sockCnt - 1);
				continue;
			}	
		}
		WSASetEvent(EventArray[0]);
	}

	WSACleanup();
	DeleteCriticalSection(&cs);
	return 0;
}
```

## **4) Callback 방식**

Callback 방식을 사용할 경우 APC 큐를 사용하게 된다. 비동기 IO 함수를 호출하면 OS, 윈도우 IO 시스템에 IO 작업을 요청하게 된다. 이때 작업을 요청한 스레드는 WaitFor-ObjectEx(), WSAWaitForMultipleEx(), SleepEx()등의 함수를 통해 alertable wait 상태에 진입한다.

alertable wait는 비동기 IO를 위한 대기 상태이다. 일반적인 wait는 wait가 끝나기 전까지 CPU을 사용하지 못하지만 alerable의 경우 끝나기 직전에 CPU 시간을 할당 받아 완료 루틴을 호출하고, 더 처리할 게 없으면 대기 상태가 끝난다. 즉, IO 함수를 호출한 스레드가 이 상태에 있어야 완료 루틴이 호출될 수 있다.

비동기 입출력 작업이 완료되면 OS는 스레드별로 있는 APC 큐에 결과를 저장한다. 이때 스레드가 alertable wait 상태면 OS는 APC 큐에 저장된 모든 완료 루틴들을 호출하며, 스레드는 이 작업이 끝나면 alertable wait 상태에서 빠져나온다. 스레드가 IO 결과를 계속 처리하려면 다시 alertable wait 상태에 진입해야 한다. 

```c++
void CALLBACK CompletionRoutine
(
    DWORD dwError,
    DWORD cbTransferred,
    LPWSAOVERLAPPED lpOverlapped,
    DWORD dwFlags
);
```
완료 루틴은 위 형태를 지닌다. dwError는 비동기 IO 결과로, 오류가 나면 0이 아닌 값이 된다. cbTransferred는 전송 바이트 수로, 상대가 접속을 종료하면 0이 된다. lpOverlapped의 경우 비동기 IO 함수 호출 시 넘긴 구조체가 이를 통해 다시 응용 프로그램으로 넘어오는데 여기서는 쓰이지 않는다. dwFlags도 현재는 쓰이지 않는다. 

이를 바탕으로 구현한 예제는 아래와 같다. 

```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <stdio.h>

#define SERVERPORT 9000
#define BUFSIZE 512

void HandleError(int line)
{
	int err = ::WSAGetLastError();
	wprintf(L"Error! Line %d: %d", line, err);
}
#define _HandleError HandleError(__LINE__)

// Socket Info
struct SOCKETINFO
{
	WSAOVERLAPPED overlapped;
	SOCKET sock;
	char buf[BUFSIZE + 1];
	int recvbytes;
	int sendbytes;
	WSABUF wsabuf;
};

SOCKET client_sock;
HANDLE hReadEvent, hWriteEvent;

void CALLBACK CompletionRoutine(DWORD dwError, DWORD cbTransferred,
	LPWSAOVERLAPPED lpOverlapped, DWORD dwFlags)
{
	int ret;

	SOCKETINFO* ptr = (SOCKETINFO*)lpOverlapped;

	SOCKADDR_IN clientaddr;
	WCHAR szClientIP[16] = { 0 };
	int addrlen = sizeof(clientaddr);
	getpeername(ptr->sock, (SOCKADDR*)&clientaddr, &addrlen);
	InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);

	if (dwError != 0 || cbTransferred == 0)
	{
		if (dwError != 0) _HandleError;
		closesocket(ptr->sock);
		wprintf(L"\n[TCP Server] Disconnect Client: IP=%s, port=%d\n",
			szClientIP, ntohs(clientaddr.sin_port));
		delete ptr;
		return;
	}

	if (ptr->recvbytes == 0)
	{
		ptr->recvbytes = cbTransferred;
		ptr->sendbytes = 0;
		ptr->buf[ptr->recvbytes] = '\0';
		wprintf(L"\n[TCP Server] IP=%s, port=%d: ",
			szClientIP, ntohs(clientaddr.sin_port));
		printf("%s\n", ptr->buf);
	}
	else
	{
		ptr->sendbytes += cbTransferred;
	}

	if (ptr->recvbytes > ptr->sendbytes)
	{
		ZeroMemory(&ptr->overlapped, sizeof(ptr->overlapped));
		ptr->wsabuf.buf = ptr->buf + ptr->sendbytes;
		ptr->wsabuf.len = ptr->recvbytes - ptr->sendbytes;

		DWORD sendbytes;

		// Send Data to Client
		ret = WSASend(ptr->sock, &ptr->wsabuf, 1,
			&sendbytes, 0, &ptr->overlapped, CompletionRoutine);

		if (ret == SOCKET_ERROR)
		{
			if (WSAGetLastError() != WSA_IO_PENDING)
			{
				_HandleError;
				return;
			}	
		}
	}
	else
	{
		ptr->recvbytes = 0;

		ZeroMemory(&ptr->overlapped, sizeof(ptr->overlapped));
		ptr->wsabuf.buf = ptr->buf;
		ptr->wsabuf.len = BUFSIZE;

		DWORD recvbytes;
		DWORD flags = 0;
		ret = WSARecv(ptr->sock, &ptr->wsabuf, 1,
			&recvbytes, &flags, &ptr->overlapped, CompletionRoutine);

		if (ret == SOCKET_ERROR)
		{
			if (WSAGetLastError() != WSA_IO_PENDING)
			{
				_HandleError;
				return;
			}
		}
	}
}

DWORD WINAPI WorkerThread(LPVOID arg)
{
	int ret;

	while (1)
	{
		while (1)
		{
			// alertable wait
			DWORD result = WaitForSingleObjectEx(hWriteEvent, INFINITE, TRUE);
			if (result == WAIT_OBJECT_0) break;
			if (result != WAIT_IO_COMPLETION) return 1;
		}

		SOCKADDR_IN clientaddr;
		WCHAR szClientIP[16] = { 0 };
		int addrlen = sizeof(clientaddr);
		getpeername(client_sock, (SOCKADDR*)&clientaddr, &addrlen);
        InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
		wprintf(L"\n[TCP Server] Connect Client: IP=%s, port=%d\n",
			szClientIP, ntohs(clientaddr.sin_port));

		SOCKETINFO* ptr = new SOCKETINFO;
		if (ptr == NULL)
		{
			wprintf(L"[Error] 메모리가 부족합니다.\n");
			return 1;
		}
	
		ZeroMemory(&ptr->overlapped, sizeof(ptr->overlapped));
		ptr->sock = client_sock;
		SetEvent(hReadEvent);
		ptr->recvbytes = ptr->sendbytes = 0;
		ptr->wsabuf.buf = ptr->buf;
		ptr->wsabuf.len = BUFSIZE;

		DWORD recvbytes;
		DWORD flags = 0;
		ret = WSARecv(ptr->sock, &ptr->wsabuf, 1,
			&recvbytes, &flags, &ptr->overlapped, CompletionRoutine);

		if (ret == SOCKET_ERROR)
		{
			if (WSAGetLastError() != WSA_IO_PENDING)
			{
				_HandleError;
				return 1;
			}
		}	
	}
}

int main(int argc, char* argv[])
{
	int ret;
	int err;

	// Initialize
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create Socket
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET)
	{
		_HandleError;
		return 0;
	}

	// Bind Socket
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, L"0.0.0.0", &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);
	ret = bind(listen_sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Socket to Listen
	ret = listen(listen_sock, SOMAXCONN);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Network Event
	hReadEvent = CreateEvent(NULL, FALSE, TRUE, NULL);
	if (hReadEvent == NULL) return 1;
	hWriteEvent = CreateEvent(NULL, FALSE, FALSE, NULL);
	if (hWriteEvent == NULL) return 1;

	HANDLE hThread = CreateThread(NULL, 0, WorkerThread, NULL, 0, NULL);
	if (hThread == NULL) return 1;
	CloseHandle(hThread);

	while (1)
	{
		WaitForSingleObject(hReadEvent, INFINITE);
		client_sock = accept(listen_sock, NULL, NULL);
		if (client_sock == INVALID_SOCKET)
		{
			_HandleError;
			break;
		}
		SetEvent(hWriteEvent);
	}

	WSACleanup();
	return 0;
}
```

<br/>

# **3. IOCP 모델**

## **1) 작동 원리**

OS는 IO Completion Port, 줄여서 IOCP라는 비동기 IO 처리용 구조를 제공한다. APC 큐는 스레드마다 자동 생성되고 해당 스레드만 접근할 수 있는 반면, IOCP는 생성 함수로 생성하고 어떤 스레드든 접근할 수 있다. 대개 IOCP용 스레드인 작업자 스레드를 별도로 두며, 그 개수는 CPU 개수의 배수를 권장한다. APC 내용을 처리할 때 해당 스레드를 alertable로 만드는 함수를 호출했듯이, IOCP에 저장된 결과를 처리할 땐 작업자 스레드가 GetQueuedCompletionStatus를 호출한다.

스레드가 비동기 IO 함수를 호출하여 OS에 IO 작업을 요청하면 모든 작업자 스레드는  GetQueuedCompletionStatus를 호출해 IOCP를 감시한다. 완료된 IO 작업이 없을 땐 모든 작업자 스레드가 대기 상태가 되고 이 스레드 목록이 IOCP 내부에 저장된다. IO 작업이 완료되면 OS는 IOCP에 결과를 저장하는데 이때 저장되는 정보를 IO 완료 패킷이라 한다. OS는 IOCP 에 저장된 작업자 스레드 목록에서 하나를 선택하여 깨우고, 깨어난 스레든느 IO 결과를 처리한다. 

## **2) 구성 요소**

```c++
HANDLE CreaetIoCompletionPort
(
    HANDLE FileHandle,
    HANDLE ExistingCompletionPort,
    ULONG CompletionKey,
    DWORD NumberOfConcurrentThreads
);
```
IOCP를 생성하거나 연결하는 함수이다. 순서대로 연결할 파일 핸들 혹은 소켓 디스크립터, IOCP 핸들, IO 완료 패킷에 들어갈 부가 정보, 동시 실행 작업자 스레드 개수를 전달한다. IOCP 핸들에 핸들 값을 넣을 경우 파일 혹은 소켓과 IOCP를 연결하는 작업을 수행하고, NULL을 넣을 경우 새로은 IOCP를 생성한다. 

```c++
BOOL GetQueuedCompletionStatus
(
    HANDLE CompletionPort,
    LPDWORD lpNumberOfBytes,
    LPDWORD lpCompletionKey,
    LPOVERLAPPED* lpOverlapped,
    DOWRD dwMilliseconds
);
```

IOCP에 IO 완료 패킷이 들어올 때까지 대기하는 함수이다. 패킷이 IOCP에 들어오면 OS는 실행 중인 작업자 스레드 개수를 체크해서 이 값이 CreateIOCP 네번째 인자 값보다 작다면 대기 상태인 작업자 스레드를 깨워 IO 완료 패킷을 처리하게 한다. 순서대로 IOCP 핸들, 바이트 수를 저장할 변수, 패킷의 부가정보를 저장할 변수, 비동기 IO 함수 호출 시 전달했던 구조체 주소 값, 스레드가 대기할 시간을 전달한다. 

이 외에도 PostQueuedCompletionStatus를 통해 IO 완료 패킷을 생성하여 응용 프로그램이 작업자 스레드에게 직접 정보를 전달할 수도 있다. 

## **3) 예제**

```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <stdio.h>

#define SERVERPORT 9000
#define BUFSIZE 512

void HandleError(int line)
{
	int err = ::WSAGetLastError();
	wprintf(L"Error! Line %d: %d", line, err);
}
#define _HandleError HandleError(__LINE__)

// Socket Info
struct SOCKETINFO
{
	WSAOVERLAPPED overlapped;
	SOCKET sock;
	char buf[BUFSIZE + 1];
	int recvbytes;
	int sendbytes;
	WSABUF wsabuf;
};

DWORD WINAPI WorkerThread(LPVOID arg)
{
	int ret;
	HANDLE hcp = (HANDLE)arg;

	while (1)
	{
		DWORD cbTransferred;
		SOCKET client_sock;
		SOCKETINFO* ptr;

		ret = GetQueuedCompletionStatus(hcp, &cbTransferred, 
			(PULONG_PTR)&client_sock, (LPOVERLAPPED*)&ptr, INFINITE);

		SOCKADDR_IN clientaddr;
		int addrlen = sizeof(clientaddr);
		getpeername(ptr->sock, (SOCKADDR*)&clientaddr, &addrlen);
		WCHAR szClientIP[16] = { 0 }; 
		InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);

		if (ret == 0 || cbTransferred == 0)
		{
			if (ret == 0)
			{
				DWORD tmp1, tmp2;
				WSAGetOverlappedResult(ptr->sock, &ptr->overlapped,
					&tmp1, FALSE, &tmp2);
				_HandleError;
			}
			closesocket(ptr->sock);
			wprintf(L"\n[TCP Server] Disconnect Client: IP=%s, port=%d\n",
				szClientIP, ntohs(clientaddr.sin_port));
			delete ptr;
			continue;
		}

		if (ptr->recvbytes == 0)
		{
			ptr->recvbytes = cbTransferred;
			ptr->sendbytes = 0;
			ptr->buf[ptr->recvbytes] = '\0';
			wprintf(L"\n[TCP Server] IP=%s, port=%d: ",
				szClientIP, ntohs(clientaddr.sin_port));
			printf("%s\n", ptr->buf);
		}
		else
		{
			ptr->sendbytes += cbTransferred;
		}

		if (ptr->recvbytes > ptr->sendbytes)
		{
			ZeroMemory(&ptr->overlapped, sizeof(ptr->overlapped));
			ptr->wsabuf.buf = ptr->buf + ptr->sendbytes;
			ptr->wsabuf.len = ptr->recvbytes - ptr->sendbytes;

			DWORD sendbytes;

			// Send Data to Client
			ret = WSASend(ptr->sock, &ptr->wsabuf, 1,
				&sendbytes, 0, &ptr->overlapped, NULL);

			if (ret == SOCKET_ERROR)
			{
				if (WSAGetLastError() != WSA_IO_PENDING)
					_HandleError;
				continue;
			}
		}
		else
		{
			ptr->recvbytes = 0;

			ZeroMemory(&ptr->overlapped, sizeof(ptr->overlapped));
			ptr->wsabuf.buf = ptr->buf;
			ptr->wsabuf.len = BUFSIZE;

			DWORD recvbytes;
			DWORD flags = 0;
			ret = WSARecv(ptr->sock, &ptr->wsabuf, 1,
				&recvbytes, &flags, &ptr->overlapped, NULL);

			if (ret == SOCKET_ERROR)
			{
				if (WSAGetLastError() != WSA_IO_PENDING)
					_HandleError;
				continue;
			}
		}
	}
	return 0;
}

int main(int argc, char* argv[])
{
	int ret;

	// Initialize
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create IOCP
	HANDLE hcp = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);
	if (hcp == NULL) return 1;

	SYSTEM_INFO si;
	GetSystemInfo(&si);

	HANDLE hThread;
	for (int i = 0; i < (int)si.dwNumberOfProcessors * 2; i++)
	{
		hThread = CreateThread(NULL, 0, WorkerThread, hcp, 0, NULL);
		if (hThread == NULL) return 1;
		CloseHandle(hThread);
	}
	
	// Create Socket
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET)
	{
		_HandleError;
		return 0;
	}

	// Bind Socket
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, L"0.0.0.0", &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);
	ret = bind(listen_sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Socket to Listen
	ret = listen(listen_sock, SOMAXCONN);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	SOCKET client_sock;
	SOCKADDR_IN clientaddr;
	int addrlen = sizeof(clientaddr);
	DWORD recvbytes, flags;

	while (1)
	{
		client_sock = accept(listen_sock, (SOCKADDR*)&clientaddr, &addrlen);
		if (client_sock == INVALID_SOCKET)
		{
			_HandleError;
			break;
		}

		WCHAR szClientIP[16] = { 0 };
		InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
		wprintf(L"\n[TCP Server] Connect Client: IP=%s, port=%d\n",
			szClientIP, ntohs(clientaddr.sin_port));

		// Connect Socket to IOCP
		CreateIoCompletionPort((HANDLE)client_sock, hcp, client_sock, 0);

		// Alloc Socket Info Struct
		SOCKETINFO* ptr = new SOCKETINFO;
		if (ptr == NULL) break;

		ZeroMemory(&ptr->overlapped, sizeof(ptr->overlapped));
		ptr->sock = client_sock;
		ptr->recvbytes = ptr->sendbytes = 0;
		ptr->wsabuf.buf = ptr->buf;
		ptr->wsabuf.len = BUFSIZE;

		DWORD recvbytes;
		DWORD flags = 0;
		ret = WSARecv(ptr->sock, &ptr->wsabuf, 1,
			&recvbytes, &flags, &ptr->overlapped, NULL);

		if (ret == SOCKET_ERROR)
		{
			if (WSAGetLastError() != WSA_IO_PENDING)
			{
				_HandleError;
			}
			continue;
		}
	}

	WSACleanup();
	return 0;
}
```

<br/>

# **출처**

TCP/IP 윈도우 소켓 프로그래밍, 저자 김선우, 한빛미디어