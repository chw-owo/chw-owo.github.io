---
title: Network Basic 06) IOCP (Completion Port) Model
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4: 게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. IOCP (Completion Port) 모델**

기존의 Overlapped 모델 방식은 스레드마다 있는 APC 큐에 루틴이 쌓이기 때문에 멀티스레드에서의 분배 시 아쉬움이 있었다. IOCP 모델을 사용할 경우 APC 큐 대신 Completion Port라는 중앙화된 큐를 사용하기 때문에 이런 한계를 보완할 수 있다. 스레드를 Alertable Wait 상태로 전환했던 것도 GetQueuedCompletionStatus 함수를 통해 Completion port 결과를 처리하는 것으로 대체된다.

<br/> 

# **2. IOCP 모델 예제**

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
