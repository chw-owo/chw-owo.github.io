---
title: Network Basic 02) TCP, UDP
categories: RookissGameServer
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

UDP는 신뢰성을 보장하지 않는 대신 속도가 빠르다. 유실 여부를 확인하지 않기 때문에 유실이 생겨도 그냥 넘어간다. 따라서 상대가 통신하기 어려운 상황이어도 일단 전송을 하기에 트래픽만 늘어나고 계속 유실되는 상황이 생길 수 있다. 대신 헤더의 기본 크기가 작으며, 더 빠른 속도를 보장할 수 있으므로 스트리밍에서 많이 사용된다. 

## **3) 데이터 경계**

TCP는 경계 없이 메시지를 상황에 맞게 나누거나 뭉쳐서 보낸다. 하지만 항상 1:1로 연결되며 순서를 보장하므로 L7에서 재조립하면 된다. 반면 UDP는 경계의 개념 없이 send 한 단위 그대로 전송된다. 그러나 L3에서 MTU 단위로 나누는 것은 동일하며, 송신 버퍼 크기가 부족할 경우 남는 부분이 유실되므로 이를 막기 위해 L7에서 적절한 크기로 나눠 보내는 게 좋다. 

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

이처럼 send보다 recv가 느리게 수행되면 송신 버퍼에 데이터가 쌓이다가 한번에 보내지는데, 이때 recv  반환값을 보면 쌓인 것들이 한번에 뭉쳐져서 들어온다. TCP는 트래픽을 줄이기 위해 send 단위로 메시지를 보내는 대신 이처럼 묶어서 보내는데 이를 Nagle이라 부른다. 반대로 수신 버퍼가 부족하면 메시지가 조각난 채로 보내지는데, 이렇게 조각나거나 뭉쳐진 데이터는 사용자가 L7에서 복원해야 한다. 

## **3) shutdown**

closesocket은 연결 해제와 소켓 리소스 반환을 동시에 진행하는 함수로, 일방적으로 연결 해제를 통지하는 용도이기 때문에 상대 측에서 이를 수용하지 않아도 일정 시간이 지나면 자동으로 연결이 해제된다. shutdown을 사용하면 일방적으로 연결을 해지하는 대신 내가 더 이상 데이터를 보내거나 받지 않을 것임을 통지할 수 있다. 이 경우 리소스 반환은 이루어지지 않는다. 그러나 일반적으로 잘 사용하진 않는다.  

<br/> 

# **3. UDP**

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
	SOCKET client_socket = ::socket(AF_INET, SOCK_DGRAM, 0);
	if (client_socket == INVALID_SOCKET)
	{
		HandleError("Create Socket");
		return 0;
	}

	// Set Server Address to Socket
	SOCKADDR_IN serverAddr;
	::memset(&serverAddr, 0, sizeof(serverAddr));
	serverAddr.sin_family = AF_INET;
	serverAddr.sin_addr.s_addr =
		::inet_pton(AF_INET, "127.0.0.1", &serverAddr.sin_addr);
	serverAddr.sin_port = ::htons(7777);

	// Connected UDP - Connect
	//int ret;
	//ret = ::connect(client_socket, (SOCKADDR*)&serverAddr, sizeof(serverAddr));
	//if (ret == SOCKET_ERROR)
	//{
	//	HandleError("Connect");
	//	return 0;
	//}
	
	int ret;
	while (1)
	{
		char sendBuffer[100] = "Hello World!";

		// send 시점에서 IP 주소와 port 번호가 설정된다.
		
		// Unconnected UDP - Send
		ret = ::sendto(client_socket, sendBuffer, 
			sizeof(sendBuffer), 0, (SOCKADDR*)&serverAddr, sizeof(serverAddr)); 
		
		// Connected UDP - Send
		//ret = ::send(client_socket, sendBuffer, sizeof(sendBuffer), 0);

		
		if (ret == SOCKET_ERROR)
		{
			HandleError("SendTo");
			return 0;
		}
		printf("Success to Send %d bytes!\n", sizeof(sendBuffer));
	
		int recvLen;
		char recvBuffer[1000];
		SOCKADDR_IN recvAddr;
		::memset(&recvAddr, 0, sizeof(recvAddr));
		int addrLen = sizeof(recvAddr);

		// Unconnected UDP - Receive
		recvLen = ::recvfrom(client_socket, recvBuffer, 
			sizeof(recvBuffer), 0, (SOCKADDR*)&recvAddr, &addrLen);
		
		// Connected UDP - Receive
		//recvLen = ::recv(client_socket, recvBuffer, sizeof(recvBuffer), 0);

		if (recvLen <= 0)
		{
			HandleError("RecvFrom");
			return 0;
		}
		printf("Success to Receive %u Bytes!: %s\n", recvLen, recvBuffer);
	
		Sleep(1000);
	}

	// Clean up Winsock
	::WSACleanup();
	return 0;
}
```

UDP에서도 connect 함수를 호출할 수 있는데, 이는 TCP에서의 connect 처럼 정말 소켓을 연결하는 개념은 아니고 동일한 소켓과 데이터를 계속 주고 받을 경우 즐겨찾기의 개념으로 사용할 수 있도록 주소를 저장했다가 불러오는 개념이다. 주석친 Connected UDP를 사용하더라도 내부적으로는 기존 UDP와 동일하게 동작한다. 

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

	SOCKET server_socket = ::socket(AF_INET, SOCK_DGRAM, 0);
	if (server_socket == INVALID_SOCKET)
	{
		HandleError("Create Socket");
		return 0;
	}

	SOCKADDR_IN serverAddr;
	::memset(&serverAddr, 0, sizeof(serverAddr));
	serverAddr.sin_family = AF_INET;
	serverAddr.sin_addr.s_addr = ::htonl(INADDR_ANY);
	serverAddr.sin_port = ::htons(7777);

	int ret;
	ret = ::bind(server_socket, (SOCKADDR*)&serverAddr, sizeof(serverAddr));
	if (ret == SOCKET_ERROR)
	{
		HandleError("Bind");
		return 0;
	}

	while (1)
	{
		SOCKADDR_IN clientAddr;
		::memset(&clientAddr, 0, sizeof(clientAddr));
		int clientLen = sizeof(clientAddr);

		char recvBuffer[1000];
		int recvLen = ::recvfrom(server_socket, recvBuffer,
			sizeof(recvBuffer), 0, (SOCKADDR*)&clientAddr, &clientLen);

		if (recvLen == 0)
		{
			HandleError("RecvFrom");
			return 0;
		}
		
		printf("Success to Receive %d bytes: %s\n", recvLen, recvBuffer);
	
		ret = ::sendto(server_socket, recvBuffer, 
			recvLen, 0, (SOCKADDR*)&clientAddr, sizeof(clientAddr));
		
		if (ret == SOCKET_ERROR)
		{
			HandleError("SendTo");
			return 0;
		}

		printf("Success to Send %d bytes.\n", recvLen);

	}
	// Clean up Winsock
	::WSACleanup();
	return 0;
}
```

listen, connect, accept가 사라진 것을 볼 수 있다. 서버에서 별도로 수락하는 단계가 없기 때문에 주소만 알면 누구나 데이터를 보낼 수 있으며, 따라서 방화벽 등으로 보안 처리를 해주어야 한다. 이 경우 간단한 에코 테스트용 코드이므로 send, recv가 실패할 경우 바로 종료하도록 만들었지만 실제로는 해당 주소에 대해서만 적절한 처리를 수행할 것이다. 

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
