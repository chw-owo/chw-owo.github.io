---
title: Network Basic 04) Select, WSAEventSelect
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Select**

Select 모델은 select 함수를 활용하여 소켓 함수 호출이 완료되는 시점을 미리 파악해서 활용하는 모델이다. 이는 Blocking, Non-Blocking 두 상황 모두 사용할 수 있다. 일반적으로 Blocking에서는 조건이 만족되지 않아 Block되는 상황을, Non-Blocking에서는 조건이 만족되지 않아 불필요하게 loop를 돌며 CPU가 낭비되는 상황을 예방하기 위해 사용한다. 

우선 socket set으로 read/write/except 중 관찰 대상 작업을 등록하고, select를 통해 해당 작업이 수행 가능한지 관찰한다. 적어도 하나의 소켓이 준비되면 준비된 개수를 반환하며 준비되지 못한 소켓들은 알아서 제거된다. 이후에는 준비된 것들을 대상으로 해당 작업을 수행한다. 이를 통해 불필요한 대기 및 loop를 방지할 수 있다. 이는 윈도우, 리눅스 공통으로 사용할 수 있다. 

이를 위해 필요한 기본적인 문법은 아래와 같다. 

|fd_set set;|기본 세팅|
|FD_ZERO(&set);|비우기|
|FD_SET(s, &set);|소켓 넣기|
|FD_CLR(s, &set);|소켓 제거|
|FD_ISSET(s, &set);| 소켓 s가 set에 있으면 0이 아닌 값을 반환|

이를 활용한 에코 서버 예제는 아래와 같다. 

**Server.cpp**
```c++
#include "pch.h"
#include "CorePch.h"

#include <vector>
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

const int BUFSIZE = 1000;
struct Session
{
	SOCKET socket;
	char recvBuffer[BUFSIZE] = {};
	int recvBytes = 0;
	int sendBytes = 0;
};
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

	vector<Session> sessions;
	sessions.reserve(100);

	fd_set reads;
	fd_set writes;

	while (true)
	{
		// Initialize Set
		FD_ZERO(&reads);
		FD_ZERO(&writes);

		// Register sockets
		FD_SET(listen_socket, &reads);
		for (Session& s : sessions)
		{
			if (s.recvBytes <= s.sendBytes)
				FD_SET(s.socket, &reads);
			else
				FD_SET(s.socket, &writes);
		}

		ret = ::select(0, &reads, &writes, nullptr, nullptr);
		if (ret == SOCKET_ERROR)
		{
			HandleError("Select");
			break;
		}

		// Check listen_socket
		if (FD_ISSET(listen_socket, &reads))
		{
			SOCKADDR_IN clientAddr;
			int addrLen = sizeof(clientAddr);
			SOCKET client_socket = ::accept(
				listen_socket, (SOCKADDR*)&clientAddr, &addrLen);

			if (client_socket != INVALID_SOCKET)
			{
				printf("Client Connected\n");
				sessions.push_back(Session{ client_socket });
			}
		}

		// Check sockets
		for (Session& s : sessions)
		{
			// Check Read
			if (FD_ISSET(s.socket, &reads))
			{
				int recvLen = ::recv(s.socket, s.recvBuffer, BUFSIZE, 0);
				if (recvLen <= 0)
				{
					//To-do: sessions 제거
					continue;
				}
				s.recvBytes = recvLen;
			}

			// Check Read
			if (FD_ISSET(s.socket, &writes))
			{
				// Blocking이라면 요청 받은 모든 데이터를 보낸 뒤 send가 return 되지만
				// Non-Blocking이라면 상대 수신 버퍼 상황에 따라 일부만 보내기도 한다. 

				int sendLen = ::send(s.socket, &s.recvBuffer[s.sendBytes], 
					s.recvBytes - s.sendBytes, 0);

				if (sendLen == SOCKET_ERROR)
				{
					//To-do: sessions 제거
					continue;
				}
				s.sendBytes += sendLen;
				if (s.recvBytes == s.sendBytes)
				{
					s.recvBytes = 0;
					s.sendBytes = 0;
				}
			}
		}
	}

	// Clean up Winsock
	::WSACleanup();
	return 0;
}
```

select를 호출하면 그 시점에서 준비되지 않았던 소켓들은 제거되기 때문에 루프를 돌 때마다 FD_ZERO, FD_SET을 다시 해주어야 한다. 이 상황은 에코 서버이므로 read 혹은 write로 나누어 등록하지만 상황에 따라 유연하게 사용해야 한다. 이를 통해 준비된 소켓에 대해서만 recv, send를 호출할 수 있다. select는 동기 함수로 동작하며, 매번 초기화를 새로 해주어야 되므로 성능 저하가 유발된다는 것, set 하나 당 64개 밖에 등록할 수 없다는 것 등의 한계가 있다. 

<br/> 

# **2. WSAEventSelect**

WSAEventSelect 모델은 앞서 말한 select 모델의 단점을 보완한 것으로 윈도우에서 제공하는 비동기 함수 WSAEventSelect를 사용한다. 이 함수를 호출하면 해당 소켓은 자동으로 Non-Blocking 모드로 전환된다. 큰 구조는 Select 모델과 유사하지만 이는 소켓과 관련된 네트워크 이벤트를 이벤트 객체를 통해 감지하며 소켓 하나 당 이벤트 객체를 하나씩 만들어서 연동한다. 사용되는 주요 함수들은 아래와 같다. 

|WSACreateEvent|수동 리셋, Non-Signaled 상태로 시작|
|WSACloseEvent|이벤트 객체 삭제|
|WSAWaitForMultipleEvents|신호 상태 감지|
|WSAEventSelect|소켓, 이벤트, 감지하려는 이벤트 종류를 입력하여 소켓과 이벤트 객체 연동|
|WSAWaitForMultipleEvents|이벤트 발생 통지를 받을 객체 등록|
|WSAEnumNetworkEvents|구체적인 이벤트 상황 파악|

네트워크 이벤트 종류는 아래와 같다. 

|FD_ACCEPT|클라이언트 접속 (accept)|
|FD_READ|데이터 수신 가능 (recv, recvfrom)|
|FD_WRITE|데이터 송신 가능 (send, sendto)|
|FD_CLOSE|상대의 접속 종료|
|FD_CONNECT|통신을 위한 연결 절차 완료|

이때 주의사항은, accept() 함수가 반환하는 소켓은 listen_socket과 동일한 속성을 갖는다. 따라서 client_socket은 FD_READ, FD_WRITE 등을 다시 등록해야 한다. 드물게 WSAEWOULDBLOCK 예외가 발생하기도 하니 이에 대한 예외 처리가 필요하다. 또 이벤트 발생 시 적절한 소켓 함수를 제때 호출해야 한다. 그러지 않으면 다음번에 해당 네트워크 이벤트가 발생하지 않는다. 

이를 활용한 예제는 아래와 같다. 

**Server.cpp**

```c++
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
	vector<WSAEVENT> wsaEvents;
	vector<Session> sessions;
	sessions.reserve(100);

	WSAEVENT listen_event = ::WSACreateEvent();
	wsaEvents.push_back(listen_event);
	sessions.push_back(Session{ listen_socket });

	ret = ::WSAEventSelect(listen_socket,
		listen_event, FD_ACCEPT | FD_CLOSE);

	if (ret == SOCKET_ERROR)
	{
		HandleError("Register Event");
		return 0;
	}

	while (1)
	{
		int idx = ::WSAWaitForMultipleEvents(wsaEvents.size(), 
			&wsaEvents[0], FALSE, WSA_INFINITE, FALSE);
		if (idx == WSA_WAIT_FAILED)
			continue;

		idx -= WSA_WAIT_EVENT_0;

		//::WSAResetEvent(wsaEvents[idx]);
		// 어차피 WSAEnumNetworkEvents에서 reset 하므로 있어도 되고 없어도 된다.

		WSANETWORKEVENTS networkEvents;
		ret = ::WSAEnumNetworkEvents(sessions[idx].socket,
			wsaEvents[idx], &networkEvents);
		if (ret == SOCKET_ERROR)
			continue;

		if (networkEvents.lNetworkEvents & FD_ACCEPT)
		{
			if (networkEvents.iErrorCode[FD_ACCEPT_BIT] != 0)
				continue;

			SOCKADDR_IN clientAddr;
			int addrLen = sizeof(clientAddr);
			SOCKET client_socket = ::accept(listen_socket,
				(SOCKADDR*)&clientAddr, &addrLen);
			if (client_socket != INVALID_SOCKET)
			{
				printf("Client Connected!\n");
				
				WSAEVENT client_event = ::WSACreateEvent();
				wsaEvents.push_back(listen_event);
				sessions.push_back(Session{ listen_socket });
				ret = ::WSAEventSelect(listen_socket,
					listen_event, FD_READ | FD_WRITE | FD_CLOSE);
				if (ret == SOCKET_ERROR)
				{
					HandleError("Event Select");
					return 0;
				}
			}
		}

		// Check Client Session Socket
		if (networkEvents.lNetworkEvents & FD_READ ||
			networkEvents.lNetworkEvents & FD_WRITE)
		{
			// Check Error
			if (((networkEvents.lNetworkEvents & FD_READ) &&
				networkEvents.iErrorCode[FD_READ_BIT] != 0) ||
				((networkEvents.lNetworkEvents & FD_WRITE) &&
					networkEvents.iErrorCode[FD_WRITE_BIT] != 0))
				continue;

			Session& s = sessions[idx];
			if (s.recvBytes == 0)
			{
				int recvLen = ::recv(s.socket, s.recvBuffer, BUFSIZE, 0);
				if (recvLen == SOCKET_ERROR && WSAGetLastError() != WSAEWOULDBLOCK)
				{
					// To-do: Remove Session
					continue;
				}
				s.recvBytes = recvLen;
				printf("Success to Receive %d bytes!", recvLen);
			}

			if (s.recvBytes > s.sendBytes)
			{
				int sendLen = ::send(s.socket, &s.recvBuffer[s.sendBytes], 
					s.recvBytes - s.sendBytes, 0);

				if (sendLen == SOCKET_ERROR && WSAGetLastError() != WSAEWOULDBLOCK)
				{
					// To-do: Remove Session
					continue;
				}
				s.sendBytes += sendLen;
				if (s.recvBytes == s.sendBytes)
				{
					s.recvBytes = 0;
					s.sendBytes = 0;
				}
				printf("Success to Send %d bytes!", sendLen);
			}
		}
		
		if (networkEvents.lNetworkEvents & FD_CLOSE)
		{
			// To-do: Remove Session
		}
	}

	// Clean up Winsock
	::WSACleanup();
	return 0;
}
```

select와 달리 매번 다시 초기화를 하지 않아도 된다. 또 WaitForMultipleEvent를 이용하기 때문에 한번에 객체들을 관찰할 수 있다. 그러나 이 역시 select와 마찬가지로 최대 64개까지 한번에 관찰할 수 있다는 한계가 있다. 또 커널 객체인 Event 객체를 매번 만들고 등록하는 과정에서 오버헤드가 생긴다. 이런 이유로 게임 클라이언트 쪽에서는 간혹 사용되지만 Stateful 서버 쪽에서는 많이 사용하지 않는다.

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
