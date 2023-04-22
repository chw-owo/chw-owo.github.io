---
title: Network Basic 03) Non-Blocking Socket
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Blocking**

|function|return 시점|
|--------|------------|
|accept()|클라이언트가 접속했을 때|
|connect()| 서버 접속에 성공했을 때|
|send(), sendto()| 요청한 데이터를 송신 버퍼에 복사했을 때|
|recv(), recvfrom()| 수신 버퍼에 도착한 데이터가 있고, 이를 유저 버퍼에 복사했을 때|

Blocking 소켓의 경우 함수가 반환되기 전까지 return 시점까지 Block이 걸리기 때문에, Stateful 서버에서는 일반적으로 Non-Blocking 소켓을 사용한다. 

# **2. Non-Blocking**

**Client.cpp**
```c++
#include "pch.h"

#include <stdio.h>
#include <winsock2.h>
#include <mswsock.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")

void HandleError(const char* cause)
{
	int err = ::WSAGetLastError();
	printf("Fail to %s: %d\n", cause, err);
}

int main()
{
	// Initialize Winsock
	WSADATA wsaData;
	if (::WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
	{
		printf("Fail to Initialize Winsock.\n");
		return 0;
	}

	// Create Socket
	SOCKET client_socket = ::socket(AF_INET, SOCK_STREAM, 0);
	if (client_socket == INVALID_SOCKET)
	{
		HandleError("Create Socket");
		return 0;
	}

	// Set Non-Blocking Socket
	int ret;
	u_long on = 1;
	ret = ::ioctlsocket(client_socket, FIONBIO, &on);
	if (ret == INVALID_SOCKET)
	{
		HandleError("Set Non-Blocking Socket");
		return 0;
	}

	// Set Server Address to Socket
	SOCKADDR_IN serverAddr;
	::memset(&serverAddr, 0, sizeof(serverAddr));
	serverAddr.sin_family = AF_INET;
	serverAddr.sin_addr.s_addr =
		::inet_pton(AF_INET, "127.0.0.1", &serverAddr.sin_addr);
	serverAddr.sin_port = ::htons(7777);

	// Connect
	while (1)
	{
		ret = ::connect(client_socket,
			(SOCKADDR*)&serverAddr, sizeof(serverAddr));
		if (ret == SOCKET_ERROR)
		{
			if (::WSAGetLastError() == WSAEWOULDBLOCK)
				continue;
			if (::WSAGetLastError() == WSAEISCONN)
				break;
			HandleError("Connect");
			break;
		}
	}
	
	printf("Success to Connect!\n");
	char sendBuffer[100] = "Hello World!";

	while (1)
	{
		ret = ::send(client_socket, sendBuffer, sizeof(sendBuffer), 0);
		if (ret == SOCKET_ERROR)
		{
			if (::WSAGetLastError() == WSAEWOULDBLOCK)
				continue;
			HandleError("Send");
			break;
		}
		printf("Success to Send %d bytes!\n", sizeof(sendBuffer));
	
		while (1)
		{
			char recvBuffer[1000];
			int ret = ::recv(client_socket, recvBuffer, sizeof(recvBuffer), 0);

			if (ret == SOCKET_ERROR)
			{
				if (::WSAGetLastError() == WSAEWOULDBLOCK)
					continue;
				HandleError("Recv");
				break;
			}
			else if (ret == 0)
			{
				break;
			}
			printf("Success to Receive %u Bytes!: %s\n", recvLen, recvBuffer);
			break;
		}
		Sleep(1000);
	}

	// Clean up Winsock
	closesocket(client_socket);
	::WSACleanup();
	return 0;
}
```

**Server.cpp**
```c++
#include "pch.h"
#include "CorePch.h"

#include <iostream>
#include <winsock2.h>
#include <mswsock.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")

void HandleError(const char* cause)
{
	int err = ::WSAGetLastError();
	printf("Fail to %s: %d\n", cause, err);
}

int main()
{
	// Initialize Winsock
	WSADATA wsaData;
	if (::WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
	{
		printf("Fail to Initialize Winsock.\n");
		return 0;
	}

	// Create Socket
	SOCKET listen_socket = ::socket(AF_INET, SOCK_STREAM, 0);
	if (listen_socket == INVALID_SOCKET)
	{
		HandleError("Create Socket");
		return 0;
	}

	// Set Non-Blocking Socket
	int ret;
	u_long on = 1;
	ret = ::ioctlsocket(listen_socket, FIONBIO, &on);
	if (ret == INVALID_SOCKET)
	{
		HandleError("Set Non-Blocking Socket");
		return 0;
	}

	SOCKADDR_IN serverAddr;
	::memset(&serverAddr, 0, sizeof(serverAddr));
	serverAddr.sin_family = AF_INET;
	serverAddr.sin_addr.s_addr = ::htonl(INADDR_ANY);
	serverAddr.sin_port = ::htons(7777);

	// Bind Socket
	ret = ::bind(listen_socket, (SOCKADDR*)&serverAddr, sizeof(serverAddr));
	if (ret == SOCKET_ERROR)
	{
		HandleError("Bind");
		return 0;
	}

	// Set Socket Listen
	ret = ::listen(listen_socket, SOMAXCONN);
	if (ret == SOCKET_ERROR)
	{
		HandleError("Listen");
		return 0;
	}

	// Accept Client Socket
	SOCKADDR_IN clientAddr;
	::memset(&clientAddr, 0, sizeof(clientAddr));
	int addrLen = sizeof(clientAddr);

	while (1)
	{
		SOCKET client_socket = ::accept(
			listen_socket, (SOCKADDR*)&clientAddr, &addrLen);
		if (client_socket == INVALID_SOCKET)
		{
			if (::WSAGetLastError() == WSAEWOULDBLOCK)
				continue;
			HandleError("Accept");
			break;
		}

		printf("Client Connected\n");

		char recvBuffer[1000] = { '\0', };
		int recvLen = 0;

		while (1)
		{
			recvLen = ::recv(client_socket, recvBuffer, sizeof(recvBuffer), 0);

			if (recvLen == SOCKET_ERROR)
			{
				if (::WSAGetLastError() == WSAEWOULDBLOCK)
					continue;
				HandleError("Receive");
				break;
			}
			else if (recvLen == 0)
			{
				closesocket(client_socket);
				break;
			}

			printf("Success to Receive %d bytes: %s\n", recvLen, recvBuffer);

			while (1)
			{
				ret = ::send(client_socket, recvBuffer, recvLen, 0);

				if (ret == SOCKET_ERROR)
				{
					if (::WSAGetLastError() == WSAEWOULDBLOCK)
						continue;
					HandleError("Send");
					break;
				}
				printf("Success to Send %d bytes.\n", recvLen);
			}
		}
	}
	
	// Clean up Winsock
	closesocket(listen_socket);
	::WSACleanup();
	return 0;
}
```

Non-Blocking 소켓을 만드는 기본 문법은 위와 같다. 그러나 테스트해보면 오히려 Blocking 소켓보다 성능이 좋지 않은 것을 확인할 수 있다. 위와 같이 적은 양의 데이터를 보내고 받기만 하는 예제의 경우, 완료될 때까지 대기하는 것보다 무한 루프를 계속 도는 것이 더 큰 성능 저하를 유발한다. 이 때문에 불필요한 CPU 낭비를 막을 수 있는 소켓 모델들이 존재하며 이를 적절히 활용해야 한다. 

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
