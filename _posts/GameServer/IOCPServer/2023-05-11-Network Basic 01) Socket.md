---
title: Network Basic 01) Socket
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Server**

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
	
		char ipAddress[16];
		::inet_ntop(AF_INET, &clientAddr.sin_addr, ipAddress, sizeof(ipAddress));
		printf("Connected to Client %s!\n", ipAddress);

		while (1)
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

<br/> 

# **2. Client**

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
		// To-do
	}

	// Clean up Winsock
	::closesocket(client_socket);
	::WSACleanup();
	return 0;
}
```

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
