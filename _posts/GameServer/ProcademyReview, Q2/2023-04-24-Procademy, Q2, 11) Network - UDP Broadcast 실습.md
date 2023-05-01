---
title: Procademy, Q2, 11) Network - UDP Broadcast 실습
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. UDP Broadcast 실습**

프로토콜: 0xff 0xee 0xdd 0xaa 0x00 0x99 0x77 0x55 0x33 0x11 ( 10 Byte )

특정 포트에 대해 위 메시지를 브로드캐스팅으로 뿌린다. 해당 포트에 서버가 열려있다면 응답 (게임 방 이름 / UTF-16) 문자열이 되돌아 오며, 로컬 네트워크 내부에 해당 포트의 서버가 없다면 응답이 오지 않는다. 

조건 1. 포트범위 10001 ~ 10099 까지를 모두 검색해야 한다.

조건 2. 데이터 응답까지는 200 밀리세컨즈를 대기한다.

조건 3. Sleep 사용금지

조건 5. 서버의 방 이름, IP, Port 번호를 출력 한다.

조건 6. 방은 총 10개 열려있으며 10개를 모두 찾아내면 성공

```c++
#include <stdio.h>
#include <iostream>
#include <winsock2.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")

#define SERVERIP L"255.255.255.255"
#define MSGSIZE 10
#define BUFSIZE 64

#define _HandleError HandleError(__LINE__)
void HandleError(int line)
{
	int err = ::WSAGetLastError();
	wprintf(L"Error! line %d, %d\n", line, err);
}

int main()
{
	// Initialize WSA
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// Create Socket
	SOCKET sock = socket(AF_INET, SOCK_DGRAM, 0);
	if (sock == INVALID_SOCKET)
	{
		_HandleError;
		return 0;
	}

	// Set Broadcast Option
	int ret;
	BOOL bEnable = TRUE;
	ret = setsockopt(sock, SOL_SOCKET, SO_BROADCAST, (char*)&bEnable, sizeof(bEnable));
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	int time = 200;
	ret = setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, (char*)&time, sizeof(time));
	if (ret == SOCKET_ERROR)
	{
		_HandleError;
		return 0;
	}

	// Set Address
	SOCKADDR_IN serveraddr;
	ZeroMemory(&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	InetPton(AF_INET, SERVERIP, &serveraddr.sin_addr);

	char msg[MSGSIZE]
		= { 0xff, 0xee, 0xdd, 0xaa, 0x00, 0x99, 0x77, 0x55, 0x33, 0x11 };

	for (int i = 10001; i < 20000; i++)
	{
		// Reset Port
		serveraddr.sin_port = htons(i);

		SOCKADDR_IN peeraddr;
		int addrlen = sizeof(peeraddr);

		// Send Data
		ret = sendto(sock, msg, MSGSIZE, 0, (SOCKADDR*)&serveraddr, sizeof(serveraddr));
		if (ret == SOCKET_ERROR)
		{
			_HandleError;
			return 0;
		}

		char buf[BUFSIZE];

		// Recieve Data
		ret = recvfrom(sock, buf, BUFSIZE, 0, (SOCKADDR*)&peeraddr, &addrlen);
		if (ret == -1)
		{
			continue;
		}
		else if (ret == SOCKET_ERROR)
		{
			_HandleError;
			return 0;
		}
		
		WCHAR szClientIP[16] = { 0 };
		InetNtop(AF_INET, &peeraddr.sin_addr, szClientIP, 16);
		
		WCHAR* roomName = (WCHAR*)buf;
		roomName[ret / 2] = L'\0';
		wprintf(L"[UDP/%s:%d] %s\n", szClientIP, ntohs(peeraddr.sin_port), roomName);
	}

	// Close Socket
	closesocket(sock);
	WSACleanup();
	return 0;
}
```

UTF-16으로 데이터가 오기 때문에 WCHAR로 변환하여 null terminate를 붙이고 출력해야 올바른 결과가 나온다. 