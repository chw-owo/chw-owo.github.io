---
title: Network Basic 01) Socket
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Winsock 사용법**

Winsock은 Window에서 제공하는 Socket 라이브러리로, 아래는 Winsock에서의 가장 기본적인 TCP 예제이다. 클라이언트의 경우 윈속 초기화 - 소켓 생성 - 서버 주소 세팅 - Connect - 통신 - 소켓 닫기 - 윈속 제거의 과정을, 서버의 경우 윈속 초기화 - 소켓 생성 - 서버 주소 바인딩 - Listen - Accept 통신 - 소켓 닫기 - 윈속 제거의 과정을 거친다. 

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
		// To-do
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

# **2. Socket Option**

setsockopt, getsockopt를 통해 옵션을 설정하고 확인할 수 있다. 

```c++
int setsocketopt
(
	SOCKET 		s,
	int 		level,
	int 		optname,
	const char*	optval,
	int 		optlen
)
```

level은 소켓, TCP, IP 중 어떤 주체의 옵션을 설정할지 입력한다. name는 SO_KEEPALIVE, SO_LINGER, SO_RCVBUF 등 안에서 구체적으로 어떤 옵션을 설정할 것인지 입력하고, 그에 대한 구체적인 설정을 optval로 정한다. 이때 옵션의 종류에 따라 val에 들어가는 자료형이 달라지므로 모두 char*로 캐스팅하여 입력해야 한다. 그 예제는 아래와 같다. 

```c++
// KEEPALIVE option
bool enable = true;
::setsockopt(s, SOL_SOCKET, SO_KEEPALIVE, (char*)&enable, sizeof(enable));

// LINGER option
LINGER linger;
linger.l_onoff = 1;
linger.l_linger = 5;
::setsockopt(s, SOL_SOCKET, SO_LINGER, (char*)&linger, sizeof(linger));
```

LINGER는 closesocket의 수행 방식을 설정하는 옵션이다. 다른 옵션의 종류 및 LINGER에 대한 자세한 설명은 [이 포스트][1]를 참고하면 된다. 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss

[1]: https://chw-owo.github.io/procademyreview/Procademy,-Q2,-10)-Network-TCP%EC%9D%98-%EC%A2%85%EB%A3%8C,-socket-opt,-UDP/