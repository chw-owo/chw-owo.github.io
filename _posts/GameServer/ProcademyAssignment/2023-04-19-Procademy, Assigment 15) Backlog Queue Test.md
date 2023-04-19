---
title: Procademy, Assignment 15) Backlog Queue Test
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

# **과제**

**-** Backlog Queue가 어디까지 늘어날 수 있는지 확인해보고 최대한으로 늘려오기.

<br/>

# **과제 풀이**

## **1. 기본값**

server.cpp
```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <winsock2.h>
#include <stdlib.h>
#include <stdio.h>

#define SERVERPORT 9000
#define BUFSIZE 512

int wmain(int argc, wchar_t* argv[])
{
	int ret;

	// Initialize Winsock
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Generate Socket
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET)
	{
		printf("Fail to Generate Socket");
		return 0;
	}

	// Bind Socket
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
	serveraddr.sin_port = htons(SERVERPORT);
	ret = bind(listen_sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (ret == SOCKET_ERROR) 
	{
		printf("Fail to Bind Socket");
		return 0;
	}

	// Set Socket to Listen
	ret = listen(listen_sock, SOMAXCONN);
	if (ret == SOCKET_ERROR)
	{
		printf("Fail to Set Socket to Listen");
		return 0;
	}

	for(;;)
	{
        // Not Accept for Backlog Queue Test
	}

	// Close Connected Socket
	ret = closesocket(listen_sock);
	if (ret == SOCKET_ERROR)
	{
		printf("Fail to Close Socket");
		WSACleanup();
		return 0;
	}
		
	// Terminate Winsock
	WSACleanup();
	return 0;
}
```

client.cpp
```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <winsock2.h>
#include <stdlib.h>
#include <stdio.h>

#define SERVERIP "127.0.0.1"
#define SERVERPORT 9000
#define BUFSIZE 512

int recvn(SOCKET s, char* buf, int len, int flags)
{
	int received;
	char* ptr = buf;
	int left = len;

	while (left > 0)
	{
		received = recv(s, ptr, left, flags);
		if (received == SOCKET_ERROR)
			return SOCKET_ERROR;
		else if (received == 0)
			break;
		left -= received;
		ptr += received;
	}
	return (len - left);
}

int wmain(int argc, wchar_t* argv[])
{
	int cnt = 0;

	for (;;)
	{
		int ret;

		// Initialize Winsock
		WSADATA wsa;
		if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
			return 1;

		// Generate Socket
		SOCKET sock = socket(AF_INET, SOCK_STREAM, 0);
		if (sock == INVALID_SOCKET)
		{
			printf("Fail to Generate Socket");
			return 0;
		}

		// Connect Socket
		SOCKADDR_IN serveraddr;
		ZeroMemory(&serveraddr, sizeof(serveraddr));
		serveraddr.sin_family = AF_INET;
		ULONG server_sin_addr;
		inet_pton(AF_INET, SERVERIP, &server_sin_addr);
		serveraddr.sin_addr.s_addr = server_sin_addr;
		serveraddr.sin_port = htons(SERVERPORT);
		ret = connect(sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
		if (ret == SOCKET_ERROR) 
		{
			printf("Fail to Connect");
			return 0;
		}

		cnt++;
		printf("put socket no.%d in the backlog queue! \n", cnt);
		
		// Close Socket
		ret = closesocket(sock);
		if (ret == SOCKET_ERROR)
		{	
			printf("Fail to Close Socket");
			WSACleanup();
			return 0;
		}
		WSACleanup();
	}
	return 0;
}
```

listen 단계에서 backlog 큐의 최대 값을 SOMAXCONN으로 설정한 뒤 server에서 accept를 하지 않도록 했다. 이 경우 200개까지만 연결이 된다. 이 이상 연결할 일은 없다고 생각해서 200개로 제한을 걸어둔 것 같다. 

## **2. SOMAXCONN_HINT**

https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-listen

위 문서를 바탕으로 SOMAXCONN_HINT(SOMAXCONN)으로 수정했더니 약 16000개까지 연결이 된 후 "각 소켓 주소(프로토콜, 네트워크 주소, 포트)는 하나만 사용할 수 있습니다." 예외가 발생한다. 동적 포트의 기본 최대값이 약 16000개임을 고려하면 동적 포트를 모두 사용했기 때문에 더 이상 연결되지 않는 것으로 추정된다. 이 예외는 리소스가 부족하다는 뜻이지 그게 꼭 포트인 것은 아니므로 상황에 따라 다르게 해석해야 된다. 


## **3. MaxUserPort = 65535**

레지스트리 아이디에서 MaxUserPort를 수정할 경우 포트의 최대 개수인 65535개까지 연결할 수 있다. 이를 바탕으로 backlog의 크기는 유저가 사용 가능한 포트 개수와 동일한 것으로 추정된다. 

## **4. 그 외**

**-** 만약 같은 포트에 바인딩 하되 상대가 다른 포트처럼 인식하도록 제한을 건다면 계속 연결이 가능할 수도 있겠지만, WinSock API에서는 사용중인 포트엔 바인딩하지 못하도록 문법적으로 막고 있다. 

**-** 일반적으로는 accept 전담 스레드를 만들어서 backlog 큐를 그때 그때 비운다. 이 경우 초당 7-8000 건을 받으며, 초당 3000건 이상 들어오는 일은 많지 않기 때문에 일반적으로는 기본 설정인 200으로 두어도 충분하다. 

**-** 작업 관리자를 보면 backlog 큐가 늘어날 때 Nonpaged pool도 같이 늘어나는 것을 확인할 수 있다. 65535를 다 채우면 거의 1GB까지 올라간다. backlog 큐가 Nonpaged pool을 차지하는 것은, 이것이 송수신 버퍼와 마찬가지로 하드웨어 인터럽트, 정확히는 이더넷 카드의 인터럽트를 사용하기 때문이다. 이더넷 카드 인터럽트를 바탕으로 L2, L3, L4가 차례대로 작동하기 때문에 TCP가 사용하는 내부적인 버퍼는 다 Nonpaged pool을 사용하게 된다.