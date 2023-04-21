---
title: Network Basic 02) TCP, UDP
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. TCP vs UDP**

## **1) 연결 지향성**

TCP는 연결형 서비스이다. 연결을 위해 할당되는 논리적인 경로가 있으며, 이를 통해 전송 순서가 보장되는 대신 1:1 통신만 가능하다. 또, 메시지 간의 경계가 없어서 이를 사용자가 처리해주어야 한다.

UDP는 비연결형 서비스이다. 연결이라는 개념이 없기 때문에 전송 순서가 보장되지 않으며 대신 1:多 통신이 가능하다. 이 때문에 send 한 대로 메시지 간의 경계가 지켜진 채 전송이 된다. 

## **2) 속도와 신뢰성**

TCP는 높은 신뢰성을 보장하는 대신 속도가 느리고 헤더의 기본 크기가 커서 트래픽이 많이 생긴다. 응답을 통해 패킷이 잘 전송되었는지, 유실이 생겼는지 매번 확인하며 유실이 생긴 경우 재전송한다. 또, 상대가 버퍼가 가득 차는 등의 이유로 통신하기 어려운 상황이라면 일부만 보내거나 보내지 않고 대기하는 등 흐름, 혼잡 제어를 위한 기능을 수행한다. 

UDP는 신뢰성을 보장하지 않는 대신 속도가 빠르다. 유실 여부를 확인하지 않기 때문에 유실이 생겨도 그냥 넘어간다. 따라서 상대가 통신하기 어려운 상황이어도 일단 전송을 하며, 유실 되는 것은 고려하지 않는다. 대신 헤더의 기본 크기가 작아서 트래픽이 줄어들며, 더 빠른 속도를 보장할 수 있으므로 스트리밍에서 많이 사용된다. 

## **3)**

# **2. TCP**

## **1) recv, send와 block**

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

recv() 역시 직접 수신을 하는 게 아니라 recv buffer에 들어온 메시지를 꺼내오는 작업을 한다. 하지만 이 경우 send와는 달리 recv buffer에 메시지가 없다면 작업을 수행할 수 없으므로 메시지가 들어올 때까지 block이 걸린다. block 걸리지 않았을 때는 recv buffer에서 사용자 버퍼로 복사가 이루어지고, block이 걸렸을 땐 TCP가 사용자 버퍼에 직접 데이터를 입력하다가 입력이 끝나면 return 된다.

## **2) TCP의 Nagle**

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

		for(int i = 0; i < 10; i++)
		{
			ret = ::send(client_socket, sendBuffer, sizeof(sendBuffer), 0); 
			if (ret == SOCKET_ERROR)
			{
				err = ::WSAGetLastError();
				printf("Fail to Send. Socket Error: %d\n", err);
				return 0;
			}
			printf("Success to Send %u Bytes!", sizeof(sendBuffer));
		}
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
            Sleep(1000);
			char recvBuffer[1000];
			ret = ::recv(client_socket, recvBuffer, sizeof(recvBuffer), 0);
			if (ret <= 0)
			{
				err = ::WSAGetLastError();
				printf("Fail to Receive. Socket Error: %d\n", err);
				return 0;
			}
			printf("Success to Receive %u Bytes!: %s", ret, recvBuffer);
        }

		::closesocket(client_socket);
	}

	// Clean up Winsock
	::closesocket(listen_socket);
	::WSACleanup();
	return 0;
}
```

위와 같이 send는 연달아서 발생하는데 recv는 띄엄띄엄 발생하는 경우 송신 버퍼에 데이터가 쌓였다가 한번에 보내지는데, 이때 recv 측의 반환값을 확인해보면 쌓였던 크기만큼 한번에 뭉쳐져서 데이터가 도착하는 것을 볼 수 있다. TCP에서는 트래픽을 줄이기 위해 send 한 단위로 메시지를 보내는 대신 이처럼 묶어서 보내기도 하는데 이를 Nagle이라 부른다. 이렇게 뭉쳐진 데이터는 사용자 측에서 분리해주어야 한다. 반대로 수신 버퍼가 충분하지 못한 경우엔 메시지가 조각난 채로 보내지는데 이 역시 사용자 측에서 처리해야 한다. 

<br/> 

# **3. UDP**

```c++
```

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
