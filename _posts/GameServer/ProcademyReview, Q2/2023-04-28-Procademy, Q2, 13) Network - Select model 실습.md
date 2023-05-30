---
title: Procademy, Q2, 13) Network - Select model 실습
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 실습 설명**

패킷 프로토콜 (16Byte 고정 패킷)

```
ID할당(0)		Type(4Byte) | ID(4Byte) | 안씀(4Byte)  | 안씀(4Byte)
별생성(1)		Type(4Byte) | ID(4Byte) | X(4Byte)     | Y(4Byte)
별삭제(2)		Type(4Byte) | ID(4Byte) | 안씀(4Byte)  | 안씀(4Byte)
이동(3)		    Type(4Byte) | ID(4Byte) | X(4Byte)     | Y(4Byte)
```
0~2번은 서버 -> 클라, 3번은 클라 <-> 서버 쌍방 패킷이다. 실제로는 클라 -> 서버, 서버 -> 클라를 분리하는 것이 일반적이지만 이 실습은 위와 같이 간단하게 사용한다. 클라이언트가 서버에 최초 접속시 ID할당(0) 패킷 수신, 별생성(1) 패킷 수신 (자신의 데이터), 별생성(1) 패킷 수신 (타 유저의 데이터)를 순차적으로 실행한다. 자기 자신이 이동할 때는 자기 좌표를 먼저 바꾸어주고 서버에 패킷을 보내며, 자신에게는 이동 메시지가 돌아오지 않는다. 서버를 경유한 사이 좌표가 바뀔 경우 튕기는 것처럼 보일 수도 있기 때문이다. 

위 예제는 모든 메시지가 고정된 크기를 갖지만 실제로는 헤더에 패킷 길이 데이터를 포함해야 한다. 가변 길이 메시지를 쓸 때 필요하기도 하고, 실제 작업을 할 땐 네트워크/콘텐츠 영역을 분리하는데 메시지의 크기와 Type을 결정하는 것은 네트워크 라이브러리의 영역이어야 하는 반면 Type은 콘텐츠 영역에서 사용하게 되기 때문에 Type은 길이와 분리하는 게 논리적으로 적절하다.

또, 원래는 패킷 조각화 및 송신 버퍼 부족을 대비해 송수신 버퍼를 Ringbuffer로 구현하는 게 일반적이다. 그러나 지금은 작은 패킷만 보낼 것이므로 하지 않는다. 같은 이유로 select 실습이긴 하지만 send는 select를 거치지 않고 전송이 필요할 때 바로 보내도록 한다. 따라서 write set 없이 read set만 사용할 것이다. 또 connect를 Non-block으로 만들면 WouldBlock을 체크해야 해서 번거롭기 때문에, Block 소켓으로 connect 하고 연결이 된 이후에 Non-block으로 바꾼다. 

select의 시간 값을 NULL로 하면 내가 움직이지 않을 땐 다른 유저의 데이터도 받을 수 없게 된다. 따라서  프레임에 맞게 적절히 사용해야 한다. 만약 프레임을 정교하게 맞추고 싶다면 0으로 설정하고 프레임 맞추는 코드를 따로 넣는 것이 낫다. 또, 이 실습은 메시지가 16byte로 규격화 되어 있으므로 16byte 씩 recv 해도 되고, 한번에 크게 받아서 나누어도 된다. recv 자체가 시간 소요가 있는 작업이므로 큰 단위로 받아서 작업하는 것이 더 효율적이다. 이렇게 직렬로 붙어있는 메시지를 크기에 맞게 분리하는 것을 두고 마샬링이라고 한다. 

<br/>

# **2. 실습 코드**

접속자를 관리하는 List<Player*>에는 직접 구현한 List를 사용했다. 

## **1) Client**

**Message.h**
```c++
#pragma once

enum MSG_TYPE
{
	TYPE_ID = 0,
	TYPE_CREATE,
	TYPE_DELETE,
	TYPE_MOVE
};

struct MSG_ID
{
	MSG_TYPE type;
	__int32 ID;
	__int32 none1;
	__int32 none2;
};

struct MSG_CREATE
{
	MSG_TYPE type;
	__int32 ID;
	__int32 X;
	__int32 Y;
};

struct MSG_DELETE
{
	MSG_TYPE type;
	__int32 ID;
	__int32 none1;
	__int32 none2;
};

struct MSG_MOVE
{
	MSG_TYPE type;
	__int32 ID;
	__int32 X;
	__int32 Y;
};
```

**Player.h**
```c++
#pragma once
#include "List.h"

#define XMAX 81
#define YMAX 24

struct Player
{
	__int32 ID;
	__int32 X;
	__int32 Y;
};

extern __int32 ID;
extern Player* gMyPlayer;
extern CList<Player*> gPlayerList;

```

**Player.cpp**
```c++
#include "Player.h"
#include "List.h"

__int32 ID = -1;
Player* gMyPlayer = nullptr;
CList<Player*> gPlayerList;
```

**Main.cpp**
```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <iostream>
#include "Message.h"
#include "List.h"
#include "Player.h"

#define SERVERPORT 3000
#define MSGSIZE 16
#define IPMAX 16

SOCKET sock;
HANDLE hConsole;
int main(int argc, char* argv[])
{
	// Initialize Winsock
	int err;
	COORD stCoord;
	stCoord.X = 0;
	stCoord.Y = YMAX;
	SetConsoleCursorPosition(hConsole, stCoord);

	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create Socket
	sock = socket(AF_INET, SOCK_STREAM, 0);
	if (sock == INVALID_SOCKET)
	{
		err = ::WSAGetLastError();
		if (err != WSAEWOULDBLOCK)
		{
			printf("Error! Line %d: %d", __LINE__, err);
			return 0;
		}
	}

	// Get IP Address to connect
	WCHAR IPbuf[IPMAX];
	wprintf(L"Enter IP Address to connect: ");
	wscanf_s(L"%s", IPbuf, IPMAX);
	wprintf(L"\n");

	// Connect Socket to Server
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, IPbuf, &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);

	int ret;
	ret = connect(sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (ret == SOCKET_ERROR)
	{
		err = ::WSAGetLastError();
		if (err != WSAEWOULDBLOCK)
		{
			printf("Error! Line %d: %d", __LINE__, err);
			return 0;
		}
	}

	// Set Socket Non-Blocking Mode
	u_long on = 1;
	ret = ioctlsocket(sock, FIONBIO, &on);
	if (ret == SOCKET_ERROR)
	{
		err = ::WSAGetLastError();
		if (err != WSAEWOULDBLOCK)
		{
			printf("Error! Line %d: %d", __LINE__, err);
			return 0;
		}
	}

	// Setting For Players
	CList<Player*>::iterator iter;

	// Setting For Network
	char recvBuf[MSGSIZE] = { 0 };
	FD_SET rset;

	// Setting For Keyboard IO
	__int32 prevX = 0;
	__int32 prevY = 0;
	clock_t timecheck = clock();

	// Setting For Render
	CONSOLE_CURSOR_INFO stConsoleCursor;
	stConsoleCursor.bVisible = FALSE;
	stConsoleCursor.dwSize = 1;
	hConsole = GetStdHandle(STD_OUTPUT_HANDLE);
	SetConsoleCursorInfo(hConsole, &stConsoleCursor);
	char screenBuffer[YMAX][XMAX] = { ' ', };

	while (1)
	{
		// Network =================================
        #pragma region Network

		FD_ZERO(&rset);
		FD_SET(sock, &rset);

		// Select 
		TIMEVAL time;
		time.tv_usec = 0;
		ret = select(0, &rset, NULL, NULL, &time);
		if (ret == SOCKET_ERROR)
		{
			err = ::WSAGetLastError();
			if (err != WSAEWOULDBLOCK)
			{
				printf("Error! Line %d: %d", __LINE__, err);
				break;
			}
		}
		else if (ret > 0)
		{
			// Send Data to Server	
			if (gMyPlayer != nullptr &&
				(gMyPlayer->X != prevX || gMyPlayer->Y != prevY))
			{
				MSG_MOVE Msg;
				Msg.type = TYPE_MOVE;
				Msg.ID = gMyPlayer->ID;
				Msg.X = gMyPlayer->X;
				Msg.Y = gMyPlayer->Y;

				ret = send(sock, (char*)&Msg, MSGSIZE, 0);
				if (ret == SOCKET_ERROR)
				{
					err = ::WSAGetLastError();
					if (err != WSAEWOULDBLOCK)
					{
						printf("Error! Line %d: %d", __LINE__, err);
						break;
					}
				}

				prevX = gMyPlayer->X;
				prevY = gMyPlayer->Y;
			}
			

			// Reveice Data from Server
			if (FD_ISSET(sock, &rset))
			{
				ret = recv(sock, recvBuf, MSGSIZE, 0);
				if (ret == SOCKET_ERROR)
				{
					err = ::WSAGetLastError();
					if (err != WSAEWOULDBLOCK)
					{
						printf("Error! Line %d: %d", __LINE__, err);
						break;
					}
				}
				else if (ret == 0)
				{
					break;
				}

				MSG_TYPE* pType = (MSG_TYPE*)recvBuf;
				switch (*pType)
				{
				case TYPE_ID:
				{
					MSG_ID* pMsg = (MSG_ID*)recvBuf;
					ID = pMsg->ID;
					break;
				}

				case TYPE_CREATE:
				{
					MSG_CREATE* pMsg = (MSG_CREATE*)recvBuf;
					if (ID == pMsg->ID && gMyPlayer == nullptr)
					{
						gMyPlayer = new Player;
						gMyPlayer->ID = pMsg->ID;
						gMyPlayer->X = pMsg->X;
						gMyPlayer->Y = pMsg->Y;
					}
					else if (ID != pMsg->ID)
					{
						Player* player = new Player;
						player->ID = pMsg->ID;
						player->X = pMsg->X;
						player->Y = pMsg->Y;
						gPlayerList.push_back(player);
					}
					break;
				}

				case TYPE_DELETE:
				{
					MSG_DELETE* pMsg = (MSG_DELETE*)recvBuf;
					if (gMyPlayer != nullptr && gMyPlayer->ID == pMsg->ID)
					{
						delete gMyPlayer;
						gMyPlayer = nullptr;
					}
					else if (gMyPlayer == nullptr || gMyPlayer->ID != pMsg->ID)
					{
						for (iter = gPlayerList.begin(); iter != gPlayerList.end(); )
						{
							if ((*iter)->ID == pMsg->ID)
							{
								delete (*iter);
								iter = gPlayerList.erase(iter);
								break;
							}
							else
							{
								++iter;
							}
						}
					}
					break;
				}

				case TYPE_MOVE:
				{
					MSG_MOVE* pMsg = (MSG_MOVE*)recvBuf;

					for (iter = gPlayerList.begin(); iter != gPlayerList.end(); ++iter)
					{
						if ((*iter)->ID == pMsg->ID)
						{
							(*iter)->X = pMsg->X;
							(*iter)->Y = pMsg->Y;
							break;
						}
					}
					break;
				}

				default:
					break;
				}
			}
		}

        #pragma endregion 

		// Keyboard IO ===================================
        #pragma region KeyboardIO

		if (gMyPlayer != nullptr && clock() - timecheck > 30)
		{
			if (GetAsyncKeyState(VK_LEFT) && gMyPlayer->X > 0)
				gMyPlayer->X--;
			if (GetAsyncKeyState(VK_RIGHT) && gMyPlayer->X < XMAX - 2)
				gMyPlayer->X++;
			if (GetAsyncKeyState(VK_UP) && gMyPlayer->Y > 0)
				gMyPlayer->Y--;
			if (GetAsyncKeyState(VK_DOWN) && gMyPlayer->Y < YMAX - 2)
				gMyPlayer->Y++;

			timecheck = clock();
		}

        #pragma endregion

		// Render ==================================
        #pragma region Render

        // Buffer Clear
		memset(screenBuffer, ' ', YMAX * XMAX);
		for (int i = 0; i < YMAX; i++)
		{
			screenBuffer[i][XMAX - 1] = '\0';
		}

		// Set Buffer
		if (gMyPlayer != nullptr)
			screenBuffer[gMyPlayer->Y][gMyPlayer->X] = '*';

		for (iter = gPlayerList.begin(); iter != gPlayerList.end(); ++iter)
			screenBuffer[(*iter)->Y][(*iter)->X] = '*';

		// Buffer Flip
		for (int i = 0; i < YMAX; i++)
		{
			stCoord.X = 0;
			stCoord.Y = i;
			SetConsoleCursorPosition(hConsole, stCoord);
			printf(screenBuffer[i]);
		}

	    #pragma endregion
	}

	closesocket(sock);
	WSACleanup();

	// Delete Players' Data
	if (gMyPlayer != nullptr)
		delete gMyPlayer; 
	for (iter = gPlayerList.begin(); iter != gPlayerList.end(); ++iter)
		delete (*iter);

	return 0;
}

```

## **2) Server**

**Main.cpp**
```c++
#pragma comment(lib, "ws2_32")
#include <ws2tcpip.h>
#include <iostream>
#include <stdarg.h>

#include "List.h"
#include "Message.h"
#include "Player.h"
#include "Network.h"

int main(int argc, char* argv[])
{
	// Initialize
	int err;
	int bindRet;
	int ioctRet;
	int listenRet;
	int selectRet;

	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create Socket
	listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET)
	{
		err = ::WSAGetLastError();
		SetCursor();
		printf("Error! Line %d: %d\n", __LINE__, err);
		return 0;
	}

	// Bind
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, L"0.0.0.0", &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);

	bindRet = bind(listen_sock, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
	if (bindRet == SOCKET_ERROR)
	{
		err = ::WSAGetLastError();
		SetCursor();
		printf("Error! Line %d: %d\n", __LINE__, err);
		return 0;
	}

	// Listen
	listenRet = listen(listen_sock, SOMAXCONN);
	if (listenRet == SOCKET_ERROR)
	{
		err = ::WSAGetLastError();
		SetCursor();
		printf("Error! Line %d: %d\n", __LINE__, err);
		return 0;
	}

	// Set Non-Blocking Mode
	u_long on = 1;
	ioctRet = ioctlsocket(listen_sock, FIONBIO, &on);
	if (ioctRet == SOCKET_ERROR)
	{
		err = ::WSAGetLastError();
		SetCursor();
		printf("Error! Line %d: %d\n", __LINE__, err);
		return 0;
	}

	FD_SET rset;
	FD_SET wset;


	CONSOLE_CURSOR_INFO stConsoleCursor;
	stConsoleCursor.bVisible = FALSE;
	stConsoleCursor.dwSize = 1;
	hConsole = GetStdHandle(STD_OUTPUT_HANDLE);
	SetConsoleCursorInfo(hConsole, &stConsoleCursor);
	char screenBuffer[YMAX][XMAX] = { ' ', };

	printf("Setting Complete!\n");
	while (1)
	{
		// Network ============================================
		FD_ZERO(&rset);
		FD_ZERO(&wset);
		FD_SET(listen_sock, &rset);
		for (CList<Player*>::iterator i = gPlayerList.begin(); i != gPlayerList.end(); i++)
		{
			FD_SET((*i)->sock, &rset);
			if((*i)->sendBuf.GetUseSize() > 0)
				FD_SET((*i)->sock, &wset);
		}

		// Select 
		selectRet = select(0, &rset, &wset, NULL, NULL);
		if (selectRet == SOCKET_ERROR)
		{
			err = ::WSAGetLastError();
			SetCursor();
			printf("Error! Line %d: %d\n", __LINE__, err);
			break;
		}
		else if (selectRet > 0)
		{
			for (CList<Player*>::iterator i = gPlayerList.begin(); i != gPlayerList.end(); i++)
			{
				if (FD_ISSET((*i)->sock, &wset) && (*i)->alive)
					SendProc((*i));
			}

			if (FD_ISSET(listen_sock, &rset))
				AcceptProc();

			for (CList<Player*>::iterator i = gPlayerList.begin(); i != gPlayerList.end(); i++)
			{
				if (FD_ISSET((*i)->sock, &rset) && (*i)->alive)
				{
					RecvProc((*i));
					SetBuffer((*i));
				}
			}

			if (deleted = true)
				DeleteDeadPlayers();
		}

		// Render ==============================================

		// Buffer Clear
		memset(screenBuffer, ' ', YMAX * XMAX);
		for (int i = 0; i < YMAX; i++)
		{
			screenBuffer[i][XMAX - 1] = '\0';
		}

		// Set Buffer
		for (CList<Player*>::iterator i = gPlayerList.begin(); i != gPlayerList.end(); i++)
		{
			if ((*i)->alive)
			{
				if ((*i)->X > XMAX - 2) (*i)->X = XMAX - 2;
				else if ((*i)->X <= 0) (*i)->X = 0;
				if ((*i)->Y > YMAX - 1) (*i)->Y = YMAX - 1;
				else if ((*i)->Y <= 0) (*i)->Y = 0;

				screenBuffer[(*i)->Y][(*i)->X] = '*';
			}
		}
		sprintf_s(screenBuffer[0], "Connect Client: %02d\n", gPlayerList.size());

		// Buffer Flip
		for (int i = 0; i < YMAX; i++)
		{
			stCoord.X = 0;
			stCoord.Y = i;
			SetConsoleCursorPosition(hConsole, stCoord);
			printf(screenBuffer[i]);
		}
	}

	closesocket(listen_sock);
	WSACleanup();
	return 0;
}

```

**Network.cpp**
```c++
#include "Network.h"
#include "Message.h"
#include "RingBuffer.h"

COORD stCoord;
HANDLE hConsole;
SOCKET listen_sock;

inline void SetCursor()
{
	stCoord.X = 0;
	stCoord.Y = YMAX;
	SetConsoleCursorPosition(hConsole, stCoord);
}

void AcceptProc()
{	
	SOCKADDR_IN clientaddr;
	int addrlen = sizeof(clientaddr);
	Player* newPlayer = new Player;

	newPlayer->sock = accept(listen_sock, (SOCKADDR*)&clientaddr, &addrlen);
	if (newPlayer->sock == INVALID_SOCKET)
	{
		int err = ::WSAGetLastError();
		SetCursor();
		printf("Error! Line %d: %d\n", __LINE__, err);
	}

	newPlayer->alive = true;
	newPlayer->ID = gIDGenerator.AllocID();
	gPlayerList.push_back(newPlayer);

	// Send <Allocate ID Message> to New Player
	MSG_ID MsgID;
	MsgID.ID = newPlayer->ID;
	SendUnicast((char*)&MsgID, newPlayer);

	// Send <Create New Player Message> to All Player
	MSG_CREATE MsgCreateNew;
	MsgCreateNew.ID = newPlayer->ID;
	MsgCreateNew.X = newPlayer->X;
	MsgCreateNew.Y = newPlayer->Y;
	SendBroadcast((char*)&MsgCreateNew, newPlayer);

	// Send <Create All Players Message> to New Player
	MSG_CREATE MsgCreateAll;
	for (CList<Player*>::iterator i = gPlayerList.begin(); i != gPlayerList.end(); i++)
	{
		MsgCreateAll.ID = (*i)->ID;
		MsgCreateAll.X = (*i)->X;
		MsgCreateAll.Y = (*i)->Y;
		SendUnicast((char*)&MsgCreateAll, newPlayer);
	}
}

void RecvProc(Player* player)
{
	char recvBuf[MAX_BUF_SIZE] = { '\0', };
	int recvRet = recv(player->sock, recvBuf, MAX_BUF_SIZE, 0);
	if (recvRet == SOCKET_ERROR)
	{
		int err = WSAGetLastError();
		if (err != WSAEWOULDBLOCK)
		{
			SetCursor();
			printf("Error! Line %d: %d\n", __LINE__, err);
			Disconnect(player);
			return;
		}
	}
	else if (recvRet == 0)
	{
		Disconnect(player);
		return;
	}

	int enqueueSize = player->recvBuf.Enqueue(recvBuf, recvRet);
	if (enqueueSize != recvRet)
	{
		SetCursor();
		printf("Error! Line %d: recv buffer enqueue\n", __LINE__);
		Disconnect(player);
		return;
	}
	
}

void SetBuffer(Player* player)
{
	while (player->recvBuf.GetUseSize() >= MSGSIZE)
	{
		char msgBuf[MSGSIZE];
		int dequeueRet = player->recvBuf.Dequeue(msgBuf, MSGSIZE);
		if (dequeueRet != MSGSIZE)
		{
			SetCursor();
			printf("Error! Line %d: recv buffer dequeue\n", __LINE__);
			Disconnect(player);
			return;
		}

		MSG_TYPE* pType = (MSG_TYPE*)msgBuf;

		switch (*pType)
		{
		case TYPE_MOVE:
		{
			MSG_MOVE* pMsg = (MSG_MOVE*)msgBuf;
			if (player->ID == pMsg->ID)
			{
				player->X = pMsg->X;
				player->Y = pMsg->Y;
				SendBroadcast((char*)pMsg, player);
			}
			break;
		}
		default:
			break;
		}
	}
}

void SendProc(Player* player)
{
	int msgBufSize = player->sendBuf.GetUseSize();
	if (msgBufSize <= 0)
	{
		return;
	}
	else if (msgBufSize % MSGSIZE != 0)
	{
		int remains = msgBufSize % MSGSIZE;
		msgBufSize -= remains;
	}

	char msgBuf[MAX_BUF_SIZE];
	int dequeueRet = player->sendBuf.Dequeue(msgBuf, msgBufSize);
	if (dequeueRet != msgBufSize)
	{
		SetCursor();
		printf("Error! Line %d: recv buffer dequeue\n", __LINE__);
		Disconnect(player);
		return;
	}

	int sendRet = send(player->sock, msgBuf, msgBufSize, 0);
	if (sendRet == SOCKET_ERROR)
	{
		int err = WSAGetLastError();
		if (err != WSAEWOULDBLOCK)
		{
			SetCursor();
			printf("Error! Line %d: %d\n", __LINE__, err);
			Disconnect(player);
			return;
		}
	}
}

void SendUnicast(char* msg, Player* player)
{
	int enqueueRet = player->sendBuf.Enqueue(msg, MSGSIZE);
	if (enqueueRet != MSGSIZE)
	{
		SetCursor();
		printf("Error! Line %d: send buffer enqueue error\n", __LINE__);
		Disconnect(player);		
	}	
}

void SendBroadcast(char* msg, Player* expPlayer)
{
	int enqueueRet;
	if (expPlayer == nullptr)
	{
		for (CList<Player*>::iterator i = gPlayerList.begin(); i != gPlayerList.end(); i++)
		{
			if ((*i)->alive)
			{
				enqueueRet = (*i)->sendBuf.Enqueue(msg, MSGSIZE);
				if (enqueueRet != MSGSIZE)
				{
					SetCursor();
					printf("Error! Line %d: send buffer enqueue error\n", __LINE__);
					Disconnect(*i);
				}
			}
		}
	}
	else
	{
		for (CList<Player*>::iterator i = gPlayerList.begin(); i != gPlayerList.end(); i++)
		{
			if (expPlayer->ID != (*i)->ID && (*i)->alive)
			{
				enqueueRet = (*i)->sendBuf.Enqueue(msg, MSGSIZE);
				if (enqueueRet != MSGSIZE)
				{
					SetCursor();
					printf("Error! Line %d: send buffer enqueue error\n", __LINE__);
					Disconnect(*i);
				}
			}
		}
	}
	
}

void Disconnect(Player* player)
{
	MSG_DELETE MsgDelete;
	MsgDelete.ID = player->ID;
	player->alive = false;
	SendBroadcast((char*)&MsgDelete);
	deleted = true;
}

void DeleteDeadPlayers()
{
	for (CList<Player*>::iterator i = gPlayerList.begin(); i != gPlayerList.end();)
	{
		if (!(*i)->alive)
		{
			Player* player = *i;
			i = gPlayerList.erase(i);
			closesocket(player->sock);
			delete(player);
		}
		else
		{
			i++;
		}
	}
	deleted = false;
}

```