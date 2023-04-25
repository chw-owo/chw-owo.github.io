---
title: Socket Model 1) Select Model
categories: TCPIPWindowSocket
tags: 
toc: true
toc_sticky: true
---

# **1. Socket IO Model**

## **1) Socket Model이란?**

소켓 입출력 모델은 다수의 소켓을 관리하고 IO를 처리하는 방식을 의미한다. 이를 통해 리소스를 적게 사용하면서도 다수의 클라이언트를 효율적으로 처리하는 서버를 만들 수 있다. 

## **2) Socket Mode**

소켓은 소켓 함수 동작 방식에 따라 Blocking, Non-Blocking으로 구분된다. 

Blocking 소켓은 소켓의 가장 기본적인 mode로, 함수 호출 시 조건이 만족되지 않으면 스레드가 정지하며 조건이 만족되어야 함수가 리턴하면서 스레드 실행을 재개한다. 소켓 함수 별 리턴하는 조건은 아래와 같다. 

|소켓 함수|리턴 조건|
|--------|--------|
|accept()|접속한 클라이언트가 있을 때|
|connect()|서버에 접속을 성공했을 때|
|send(), sendto()|응용 프로그램이 전송을 요청한 데이터를 소켓 송신 버퍼에 모두 복사했을 때|
|recv(), recvfrom()| 소켓 수신 버퍼에 도착한 데이터가 1byte 이상 있고 이를 응용 프로그램이 제공한 버퍼에 복사했을 때|

Non-Blocking 소켓은 조건이 만족되지 않더라도 스레드 중단 없이 일단 함수가 리턴한다. 이때 조건이 만족되지 않은 상황이면 WSAEWOULDBLOCK 오류 코드를 반환하므로 이 경우 나중에 다시 소켓 함수를 호출해주어야 한다. Non-Blocking 소켓을 사용하기 위해서는 아래와 같이 ioctlsocket()을 호출해 소켓 모드를 변경해야 한다. 

```c++
SOCKET sock = socket(AF_INET, SOCK_STREAM, 0);
if(sock == INVALID_SOCKET) _HandleError;

u_long on = 1;
ret = ioctlsocket(sock, FIONBIO, &on);
if(ret == SOCKET_ERROR) _HandleError;
```

동일한 기능을 두가지 모드로 구현, 비교해보면 논블락에서 더 높은 CPU 사용률을 보인다. 이는 조건이 만족되지 않아도 스레드가 정지 없이 다음 코드를 실행하기 때문이며, 덕분에 스레드가 오랜 시간 정지하는 상황을 없앨 수 있다. 이를 활용해 멀티스레드 없이도 여러 소켓에 대해 돌아가며 입출력 처리를 하거나 다른 작업들을 병행할 수 있다.

반면 함수마다 WOULDBLOCK을 확인하고 처리해야 하므로 프로그램이 복잡해진다는 단점이 있다. 또 앞서 말한 높은 CPU 사용률 역시 단점이 된다. 간단한 송수신 작업의 경우 WOULDBLOCK 처리 로직 때문에 오히려 성능이 더 안좋게 나오기도 한다. 이때 소켓 IO 모델을 사용하면 장점은 유지하되 단점은 해결할 수 있다. 

## **3) Non-Blocking Socket 예제**

**Server.cpp**
```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <stdio.h>

#define SERVERPORT 9000
#define BUFSIZE 512

void HandleError(int line)
{
	int err = ::WSAGetLastError();
	wprintf(L"Error! Line %d: %d", line, err);
}
#define _HandleError HandleError(__LINE__)

int main(int argc, char* argv[])
{
	int ret;
	int err;

	// Initialize Winsock
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create Socket
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET)
	{
		_HandleError;
		return 0;
	}

	// Set Socket Non-Blocking Mode
	u_long on = 1;
	ret = ioctlsocket(listen_sock, FIONBIO, &on)5;
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Bind Socket
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, L"0.0.0.0", &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);
	ret = bind(listen_sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Socket to Listen
	ret = listen(listen_sock, SOMAXCONN);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	SOCKET client_sock;
	SOCKADDR_IN clientaddr;
	int addrlen;
	char buf[BUFSIZE + 1];

	while (1)
	{
	ACCEPT_AGAIN:
		// Accept Client Socket
		addrlen = sizeof(clientaddr);
		client_sock = accept(listen_sock, (SOCKADDR*)&clientaddr, &addrlen);
		if (client_sock == INVALID_SOCKET)
		{
			err = ::WSAGetLastError();
			if (err == WSAEWOULDBLOCK)
				goto ACCEPT_AGAIN;
			wprintf(L"Error! Line %d: %d", __LINE__, err);
			break;
		}

		WCHAR szClientIP[16] = { 0 };
		InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
		wprintf(L"\n[TCP Server] Accept Client: IP=%s, port=%d\n",
			szClientIP, ntohs(clientaddr.sin_port));

		// Receive Data from Client
		while (1)
		{
		RECEIVE_AGAIN:
			ret = recv(client_sock, buf, BUFSIZE, 0);
			if (ret == SOCKET_ERROR)
			{
				err = ::WSAGetLastError();
				if (err == WSAEWOULDBLOCK)
					goto RECEIVE_AGAIN;
				wprintf(L"Error! Line %d: %d", __LINE__, err);
				break;
			}
			else if (ret == 0)
				break;

			// Print Received Data
			buf[ret] = '\0';
			wprintf(L"[TCP/%s:%d]", szClientIP, ntohs(clientaddr.sin_port));
			printf(" %s\n", buf);

		SEND_AGAIN:
			// Send Data to Client
			ret = send(client_sock, buf, ret, 0);
			if (ret == SOCKET_ERROR)
			{
				err = ::WSAGetLastError();
				if (err == WSAEWOULDBLOCK)
					goto SEND_AGAIN;
				wprintf(L"Error! Line %d: %d", __LINE__, err);
				break;
			}
		}

		// Close Client Socket
		closesocket(client_sock);
		wprintf(L"[TCP Server] Close client socket: IP=%s, port=%d\n",
			szClientIP, ntohs(clientaddr.sin_port));
	}

	// Close Server Socket
	closesocket(listen_sock);
	WSACleanup();
	return 0;
}
```

이때 listen_sock이 논블로킹이면 client_sock도 자동으로 논블로킹이 되기 때문에 ioctlsocket 과정을 한번만 거쳐도 괜찮다. 

<br/>

# **2. 서버 작성 모델의 종류**

## **1) 반복 서버 vs 병행 서버**

반복 서버는 클라이언트를 한번에 하나씩 처리한다. 스레드 한 개로 구현하므로 리소스 소모가 적지만, 한 클라이언트의 처리 시간이 길어지면 다른 클라이언트는 대기하고 있어야 한다. 각 클라이언트와 아주 짧은 시간만 통신하거나 동시 접속 수에 제한이 있는 경우 혹은 연결 개념이 없는 UDP 서버를 작성하는 경우에 적합하다. 

병행 서버는 여러 클라이언트를 동시에 처리한다. 멀티스레드로 구현한 TCP 서버가 이 예시이다. 한 클라이언트의 처리 시간이 길어져도 다른 클라이언트에게 영향을 주지 않는 장점이 있지만, 스레드를 여러개 생성하므로 리소스 소모가 크다는 단점이 있다. 윈도우로 구현하는 대부분의 서버는 병행 서버를 사용한다. 

## **2) 이상적인 소켓 IO 모델**

이상적인 소켓 IO 모델에서는 논블로킹 함수 호출이 항상 성공하도록 만들어서 WOULDBLOCK과 같은 예외 처리를 최소로 만들고 CPU 사용률을 낮춰야 한다. 

또, 스레드 개수가 너무 많아져서 메모리 사용량 및 스위칭이 너무 자주 일어나지 않도록 해주어야 한다. 동시 실행 가능한 스레드 수는 CPU 개수와 일치하므로 이에 맞춰 스레드를 만드는 게 좋다. 

마지막으로, 유저 모드와 커널 모드 전환을 최소화하고 CPU 명령 수행과 IO 작업은 병행하는 것이 좋다. 하드웨어는 병렬 동작이 가능하므로 CPU 명령 수행(코드 실행)과 IO 작업(네트워크)을 동시에 진행할 수 있다. 

## **3) 소켓 IO 모델의 종류**

윈도우 운영 체제에서는 Select, WSAAsyncSelect, WSAEventSelect, Overlapped, Completion Port 총 5가지의 소켓 IO 모델을 제공한다. 

<br/>

# **3. Select 모델**

## **1) 동작 원리**

Select 모델은 select() 함수를 사용하는 모델로, 소켓 모드와 상관 없이 소켓 함수 호출 성공 시점을 미리 파악하여 여러 소켓을 한 스레드로 처리할 수 있다. Blocking 일 때는 함수 호출 시 조건이 만족되지 않아 Block 되는 상황을 막을 수 있으며, Non-Blocking 일 때는 조건이 만족되지 않고 반환되어 재호출 해야 하는 상황을 막을 수 있다. 

Select 모델은 socket set 세개(read set, write set, except set)를 준비하여 함수에 따라 소켓을 적절한 set에 넣는다. 그 후 select()를 호출하면 set에 있는 소켓들이 IO를 위한 준비가 될 때까지 대기하다가 적어도 한 소켓이 준비된 시점에 return 되며, 이때 set에는 IO 가능한 소켓만 남고 나머지는 모두 제거된다. 

이런 작업을 통해 소켓 함수의 성공적 호출 시점을 알 수 있다. 즉, select()가 반환되는 시점에 set에 남아있는 소켓들에 대해서만 소켓 함수를 호출하면 간단하게 블락 혹은 재호출 없이 원하는 작업을 할 수 있는 것이다. 또, 드문 상황이지만 소켓 set을 함수의 호출 결과를 확인할 수도 있다. 

## **2) 구성 요소**

|set|시점/결과|역할|
|---|--------|---|
|read set|함수 호출 시점|접속한 클라이언트가 있을 때 accept()를 호출한다. 또, 소켓 수신 버퍼에 도착한 데이터가 있을 때 혹은 TCP 연결이 종료 되었을 때 recv(), recvfrom()를 호출해 도착한 데이터 및 연결 종료를 확인할 수 있다.|
|write set|함수 호출 시점|소켓 송신 버퍼 공간이 충분할 때 send(), sendto()를 호출하여 데이터를 보낼 수 있다.|
|write set|함수 호출 결과|논블로킹 소켓을 사용한 connect() 호출이 성공했다.|
|exception set|함수 호출 시점|OOB 데이터가 도착했으므로 recv(), recvfrom()을 호출하여 OOB 데이터를 받을 수 있다.|
|exception set|함수 호출 결과|논블로킹 소켓을 사용한 connect() 호출이 실패했다.|

소켓 set의 역할은 위와 같다. 

```c++
int select
(
    int nfds,
    fd_set *readfds,
    fd_set *writefds,
    fd_set *exceptfds,
    const struct timeval *timeout
);
```

select() 함수 원형은 위와 같다. 

nfds는 리눅스 호환을 위한 인자로 윈도우에서는 무시된다. fd_set 인자들은 각각 read set, write set, except set을 의미하며 사용하지 않으면 null을 전달한다. timeout이 IO 가능한 소켓이 없어도 무조건 return 한다. 여기에 null을 전달하면 무한히 기다린다. 0을 전달하면 바로, 양수를 입력하면 그 시간만큼 기다린 후 모든 소켓을 검사하여 조건을 만족한 소켓의 개수 (없을 경우 0)를 return 한다. 

select()를 이용한 소켓 입출력 절차는 소켓 셋 초기화 - 소켓 셋에 소켓 삽입 - select() 호출 - 셋에 남은 모든 소켓에 대해 적절한 함수 호출 순으로 이루어진다. 이 과정을 반복한다. 즉, 적절한 함수를 호출하고 나면 매번 소켓 셋 초기화, 소켓 삽입을 다시 해주어야 한다. 이때 set 당 넣을 수 있는 소켓 최대 개수는 64로 정해져있다. 

|매크로 함수|기능|
|----------|----|
|FD_ZERO(fd_set *set)|set을 비운다(초기화)|
|FD_SET(SOCKET s, fd_set *set)|set에 소켓을 넣는다|
|FD_CLR(SOCKET s, fd_set *set)|set에서 소켓을 제거한다|
|FD_ISSET(SOCKET s, fd_set *set)|소켓이 set에 들어 있으면 0이 아닌 값을 반환한다.|

Socket Set 조작 매크로 함수는 다음과 같다. 

## **3) 예제**

**Server.cpp**

```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <stdio.h>

#define SERVERPORT 9000
#define BUFSIZE 512

void HandleError(int line)
{
	int err = ::WSAGetLastError();
	wprintf(L"Error! Line %d: %d", line, err);
}
#define _HandleError HandleError(__LINE__)

// Socket Info
struct SOCKETINFO
{
	SOCKET sock;
	char buf[BUFSIZE + 1];
	int recvbytes;
	int sendbytes;
};

int sockCnt = 0;
SOCKETINFO* SocketInfoArray[FD_SETSIZE];
BOOL AddSocketInfo(SOCKET sock)
{
	if (sockCnt >= FD_SETSIZE)
	{
		wprintf(L"[Error] Socket Set is full.\n");
		return FALSE;
	}

	SOCKETINFO* ptr = new SOCKETINFO;
	if (ptr == NULL)
	{
		wprintf(L"[Error] Can not Alloc Memory.\n");
		return FALSE;
	}

	ptr->sock = sock;
	ptr->recvbytes = 0;
	ptr->sendbytes = 0;

	SocketInfoArray[sockCnt] = ptr;
	sockCnt++;

	return TRUE;
}

void RemoveSocketInfo(int idx)
{
	SOCKETINFO* ptr = SocketInfoArray[idx];

	SOCKADDR_IN clientaddr;
	WCHAR szClientIP[16] = { 0 };
	int addrlen = sizeof(clientaddr);
	getpeername(ptr->sock, (SOCKADDR*)&clientaddr, &addrlen);
	InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
	wprintf(L"[TCP Server] Disconnect Client %s:%d!\n", 
		szClientIP, ntohs(clientaddr.sin_port));

	closesocket(ptr->sock);
	delete ptr;

	if (idx != (sockCnt - 1))
		SocketInfoArray[idx] = SocketInfoArray[sockCnt - 1];
	
	sockCnt--;
}
```
```c++
int main(int argc, char* argv[])
{
	int ret;
	int err;

	// Initialize Winsock
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create Socket
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET)
	{
		_HandleError;
		return 0;
	}

	// Set Socket Non-Blocking Mode
	u_long on = 1;
	ret = ioctlsocket(listen_sock, FIONBIO, &on);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Bind Socket
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, L"0.0.0.0", &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);
	ret = bind(listen_sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Socket to Listen
	ret = listen(listen_sock, SOMAXCONN);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	FD_SET rset, wset;
	SOCKET client_sock;
	SOCKADDR_IN clientaddr;
	int addrlen;
	char buf[BUFSIZE + 1];


	while (1)
	{
		// Socket Set Setting 
		FD_ZERO(&rset);
		FD_ZERO(&wset);
		FD_SET(listen_sock, &rset);
		for (int i = 0; i < sockCnt; i++)
		{
			if (SocketInfoArray[i]->recvbytes >
				SocketInfoArray[i]->sendbytes)
				FD_SET(SocketInfoArray[i]->sock, &wset);
			else
				FD_SET(SocketInfoArray[i]->sock, &rset);
		}

		// Select 
		ret = select(0, &rset, &wset, NULL, NULL);
		if (ret == SOCKET_ERROR)
		{
			_HandleError;
			break;
		}

		// Accept Client Socket
		WCHAR szClientIP[16] = { 0 };
		if (FD_ISSET(listen_sock, &rset))
		{
			addrlen = sizeof(clientaddr);
			client_sock = accept(listen_sock, (SOCKADDR*)&clientaddr, &addrlen);
			if (client_sock == INVALID_SOCKET)
			{
				_HandleError;
				break;
			}
			
			InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
			wprintf(L"\n[TCP Server] Accept Client: IP=%s, port=%d\n",
				szClientIP, ntohs(clientaddr.sin_port));
			AddSocketInfo(client_sock);
		}
		
		
		// Receive Data from Client
		for(int i = 0; i < sockCnt; i++)
		{
			SOCKETINFO* ptr = SocketInfoArray[i];
			if (FD_ISSET(ptr->sock, &rset))
			{
				ret = recv(ptr->sock, ptr->buf, BUFSIZE, 0);
				if (ret == SOCKET_ERROR)
				{
					_HandleError;
					RemoveSocketInfo(i);
					continue;
				}
				else if (ret == 0)
				{
					RemoveSocketInfo(i);
					continue;
				}

				// Print Received Data
				ptr->recvbytes = ret;
				ptr->buf[ret] = '\0';

				addrlen = sizeof(clientaddr);
				getpeername(ptr->sock, (SOCKADDR*)&clientaddr, &addrlen);
				wprintf(L"[TCP/%s:%d]", szClientIP, ntohs(clientaddr.sin_port));
				printf(" %s\n", buf);

				if (FD_ISSET(ptr->sock, &wset))
				{
					// Send Data to Client
					ret = send(ptr->sock, ptr->buf + ptr->sendbytes, 
								ptr->recvbytes - ptr->sendbytes, 0);
					if (ret == SOCKET_ERROR)
					{
						_HandleError;
						RemoveSocketInfo(i);
						continue;
					}
					ptr->sendbytes += ret;
					if (ptr->recvbytes == ptr->sendbytes)
						ptr->recvbytes = ptr->sendbytes = 0;
				}
			}
		}
	}

	WSACleanup();
	return 0;
}
```

위 예제는 Non-Blocking 예제처럼 모든 소켓이 논블로킹이지만 CPU 사용률이 매우 낮게 나온다. Select 모델은 블로킹, 논블로킹 모두 사용할 수 있지만 논블로킹에서 사용하는 것이 더 효율적이다. 단, 논블로킹을 쓰면 send()에서 지정 값보다 작은 값이 return 될 수 있다. 

예제에서 볼 수 있듯이 select() 함수는 함수 호출 시점 및 결과를 알려줄 뿐 어떤 소켓인지는 알려주지 않기 때문에 모든 소켓에 대해 셋에 들어있는지 여부를 일일이 확인해야 한다. 버퍼, 바이트 정보 등과 같은 소켓 정보의 관리 역시 사용자가 직접 구현해야 한다. 

<br/>

# **3. WSAAsyncSelect 모델**

## **1) 동작 원리**

WSAAsyncSelect() 함수를 통해 소켓 관련 네트워크 이벤트를 윈도우 메시지로 받을 수 있다. 이를 통해 소켓 함수 호출 성공 시점을 윈도우 메시지 수신으로 알 수 있게 된다. 이때 모든 메시지가 한 윈도우 프로시저에 전달되므로 단일 스레드로도 여러 소켓을 처리할 수 있다. 

WSAAsyncSelect()로 네트워크 이벤트와 이를 알려줄 윈도우 메시지를 등록할 수 있는데 그러면 이벤트 발생 시 윈도우 메시지가 발생하여 윈도우 프로시저가 호출된다. 이후 메시지 종류에 따라 적절한 소켓함수를 호출하면 함수 조건이 만족되지 않아 생기는 문제를 해결할 수 있다. 

## **2) 구성 요소**

```c++
int WSAAsyncSelect
(
    SOCKET s,
    HWND hWnd,
    unsigned int wMsg,
    long lEvent
);
```
WSAAsyncSelect() 함수 원형은 위와 같다. 

```c++
#define WM_SOCKET (WM_USER + 1) // 사용자 정의 메시지
WSAAsyncSelect(s, hWnd, WM_SOCKET, FD_READ|FD_WRITE);
```

순서대로 네트워크 이벤트를 처리할 소켓, 메시지를 받을 윈도우의 핸들, 이벤트 발생 시 받을 메시지, 이벤트의 종류를 전달한다. 이벤트 종류는 FD_ACCEPT, FD_READ, FD_WRITE 등을 비트 마스크 조합으로 나타낸다. 예제는 이를 이용해 read, write 이벤트를 등록하는 코드이다. 

블로킹 소켓은 윈도우 메시지 루프를 정지시킬 수 있기 때문에 WSAAsyncSelect 호출 시 해당 소켓은 자동으로 논블로킹 모드로 전환된다. 윈도우 메시지에 등하여 소켓 함수를 호출하면 대부분 성공하지만 간혹 WouldBlock 코드가 발생할 수도 있으니 처리해주어야 한다. 

또, accept()가 반환하는 소켓은 연결 대기 소켓과 동일 속성을 갖게 된다. 연결 대기 소켓은 직접 데이터 송수신을 하지 않으므로 FD_READ, FD_WRITE 이벤트를 처리하지 않는 반면, accept()가 반환하는 소켓은 이를 처리해야 하므로 다시 WSAAsycnSelect()를 호출하여 이벤트를 등록해야 한다. 

윈도우 메시지를 받고 대응 함수를 호출하지 않으면 더 이상 같은 메시지가 발생하지 않는다. 따라서 바로 대응 함수를 호출해야 하며, 그렇지 않을 경우 이후엔 PostMessage()로 윈도우 메시지 큐에 직접 메시지를 넣어야 한다. 이를 두고 응용 프로그램이 직접 메시지를 발생시킨다고 한다. 

|네트워크 이벤트|대응 함수|
|--------------|--------|
|FD_ACCEPT|accept()|
|FD_READ|recv(), recvfrom()|
|FD_WRITE|send(), sendto()|
|FD_CLOSE|없음|
|FD_CONNECT|없음|
|FD_OOB|recv(), recvfrom()|

네트워크 이벤트 별 적절한 대응 함수는 위와 같다. 

```c++
LRESULT CALLBACK WndProc (HWND hWnd, UINT yMsg, WPARAM wParam, LPARAM lParam)
{
    // ...
}
```

윈도우 프로시저는 위와 같이 총 네개의 인자로 데이터를 전달받는다. 순서대로 메시지가 발생한 윈도우의 핸들, 등록했던 사용자 정의 메시지, 네트워크 이벤트가 발생한 소켓, 네트워크 이벤트와 오류 코드가 전달된다. 

소켓의 경우 SOCKET 형변환 이후 사용해야 한다. lParam은 하위 16bit를 이벤트에, 상위 16bit를 오류 코드에 사용하기 때문에 반드시 오류 코드를 먼저 확인하고 이벤트를 처리해야 한다. 이 경우 기존의 WSAGetLastError로는 오류 코드를 알 수 없다. 

## **3) 예제**

**Socket Info 관리**

```c++
// Socket Info
struct SOCKETINFO
{
	SOCKET sock;
	char buf[BUFSIZE + 1];
	int recvbytes;
	int sendbytes;
	BOOL recvdelayed;
	SOCKETINFO* next;
};

SOCKETINFO* SocketInfoList;


BOOL AddSocketInfo(SOCKET sock)
{
	SOCKETINFO* ptr = new SOCKETINFO;
	if (ptr == NULL)
	{
		wprintf(L"[Error] Can not Alloc Memory.\n");
		return FALSE;
	}

	ptr->sock = sock;
	ptr->recvbytes = 0;
	ptr->sendbytes = 0;
	ptr->recvdelayed = FALSE;
	ptr->next = SocketInfoList;
	SocketInfoList = ptr;

	return TRUE;
}

SOCKETINFO* GetSocketInfo(SOCKET sock)
{
	SOCKETINFO* ptr = SocketInfoList;

	while (ptr)
	{
		if (ptr->sock == sock)
			return ptr;
		ptr = ptr->next;
	}
	return NULL;
}

void RemoveSocketInfo(SOCKET sock)
{
	SOCKADDR_IN clientaddr;
	WCHAR szClientIP[16] = { 0 };
	int addrlen = sizeof(clientaddr);
	getpeername(sock, (SOCKADDR*)&clientaddr, &addrlen);
	InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
	wprintf(L"[TCP Server] Disconnect Client %s:%d!\n",
		szClientIP, ntohs(clientaddr.sin_port));

	SOCKETINFO* cur = SocketInfoList;
	SOCKETINFO* prev = NULL;

	while (cur)
	{
		if (cur->sock == sock)
		{
			if (prev)
				prev->next = cur->next;
			else
				SocketInfoList = cur->next;
		
			closesocket(cur->sock);
			delete cur;
			return;
		}
		prev = cur;
		cur = cur->next;
	}
}
```

**Window Message 처리**

```c++
LRESULT CALLBACK WndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
	switch (uMsg)
	{
	case WM_SOCKET:
		ProcessSocketMessage(hWnd, uMsg, wParam, lParam);
		return 0;
	case WM_DESTROY:
		PostQuitMessage(0);
		return 0;
	}
	return DefWindowProc(hWnd, uMsg, wParam, lParam);
}

void ProcessSocketMessage(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
	int ret;
	int addrlen;
	SOCKETINFO* ptr;
	SOCKET client_sock;
	SOCKADDR_IN clientaddr;

	if (WSAGETSELECTERROR(lParam))
	{
		_HandleError;
		RemoveSocketInfo(wParam);
		return;
	}

	switch (WSAGETSELECTEVENT(lParam))
	{
	case FD_ACCEPT:
		addrlen = sizeof(clientaddr);
		client_sock = accept(wParam, (SOCKADDR*)&clientaddr, &addrlen);
		if (client_sock == INVALID_SOCKET)
		{
			_HandleError;
			return;
		}
		WCHAR szClientIP[16] = { 0 };
		InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
		wprintf(L"\n[TCP Server] Accept Client: IP=%s, port=%d\n",
			szClientIP, ntohs(clientaddr.sin_port));
		
		AddSocketInfo(client_sock);
		ret = WSAAsyncSelect(client_sock, hWnd, WM_SOCKET, FD_READ | FD_WRITE | FD_CLOSE);
		if (ret == SOCKET_ERROR)
		{
			_HandleError;
			RemoveSocketInfo(client_sock);
		}
		break;

	case FD_READ:
		ptr = GetSocketInfo(wParam);
		if (ptr->recvbytes > 0)
		{
			ptr->recvdelayed = TRUE;
			return;
		}

		ret = recv(ptr->sock, ptr->buf, BUFSIZE, 0);
		if (ret == SOCKET_ERROR)
		{
			_HandleError;
			RemoveSocketInfo(wParam);
			return;
		}

		// Print Received Data
		ptr->recvbytes = ret;
		ptr->buf[ret] = '\0';

		addrlen = sizeof(clientaddr);
		getpeername(wParam, (SOCKADDR*)&clientaddr, &addrlen);
		wprintf(L"[TCP/%s:%d]", szClientIP, ntohs(clientaddr.sin_port));
		printf(" %s\n", ptr->buf);
		
		// echo server이므로 read 처리 이후 
        // 바로 write 처리할 것이라 break 생략

	case FD_WRITE:
		ptr = GetSocketInfo(wParam);
		if (ptr->recvbytes <= ptr->sendbytes)
			return;

		ret = send(ptr->sock, ptr->buf + ptr->sendbytes,
			ptr->recvbytes - ptr->sendbytes, 0);
		if (ret == SOCKET_ERROR)
		{
			_HandleError;
			RemoveSocketInfo(wParam);
			return;
		}

		ptr->sendbytes += ret;
		if (ptr->recvbytes == ptr->sendbytes)
		{
			ptr->recvbytes = ptr->sendbytes = 0;
			if (ptr->recvdelayed)
			{
				ptr->recvdelayed = FALSE;
				PostMessage(hWnd, WM_SOCKET, wParam, FD_READ);
			}
		}
		break;
	case FD_CLOSE:
		RemoveSocketInfo(wParam);
		break;
	}
}
```

**Main**
```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <stdio.h>

#define SERVERPORT 9000
#define BUFSIZE 512
#define WM_SOCKET (WM_USER + 1)

void HandleError(int line)
{
	int err = ::WSAGetLastError();
	wprintf(L"Error! Line %d: %d", line, err);
}
#define _HandleError HandleError(__LINE__)

int main(int argc, char* argv[])
{
	int ret;
	int err;

	// Register Window Class
	WNDCLASS wndclass;
	wndclass.style = CS_HREDRAW | CS_VREDRAW;
	wndclass.lpfnWndProc = WndProc;
	wndclass.cbClsExtra = 0;
	wndclass.cbWndExtra = 0;
	wndclass.hInstance = NULL;
	wndclass.hIcon = LoadIcon(NULL, IDI_APPLICATION);
	wndclass.hCursor = LoadCursor(NULL, IDC_ARROW);
	wndclass.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1);
	wndclass.lpszMenuName = NULL;
	wndclass.lpszClassName = L"MyWndClass";
	if (!RegisterClass(&wndclass)) return 1;
	
	HWND hWnd = CreateWindowW(L"MyWndClass", L"TCP Server",
		WS_OVERLAPPEDWINDOW, 0, 0, 600, 200, NULL, NULL, NULL, NULL);
	if (hWnd == NULL) return 1;
	ShowWindow(hWnd, SW_SHOWNORMAL);
	UpdateWindow(hWnd);

	// Initialize Winsock
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create Socket
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET)
	{
		_HandleError;
		return 0;
	}

	// Bind Socket
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, L"0.0.0.0", &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);
	ret = bind(listen_sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Socket to Listen
	ret = listen(listen_sock, SOMAXCONN);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// WSAAsyncSelect()
	ret = WSAAsyncSelect(listen_sock, hWnd, WM_SOCKET, FD_ACCEPT | FD_CLOSE);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Loop For Window Message
	MSG msg;
	while (GetMessage(&msg, 0, 0, 0) > 0)
	{
		TranslateMessage(&msg);
		DispatchMessage(&msg);
	}

	WSACleanup();
	return 0;
}
```
Select 모델 예제처럼 단일 스레드, 논블로킹 소켓을 사용하면서도 낮은 CPU 사용률을 확인할 수 있다. Select 모델은 셋 당 64개의 소켓만 처리할 수 있었던 것과 달리 연결리스트를 이용해 개수 제한 없이 소켓을 관리하는 것을 볼 수 있다. WSAAsyncSelect도 이벤트를 알려줄 뿐 소켓 정보를 관리해주지는 않으므로 이 부분은 사용자가 구현해야 한다.

recvdelayed는 FD_READ에 recv()를 하지 않은 경우를 위한 멤버이다. 이전에 받았지만 아직 보내지 않은 데이터가 있으면 recv()를 호출하지 않고 바로 return 해야 한다. 이때 recvdelayed 바탕으로 recv() 호출 여부를 확인할 수 있다. 만약 호출하지 않았다면 PostMessage로 FD_READ를 발생시킴으로써 이전에 도착했지만 처리하지 못한 데이터들을 얻을 수 있다. 

<br/>

# **3. WSAEventSelect 모델**

## **1) 동작 원리**

WSAEventSelect()를 사용하면 이벤트 객체를 소켓 당 하나씩 생성하여 네트워크 이벤트 발생을 감지할 수 있으며, 이를 통해 단일 스레드로도 여러 소켓을 처리할 수 있다.

## **2) 구성 요소**

WSAEventSelect 모델에서 사용되는 함수는 다음과 같다. 이벤트 객체 상태를 관찰하는 것만으로는 이벤트 종류 및 오류 상황에 대해 알 수 없기 때문에 상황에 맞게 이러한 함수들을 사용해야 한다.

|기능|함수|
|---|----|
|이벤트 객체 생성|WSACreateEvent()|
|이벤트 객체 제거|WSACloseEvent()|
|소켓과 이벤트 객체 연결|WSAEventSelect()|
|이벤트 객체 신호 상태 감지|WSAWaitForMultipleEvents()|
|네트워크 이벤트 종류 파악|WSAEnumNetworkEvents()|

이벤트 객체는 비신호 상태의 수동 리셋 이벤트로 생성된다. 만약 상태를 직접 바꾸고 싶다면 WSASetEvent(), WSAResetEvent()를 사용한다. 사용을 마친 후에는 WSACloseEvent()로 이벤트 객체를 제거해야 한다. 

WSAEventSelect()는 여러 방면에서 WSAAsyncSelect()와 유사하다. 우선 소켓, 이벤트 객체 핸들, 이벤트 종류를 전달하며 마지막 인자도 동일하게 FD 플래그로 설정한다. 또, 이 함수 호출 시 해당 소켓은 자동으로 논블로킹 모드가 되며 드물게 WouldBlock 오류가 생길 수 있다. 

또, accept() 함수가 반환하는 소켓은 연결 대기 소켓과 동일한 속성을 가지므로 반환 이후 WSAEventSelect()를 재호출하여 FD_READ, FD_WRITE로 등록해주어야 한다. 마지막으로 네트워크 이벤트 발생 시 대응 함수를 호출하지 않으면 같은 이벤트가 발생하지 않는 것 역시 동일하다. 

```c++
DWORD WSAWaitForMultipleEvents
(
    DWORD cEvent,
    const WSAEVENT *lphEvents,
    BOOL fWaitAll,
    DWORD dwTimeout,
    BOOL fAlertable
);
```

이는 이벤트 객체들을 동시에 관찰하는 함수이다. lphEvents는 이벤트 핸들 배열의 시작 주소를, cEvent는 원소 개수를 전달한다. fWaitAll이 TRUE면 모든 객체가 신호 상태일 때, FALSE면 하나 이상이 신호 상태일 때 반환한다. fAlertable은 IO 완료 루틴에서 쓰이며 이 모델에서는 FALSE를 전달한다. 

WSAEnumNetworkEvents()는 구체적으로 어떤 네트워크 이벤트가 발생했는지 마지막 인자인 WSANETWORKEVENTS 구조체를 통해 알려준다. 이는 FD 플래그가 조합된 네트워크 이벤트 정보와 배열로 이루어진 오류 코드를 포함한다. FD_READ_BIT, FD_WRITE_BIT 등을 인덱스 삼아 오류 정보를 참조할 수 있다. 

## **3) 예제**

```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <stdio.h>

#define SERVERPORT 9000
#define BUFSIZE 512

void HandleError(int line)
{
	int err = ::WSAGetLastError();
	wprintf(L"Error! Line %d: %d", line, err);
}
#define _HandleError HandleError(__LINE__)

// Socket Info
struct SOCKETINFO
{
	SOCKET sock;
	char buf[BUFSIZE + 1];
	int recvbytes;
	int sendbytes;
};

int sockCnt = 0;
SOCKETINFO* SocketInfoArray[WSA_MAXIMUM_WAIT_EVENTS];
WSAEVENT EventArray[WSA_MAXIMUM_WAIT_EVENTS];

BOOL AddSocketInfo(SOCKET sock)
{
	SOCKETINFO* ptr = new SOCKETINFO;
	if (ptr == NULL)
	{
		wprintf(L"[Error] Can not Alloc Memory.\n");
		return FALSE;
	}

	WSAEVENT hEvent = WSACreateEvent();
	if (hEvent == WSA_INVALID_EVENT)
	{
		_HandleError;
		return FALSE;
	}

	ptr->sock = sock;
	ptr->recvbytes = 0;
	ptr->sendbytes = 0;
	SocketInfoArray[sockCnt] = ptr;
	EventArray[sockCnt] = hEvent;
	sockCnt++;

	return TRUE;
}

void RemoveSocketInfo(int idx)
{
	SOCKETINFO* ptr = SocketInfoArray[idx];

	SOCKADDR_IN clientaddr;
	WCHAR szClientIP[16] = { 0 };
	int addrlen = sizeof(clientaddr);
	getpeername(ptr->sock, (SOCKADDR*)&clientaddr, &addrlen);
	InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
	wprintf(L"[TCP Server] Disconnect Client %s:%d!\n",
		szClientIP, ntohs(clientaddr.sin_port));

	closesocket(ptr->sock);
	delete ptr;
	WSACloseEvent(EventArray[idx]);

	if (idx != (sockCnt - 1))
	{
		SocketInfoArray[idx] = SocketInfoArray[sockCnt - 1];
		EventArray[idx] = EventArray[sockCnt - 1];
	}
		
	sockCnt--;
}

int main(int argc, char* argv[])
{
	int ret;
	int err;

	// Initialize Winsock
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create Socket
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET)
	{
		_HandleError;
		return 0;
	}

	// Bind Socket
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, L"0.0.0.0", &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);
	ret = bind(listen_sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Socket to Listen
	ret = listen(listen_sock, SOMAXCONN);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Network Event
	AddSocketInfo(listen_sock);
	ret = WSAEventSelect(listen_sock, EventArray[sockCnt - 1], FD_ACCEPT | FD_CLOSE);
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	int i, addrlen;
	SOCKET client_sock;
	SOCKADDR_IN clientaddr;
	WSANETWORKEVENTS NetworkEvents;
	
	while (1)
	{
		// Check Event
		i = WSAWaitForMultipleEvents(sockCnt, EventArray, FALSE, WSA_INFINITE, FALSE);
		if (i == WSA_WAIT_FAILED) continue;
		i -= WSA_WAIT_EVENT_0;

		// Get Network Event Info
		ret = WSAEnumNetworkEvents(SocketInfoArray[i]->sock,
			EventArray[i], &NetworkEvents);
		if (ret == SOCKET_ERROR) continue;

		// Accept Client Socket
		WCHAR szClientIP[16] = { 0 };
		if (NetworkEvents.lNetworkEvents & FD_ACCEPT)
		{
			if (NetworkEvents.iErrorCode[FD_ACCEPT_BIT] != 0)
			{
				_HandleError;
				continue;
			}

			addrlen = sizeof(clientaddr);
			client_sock = accept(SocketInfoArray[i]->sock, (SOCKADDR*)&clientaddr, &addrlen);
			if (client_sock == INVALID_SOCKET)
			{
				_HandleError;
				continue;
			}

			InetNtop(AF_INET, &clientaddr.sin_addr, szClientIP, 16);
			wprintf(L"\n[TCP Server] Accept Client: IP=%s, port=%d\n",
				szClientIP, ntohs(clientaddr.sin_port));

			if (sockCnt >= WSA_MAXIMUM_WAIT_EVENTS)
			{
				wprintf(L"[Error] Event List is full!\n");
				closesocket(client_sock);
				continue;
			}

			AddSocketInfo(client_sock);
			ret = WSAEventSelect(client_sock, EventArray[sockCnt - 1],
				FD_READ | FD_WRITE | FD_CLOSE);
			if (ret == SOCKET_ERROR)
			{
				_HandleError;
				return 0;
			}
		}

		// Receive & Send Data from Client
		if (NetworkEvents.lNetworkEvents & FD_READ ||
			NetworkEvents.lNetworkEvents & FD_WRITE)
		{
			if (NetworkEvents.lNetworkEvents & FD_READ &&
				NetworkEvents.iErrorCode[FD_READ_BIT] != 0)
			{
				_HandleError;
				continue;
			}

			if (NetworkEvents.lNetworkEvents & FD_WRITE &&
				NetworkEvents.iErrorCode[FD_WRITE_BIT] != 0)
			{
				_HandleError;
				continue;
			}

			SOCKETINFO* ptr = SocketInfoArray[i];
			if (ptr->recvbytes == 0)
			{

				ret = recv(ptr->sock, ptr->buf, BUFSIZE, 0);
				if (ret == SOCKET_ERROR)
				{
					if (WSAGetLastError() != WSAEWOULDBLOCK)
					{
						_HandleError;
						RemoveSocketInfo(i);
					}
					continue;
				}

				// Print Received Data
				ptr->recvbytes = ret;
				ptr->buf[ret] = '\0';

				addrlen = sizeof(clientaddr);
				getpeername(ptr->sock, (SOCKADDR*)&clientaddr, &addrlen);
				wprintf(L"[TCP/%s:%d]", szClientIP, ntohs(clientaddr.sin_port));
				printf(" %s\n", ptr->buf);
			}

			if (ptr->recvbytes > ptr->sendbytes)
			{
				// Send Data to Client
				ret = send(ptr->sock, ptr->buf + ptr->sendbytes,
					ptr->recvbytes - ptr->sendbytes, 0);
				if (ret == SOCKET_ERROR)
				{
					if (WSAGetLastError() != WSAEWOULDBLOCK)
					{
						_HandleError;
						RemoveSocketInfo(i);
					}
					continue;
				}
				ptr->sendbytes += ret;
				if (ptr->recvbytes == ptr->sendbytes)
					ptr->recvbytes = ptr->sendbytes = 0;
			}
		}

		if (NetworkEvents.lNetworkEvents & FD_CLOSE)
		{
			if (NetworkEvents.iErrorCode[FD_CLOSE_BIT] != 0)
				_HandleError;
			RemoveSocketInfo(i);
		}
	}

	WSACleanup();
	return 0;
}
```

<br/>

# **출처**

TCP/IP 윈도우 소켓 프로그래밍, 저자 김선우, 한빛미디어