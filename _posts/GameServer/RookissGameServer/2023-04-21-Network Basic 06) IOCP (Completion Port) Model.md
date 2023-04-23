---
title: Network Basic 06) IOCP (Completion Port) Model
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4: 게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Completion Port**

```c++
HANDLE CreateIoCompletionPort
(
	HANDLE hFile,
	HANDLE hExistingCompletionPort,
	ULONG_PTR ComletionKey,
	DWORD dwNumberOfConcurrentThreads
);
```

Completion Port는 멀티스레드 환경에서 스레드 간으니 컨텍스트 스위칭을 줄이고 효율적으로 부하 분산을 할 수 있도록 만들기 위해 고안된 커널 오브젝트이다. CreateIoCompletionPort로 IOCP 생성, 장치와 IOCP의 연계 두 작업을 수행할 수 있다. IOCP는 커널 오브젝트지만 단일 프로세스 내에서만 수행되기 때문에 생성 시 Secutiry Attributes 구조체를 전달하지 않는다. 

![image](https://user-images.githubusercontent.com/96677719/223046757-add673d7-daec-4a14-9b52-3ba98a2955a6.png)

IOCP를 생성하면 커널은 장치 리스트, IO 컴플리션 큐, 대기 스레드 큐, Released 스레드 리스트, Paused 스레드 리스트 총 5개의 데이터 구조를 생성한다. 장치 리스트는 CreateIoCompletionPort 호출 시 항목이 추가되며 핸들이 Close 될 때 제거된다. IO 컴플리션 큐는 FIFO로 동작하며 IO 요청 완료 시, PostQueuedCompletionStatus 호출 시 항목이 추가되고 IOCP가 대기 스레드 큐에 있는 항목을 가져올 때 제거된다. 

Released 스레드 리스트에는 대기 스레드 큐, Paused 스레드 리스트의 항목들이 저장된다. IOCP가 대기 스레드 큐에 있는 스레드를 깨우는 경우, Paused 되었던 스레드가 깨어난 경우에 항목이 추가된다. 이때 스레드가 Paused 된 함수를 호출하면 Paused 스레드 리스트로, GetQueuedCompletionStatus를 호출하면 대기 스레드 큐로 항목이 이동한다. 

마찬가지로 대기 스레드 큐는 GetQueuedCompletionStatus 호출 시 항목이 추가되며, IO 컴플리션 큐가 비어있지 않고 수행 중인 스레드 개수가 동시 수행 가능한 수일 경우 제거된다. 이는 LIFO로 동작한다. Paused 스레드 리스트 역시 스레드가 스레드 정지 함수를 호출했을 때 항목이 추가되고, Paused 되었던 스레드가 깨어났을 때 항목이 제거된다. 

완료 통지가 삽입되면 IOCP는 대기 중인 스레드를 깨운다. 동시 수행 스레드 수가 2라면 스레드 4개가 대기 중이어도 2개의 스레드만 수행을 재개하고 나머지 2개는 계속 대기한다. 수행을 재개한 스레드는 완료 통지를 처리한 후 다시 GetQueued-를 호출할 것이고, 시스템은 컴플리션 큐의 항목을 처리하기 위해 이 스레드를 다시 깨울 것이다. 

# **2. IOCP 모델**

네트워크 IO 처리에 IOCP를 사용하는 모델이다. Overlapped 모델은 스레드 별 APC 큐에 루틴이 쌓여서 부하 분산에 한계가 있었다. IOCP 모델은 APC 대신 중앙화된 큐 역할을 하는 CP를 사용하므로 이를 보완할 수 있다. 스레드를 Alertable Wait 상태로 전환했던 것도 GetQueuedCompletionStatus 함수로 CP 결과를 처리하는 것으로 대체된다.

<br/> 

# **3. IOCP 모델 예제**

```c++
#include "pch.h"
#include "CorePch.h"
#include "ThreadManager.h"

#include <iostream>
#include <vector>
#include <windows.h>
#include <winsock2.h>
#include <mswsock.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")

void HandleError(const char* cause)
{
	int32 errCode = ::WSAGetLastError();
	cout << cause << " ErrorCode : " << errCode << endl;
}

const int32 BUFSIZE = 1000;
struct Session
{
	SOCKET socket = INVALID_SOCKET;
	char recvBuffer[BUFSIZE] = {};
	int32 recvBytes = 0;	
};

enum IO_TYPE
{
	DEFAULT,
	READ,
	WRITE,
	ACCEPT,
	CONNECT,
};

struct OverlappedEx
{
	WSAOVERLAPPED overlapped = {};
	IO_TYPE type = DEFAULT;
};

void WorkerThreadMain(HANDLE iocpHandle)
{
	while (true)
	{
		DWORD bytesTransferred = 0;
		Session* session = nullptr;
		OverlappedEx* overlappedEx = nullptr;

		BOOL ret = ::GetQueuedCompletionStatus(iocpHandle, &bytesTransferred,
			(ULONG_PTR*)&session, (LPOVERLAPPED*)&overlappedEx, INFINITE);

		if (ret == FALSE || bytesTransferred == 0)
		{
			// TODO : 연결 끊김
			continue;
		}

		ASSERT_CRASH(overlappedEx->type == IO_TYPE::READ);

		cout << "Recv Data IOCP = " << bytesTransferred << endl;

		WSABUF wsaBuf;
		wsaBuf.buf = session->recvBuffer;
		wsaBuf.len = BUFSIZE;

		DWORD recvLen = 0;
		DWORD flags = 0;
		::WSARecv(session->socket, &wsaBuf, 1, &recvLen, &flags, &overlappedEx->overlapped, NULL);
	}
}

int main()
{
	WSAData wsaData;
	if (::WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
		return 0;

	SOCKET listenSocket = ::socket(AF_INET, SOCK_STREAM, 0);
	if (listenSocket == INVALID_SOCKET)
		return 0;

	SOCKADDR_IN serverAddr;
	::memset(&serverAddr, 0, sizeof(serverAddr));
	serverAddr.sin_family = AF_INET;
	serverAddr.sin_addr.s_addr = ::htonl(INADDR_ANY);
	serverAddr.sin_port = ::htons(7777);

	if (::bind(listenSocket, (SOCKADDR*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR)
		return 0;

	if (::listen(listenSocket, SOMAXCONN) == SOCKET_ERROR)
		return 0;

	cout << "Accept" << endl;
	vector<Session*> sessionManager;

	// Create Completion Port
	HANDLE iocpHandle = ::CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);

	// Worker Thread: 관찰 담당
	for (int32 i = 0; i < 5; i++)
		GThreadManager->Launch([=]() { WorkerThreadMain(iocpHandle); });

	// Main Thread: Accept 담당
	while (true)
	{
		SOCKADDR_IN clientAddr;
		int32 addrLen = sizeof(clientAddr);

		SOCKET clientSocket = ::accept(listenSocket, (SOCKADDR*)&clientAddr, &addrLen);
		if (clientSocket == INVALID_SOCKET)
			return 0;

		Session* session = new Session();
		session->socket = clientSocket;
		sessionManager.push_back(session);

		cout << "Client Connected !" << endl;

		// 소켓을 Completion Port에 등록
		::CreateIoCompletionPort((HANDLE)clientSocket, iocpHandle, /*Key*/(ULONG_PTR)session, 0);

		WSABUF wsaBuf;
		wsaBuf.buf = session->recvBuffer;
		wsaBuf.len = BUFSIZE;

		OverlappedEx* overlappedEx = new OverlappedEx();
		overlappedEx->type = READ;

		DWORD recvLen = 0;
		DWORD flags = 0;
		::WSARecv(clientSocket, &wsaBuf, 1, &recvLen, &flags, &overlappedEx->overlapped, NULL);
	}

	GThreadManager->Join();
	::WSACleanup();
}
```

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
