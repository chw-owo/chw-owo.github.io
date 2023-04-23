---
title: Network Basic 05) Overlapped Model
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 동기/비동기**

![image](https://user-images.githubusercontent.com/96677719/233774187-d3b71921-bceb-4ec8-8271-3543aa153ca4.png)

Blocking/Non-Blocking, Sync/Async는 얼핏 비슷한 듯 보이지만 서로 다르다. 위 이미지는 이 차이를 간단하게 정리한 표이다. Blocking은 작업이 완료될 때까지 대기하는 함수를, Non-Blocking은 호출 즉시 반환되는 함수를 의미한다. 반면 Sync는 함수를 호출한 시점에 작업이 이루어지는 함수를, Async는 그 시점에 이루어지지 않을 수 있는 함수를 의미한다. 앞서 배운 send, recv는 Blocking/Non-Blocking에서 모두 동기 함수이다.

![image](https://user-images.githubusercontent.com/96677719/233774452-7d55bee7-675a-4812-8c78-6a72cb5c0af0.png)

![image](https://user-images.githubusercontent.com/96677719/233774447-b18b6587-34e2-4445-9db1-bd1e3deda4a2.png)


read()가 Blocking일 때와 Non-Blocking일 때 내부적으로 어떻게 다르게 동작하는지 살펴보면 위 그림과 같다. Blocking을 때는 커널 모드로 전환되어 IO 작업을 수행하며, 데이터를 유저 버퍼로 복사하는 작업까지 끝났을 때 함수를 반환한다. 반면 Non-Blocking일 때는 함수 자체는 바로 반환하는 대신 데이터를 불러올 때까지 loop를 돌면서 커널 모드 전환이 계속 일어나고, 완료되지 않은 경우 그때마다 WouldBlock을 보낸다. 

Sync 함수를 호출하면 호출한 시점에 바로 작업이 이루어진다. 반면 Async 함수를 호출한다는 것은 작업을 예약하는 것과 유사하다. 당장 처리해야 하는 작업이 아닐 때 "언젠가 처리해달라" 고 요청하는 것이기 때문에 작업 수행 시점이 보장되지 않는다. 이때 작업 수행이 완료되는 시점은 Event 통지 방식 혹은 Callback 함수 방식으로 알게 된다. 위 그림에서 Async-Blocking 방식은 잘 사용되지 않으며 대부분의 작업은 그 외의 3가지 조합에 해당된다.

<br/>

# **2. 비동기 IO의 기본**

네트워크를 포함한 장치 IO는 컴퓨터가 수행하는 작업들 중 가장 느리고 예측하기 어렵다. 따라서 비동기 IO를 사용하면 리소스 사용에 대한 성능을 개선할 수 있다. 스레드의 비동기 IO 요청은 IO 작업을 수행할 장치 (이더넷, 디스크 등)의 디바이스 드라이버로 전달되며, 드라이버가 장치로부터의 응답을 대기하다가 완료 통지를 보낸다. 따라서 스레드는 IO 요청이 완료될 때까지 대기할 필요가 없다.

```c++
typedef struct _OVERLAPPED
{
	DWORD Internal;
	DWORD InternalHigh;
	DWORD Offset;
	DWORD OffsetHigh;
	HANDLE hEvent;
}
OVERLAPPED, *LPOVERLAPPED;
```

비동기 IO를 수행할 땐 OVERLAPPED 구조체 주소를 전달한다. 이때 Overlapped은 스레드가 다른 작업을 수행하는 동안 IO 작업을 시작하는 중첩을 의미한다. Internal, Internal High는 디바이스 드라이버에 의해 설정되는 값으로 IO 완료 여부 확인에 쓸 수 있다. Offset, OffsetHigh는 파일 IO 시 파일 시작 위치 전달 용도로 사용하며 그 외의 경우 0으로 설정한다. hEvent는 Alertable IO 통지 방법에서 사용된다.

비동기 IO 요청이 완료되면 요청 시 전달했던 OVERLAPPED 구조체 주소를 돌려준다. 따라서 이에 추가적인 컨텍스트 정보를 포함시켜서 유용하게 활용할 수 있다. 예를 들어 OVERLAPPED 구조체를 상속하는 클래스를 만든 뒤, 추가적인 정보를 저장한다면 OVERLAPPED 구조체 주소를 캐스팅하여 클래스에 추가되었던 컨텍스트 정보에도 접근할 수 있게 된다. 이 방법은 Network Library 제작 파트에서 실제로 사용된다. 

이때 유의할 것은, 비동기 IO 요청 시 사용되는 데이터 버퍼와 OVERLAPPED 구조체는 IO 요청이 완료될 때까지 옮겨지거나 삭제되어서는 안된다. 디바이스 드라이버로 요청이 전달될 때 메모리 복사에 드는 시간을 절약하기 위해 실제 블록이 아닌 주소를 전달하기 때문이다. 만약 스택에 해당 버퍼와 구조체를 선언한다면 함수가 반환되어 스택에 생성되었던 값들이 삭제되지는 않는지 유의해야 한다. 

<br/>

# **3. Overlapped 모델**

Overlapped 모델은 send, recv를 Async-NonBlocking 방식으로 사용하는 모델이다. 이 경우 WSARecv, WSASend, AcceptEx, ConnectEx 등의 함수를 호출하며 WSABUF, WSAOVERLAPPED 등의 구조체를 사용한다. 함수 호출 결과는 Event 혹은 콜백으로 통지 받을 수 있다. AcceptEx, ConnectEx의 경우 설정이 다소 복잡하므로 이 포스트 예제들은 WSARecv, WSASend만 사용하고 해당 함수들은 추후에 다시 다룬다. 

<br/> 

# **4. Event 기반 통지**

우선 비동기 입출력 지원 소켓과 통지용 이벤트 객체를 생성한다. 그리고 입출력 함수 호출 시 이벤트 객체를 같이 넘긴다. 비동기 작업이 완료되지 않으면 PENDING 예외 코드가 뜨고, 완료되면 OS가 이벤트 객체를 singaled 상태로 만드는데 이는 WSAWaitForMultipleEvents 함수로 확인할 수 있다. 이후 WSAGetOvelappedResult를 호출하여 입출력 결과를 확인하고 이에 따른 데이터 처리를 한다. 

**Client.cpp**

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
	WSAEVENT wsaEvent = ::WSACreateEvent();
	WSAOVERLAPPED overlapped = {};

	while (1)
	{
		WSABUF wsaBuf;
		wsaBuf.buf = sendBuffer;
		wsaBuf.len = 100;

		DWORD sendLen = 0;
		DWORD flags = 0;
		
		ret = ::WSASend(client_socket, 
			&wsaBuf, 1, &sendLen, flags, &overlapped, nullptr);
		
		if (ret == SOCKET_ERROR)
		{
			if (::WSAGetLastError() == WSA_IO_PENDING)
			{
				::WSAWaitForMultipleEvents(1, &wsaEvent, TRUE, WSA_INFINITE, FALSE);
				::WSAGetOverlappedResult(client_socket, &overlapped, &sendLen, FALSE, &flags);
			}
			else
			{
				HandleError("WSASend");
				// To-do: 연결을 끊는 등의 예외 처리
				break;
			}
		}
		printf("Success to Send %d bytes!\n", sizeof(sendBuffer));

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
struct Session
{
	SOCKET socket;
	char recvBuffer[BUFSIZE] = {};
	int recvBytes = 0;
	WSAOVERLAPPED overlapped = {};
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

	while (1)
	{
		SOCKADDR_IN clientAddr;
		int addrLen = sizeof(clientAddr);
		SOCKET client_socket;
		while (1)
		{
			client_socket = ::accept(listen_socket, (SOCKADDR*)&clientAddr, &addrLen);
			if (client_socket != INVALID_SOCKET)
				break;
			if (::WSAGetLastError() == WSAEWOULDBLOCK)
				continue;
			return 0;
		}

		Session session = Session{ client_socket };
		WSAEVENT wsaEvent = ::WSACreateEvent();
		session.overlapped.hEvent = wsaEvent;

		printf("Success to Client Connect!\n");

		while (1)
		{
			WSABUF wsaBuf;
			wsaBuf.buf = session.recvBuffer;
			wsaBuf.len = BUFSIZE;

			DWORD recvLen = 0;
			DWORD flags = 0;
			ret = ::WSARecv(client_socket, &wsaBuf,
				1, &recvLen, &flags, &session.overlapped, nullptr);
			if (ret == SOCKET_ERROR)
			{
				if (::WSAGetLastError() == WSA_IO_PENDING)
				{
					::WSAWaitForMultipleEvents(1, &wsaEvent, TRUE, WSA_INFINITE, FALSE);
					::WSAGetOverlappedResult(session.socket, &session.overlapped, &recvLen, FALSE, &flags);
				}
				else
				{
					HandleError("WSARecv");
					// To-do: 연결을 끊는 등의 예외 처리
					break;
				}		
			}
			printf("Success to Receive %d bytes\n", recvLen);	
		}
		::closesocket(session.socket);
		::WSACloseEvent(wsaEvent);
	}

	// Clean up Winsock
	::WSACleanup();
	return 0;
}
```
<br/> 

# **5. Alertable IO**

![image](https://user-images.githubusercontent.com/96677719/233782138-a706cb3a-5ad0-4fa4-868a-3d201bbc0662.png)

시스템은 스레드별로 Async Procedure Call, APC 큐를 하나씩 생성한다. 이때 WaitForObjectEx, SleepEx등의 함수를 사용하면 비동기 IO 요청을 전달하는 함수를 호출할 때 디바이스 드라이버에게 IO 작업 완료 통지를 APC 큐에 삽입해달라고 요청할 수 있다. 이는 Completion Routine, 완료 루틴이라 불리는 콜백 함수 주소를 필요로 한다. 완료 루틴은 반드시 아래 형태로 구현되어야 한다. 

```c++
void CALLBACK CompletionRoutine 
(
    DWORD           dwError,
    DWORD           cbTransferred,
    LPWSAOVERLAPPED lpOverlapped,
    DWORD           dwFlags
);
```

APC 큐 항목을 처리할 땐 스레드가 자신을 Alertable로 변경하여 인터럽트 가능한 상태임을 알려야 한다. 그러면 시스템은 APC 큐를 확인하는데, 항목이 있을 경우 스레드를 대기 상태로 바꾸지 않고 항목 하나를 Dequeue하여 호출한다. 루틴이 반환되면 항목이 더 있는지 확인, 큐가 비워질 때까지 반복한다. 항목이 없을 때 상태를 바꾸면 스레드가 정지되며, 항목이 삽입될 때 수행을 재개한다. 

이 경우 콜백 함수를 반드시 필요로 하기 때문에 코드를 이해하기 어렵게 만든다. 또, IO 작업을 요청한 스레드가 반드시 완료 통지도 함께 처리해야 한다는 스레딩 문제가 있다. 단일 스레드가 여러번의 IO를 수행한 경우 각 IO 요청 별로 발생하는 완료 통지를 자신이 모두 처리해야 한다. 따라서 멀티스레드 환경에서 부하 분산을 제대로 수행하기 어렵다. 

<br/>

# **6. Callback 기반 통지**

앞서 설명한 APC 큐를 활용하는 방식이다. 비동기 IO가 완료되면 OS는 완료 루틴을 호출하고, 완료 루틴 호출이 끝나면 스레드가 Alertable 상태에서 빠져나오게 된다. 이 외의 기본적인 구조 및 함수 사용은 Event 기반 방식과 동일하다. 

// Example
```c++
void CALLBACK RecvCallback(DWORD error, DWORD recvLen, 
                            LPWSAOVERLAPPED overlapped, DWORD flags)
{
    printf("Success to Receive %d bytes!\n", recvLen);
}
```

이는 완료 루틴의 예시이다. 매개변수는 각각 오류 발생 시 반환 값, 전송 바이트 수, 비동기 입출력 함수 호출 시 넘겨줄 WSAOVERLAPPED 구조체의 주소, 플래그를 의미한다. 함수 포인터를 nullptr을 넘겼던 WSARecv, WSASend의 마지막 인자에 전달하면 해당 작업이 끝났을 때 위 그림처럼 APC 큐를 차례로 비우며 함수를 호출한다. Alertable 상태에 한번만 진입해도 큐를 모두 비우며, 다 비운 이후에는 Aleratble 상태에서 빠져나와 이후 코드를 실행한다. 

**Server.cpp**

```c++
int main()
{
    // ...

	while (1)
	{
		SOCKADDR_IN clientAddr;
		int addrLen = sizeof(clientAddr);
		SOCKET client_socket;
		while (1)
		{
			client_socket = ::accept(listen_socket, (SOCKADDR*)&clientAddr, &addrLen);
			if (client_socket != INVALID_SOCKET)
				break;
			if (::WSAGetLastError() == WSAEWOULDBLOCK)
				continue;
			return 0;
		}

		Session session = Session{ client_socket };
		printf("Success to Client Connect!\n");

		while (1)
		{
			WSABUF wsaBuf;
			wsaBuf.buf = session.recvBuffer;
			wsaBuf.len = BUFSIZE;

			DWORD recvLen = 0;
			DWORD flags = 0;
			ret = ::WSARecv(client_socket, &wsaBuf,
				1, &recvLen, &flags, &session.overlapped, RecvCallback);
			if (ret == SOCKET_ERROR)
			{
				if (::WSAGetLastError() == WSA_IO_PENDING)
				{
					// Set Thread Alertable Wait
					::SleepEx(INFINITE, TRUE); 
				}
				else
				{
					HandleError("WSARecv");
					// To-do: 연결을 끊는 등의 예외 처리
					break;
				}		
			}
		}
		::closesocket(session.socket);
	}

	// Clean up Winsock
	::WSACleanup();
	return 0;
}
```

이는 한번에 최대 64개만 관찰할 수 있는 이벤트 객체와 달리, SleepEx 한번으로도 쌓여있는 모든 완료 루틴을 처리할 수 있다는 점에서 더 간편하다. 그러나 AcceptEx 등은 비동기 함수임에도 콜백 함수를 사용을 지원하지는 않는다는 한계가 있다. 또 APC 큐는 그것을 가진 스레드만이 처리할 수 있어서 부하 분산을 하기 어려우며, 빈번한 Alertable Wait 인한 성능 저하 역시 문제가 된다.

```c++
struct Session
{
	WSAOVERLAPPED overlapped = {};
	SOCKET socket;
	char recvBuffer[BUFSIZE] = {};
	int recvBytes = 0;
};

void CALLBACK RecvCallback(DWORD error, DWORD recvLen,
	LPWSAOVERLAPPED overlapped, DWORD flags)
{
	printf("Success to Receive %d bytes!\n", recvLen);
	Session* session = (Session*)overlapped;
    // ...
}
```

이때 OVERLAPPED 구조체를 Session 맨 앞에 둘 경우 session의 주소와 overlapped의 주소가 일치하기 때문에 위와 같이 캐스팅 하여 사용할 수 있다. 

이 외에도 WSAAsyncSelect 모델이 있다. 이는 소켓 이벤트를 윈도우 메시지 형태로 처리하는 것으로, 그러나 일반 윈도우 메시지와 네트워크 메시지를 같이 처리하기 때문에 성능을 보장하기 어려워서 많이 사용되지는 않는다. 

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
