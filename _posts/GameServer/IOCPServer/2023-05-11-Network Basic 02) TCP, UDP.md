---
title: Network Basic 02) TCP, UDP
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. TCP**

**Client.cpp**
```c++
#include "pch.h"

#include <stdio.h>
#include <winsock2.h>
#include <mswsock.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")

int main()
{
	// Initialize Winsock
	WSADATA wsaData;
	if (::WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
	{
		printf("Fail to Initialize Winsock.\n");
		return 0;
	}
	int err;

	// Create Socket
	SOCKET client_socket = ::socket(AF_INET, SOCK_STREAM, 0);
	if (client_socket == INVALID_SOCKET)
	{
		err = ::WSAGetLastError();
		printf("Fail to Create Socket. Socket Error: %d\n", err);
		return 0;
	}

	// Set Server Address to Socket
	SOCKADDR_IN serverAddr;
	::memset(&serverAddr, 0, sizeof(serverAddr));
	serverAddr.sin_family = AF_INET;
	serverAddr.sin_addr.s_addr =
		::inet_pton(AF_INET, "127.0.0.1", &serverAddr.sin_addr);
	serverAddr.sin_port = ::htons(7777);

	// Connect Socket to Server
	if (::connect(client_socket, (SOCKADDR*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR)
	{
		err = ::WSAGetLastError();
		printf("Fail to Connect Socket. Socket Error: %d\n", err);
		return 0;
	}

	printf("Connected to Server!\n");
	while (1)
	{
		int ret;
		char sendBuffer[100] = "Hello World!";

		// 버퍼 크기 전체를 전송할 경우
		ret = ::send(client_socket, sendBuffer, sizeof(sendBuffer), 0); 
		if (ret == SOCKET_ERROR)
		{
			err = ::WSAGetLastError();
			printf("Fail to Send. Socket Error: %d\n", err);
			return 0;
		}
		printf("Success to Send %u Bytes!", sizeof(sendBuffer));
	
		char recvBuffer[1000];
		ret = ::recv(client_socket, recvBuffer, sizeof(recvBuffer), 0);
		if (ret <= 0)
		{
			err = ::WSAGetLastError();
			printf("Fail to Receive. Socket Error: %d\n", err);
			return 0;
		}
		printf("Success to Receive %u Bytes!: %s", ret, recvBuffer);
	
		Sleep(1000);
	}

	// Clean up Winsock
	::closesocket(client_socket);
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
int main()
{
	// Initialize Winsock
	WSADATA wsaData;
	if (::WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
	{
		printf("Fail to Initialize Winsock.\n");
		return 0;
	}

	int err;

	// Create Socket
	SOCKET listen_socket = ::socket(AF_INET, SOCK_STREAM, 0);
	if (listen_socket == INVALID_SOCKET)
	{
		err = ::WSAGetLastError();
		printf("Fail to Create Socket. Socket Error: %d\n", err);
		return 0;
	}

	// Set Any Client Address to Socket
	SOCKADDR_IN serverAddr;
	::memset(&serverAddr, 0, sizeof(serverAddr));
	serverAddr.sin_family = AF_INET;
	serverAddr.sin_addr.s_addr = ::htonl(INADDR_ANY);
	serverAddr.sin_port = ::htons(7777);

	// Bind Socket
	if (::bind(listen_socket, (SOCKADDR*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR)
	{
		err = ::WSAGetLastError();
		printf("Fail to Bind Socket. Socket Error: %d\n", err);
		return 0;
	}

	// Set Socket to Listen
	if(::listen(listen_socket, SOMAXCONN) == SOCKET_ERROR)
	{
		err = ::WSAGetLastError();
		printf("Fail to Set Socket to Listen. Socket Error: %d\n", err);
		return 0;
	}

	while (1)
	{
		SOCKADDR_IN clientAddr;
		::memset(&clientAddr, 0, sizeof(clientAddr));
		int addrLen = sizeof(clientAddr);
		SOCKET client_socket = ::accept(listen_socket, (SOCKADDR*)&clientAddr, &addrLen);
		if(client_socket == INVALID_SOCKET)
		{
			err = ::WSAGetLastError();
			printf("Fail to Accept Socket. Socket Error: %d\n", err);
			return 0;
		}

        while(1)
        {
            // To-do
        }

		::closesocket(client_socket);
	}

	// Clean up Winsock
	::closesocket(listen_socket);
	::WSACleanup();
	return 0;
}
```

send, recv는 block 소켓일 때 block 함수로 작동한다. 그러나 위와 같이 Server에서 accept만 하고 recv를 하지 않아도 send는 무사히 반환이 되는 것을 볼 수 있다. 이는 send 함수가 직접 전송을 하는 함수가 아니라 소켓의 send buffer에 메시지를 넣는 역할만 하는 함수이기 때문이다. 실제로 전송을 하는 것은 TCP 계층에서 담당하며, send buffer에 메시지를 넣은 이후에는 다음 작업으로 넘어갈 수 있다.

recv 함수 역시 직접 수신을 하는 게 아니라 recv buffer에 들어온 메시지를 꺼내오는 작업만을 한다. 하지만 이 경우 send와는 달리 recv buffer에 메시지가 없다면 작업을 수행할 수 없으므로 메시지가 들어올 때까지 block이 걸린다. 그리고 메시지가 들어와서 block이 풀리면, 그때 recv buffer에 들어온 메시지를 사용자가 세팅한 buffer에 복사한다는 게 일반적인 설명이다.

그러나 실제로는 block 걸리지 않은 상태에서만 recv buffer를 통한 복사가 이루어진다. block이 걸린 상태에서 메시지가 들어올 경우, recv buffer를 통하는 대신 TCP가 사용자가 세팅한 buffer에 직접 데이터를 입력한다. 

<br/> 

# **2. UDP**

```c++
```

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
