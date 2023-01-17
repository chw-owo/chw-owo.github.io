---
title: Server Intro
categories: WindowsIOCPServer
tags: 
toc: true
toc_sticky: true
---
## **1. 서버란 무엇일까?**

Server, 제공하다라는 뜻의 Serve에 -er이 붙은 단어이다. 클라이언트가 요청을 보내면 해당 요청에 맞는 서비스를 제공하는 역할을 한다. 일반적으로 다른 컴퓨터에서 언제든 연결이 가능하도록 대기 상태로 상시 실행 중이다. 서버가 어떤 역할을 할 것인지는 서비스의 성격에 따라 정해지게 된다. 이 카테고리에서는 Windows IOCP Server를 이용한 실시간 게임 서버 구현을 목표로 할 것이다. 

<br/>

## **2. 웹서버**

HTTP Server가 대표적인 웹서버이다. 정보의 요청/갱신/응답이 게임 서버에 비해 드물게 발생하며 실시간 Interaction이 필요하지 않다. 따라서  Server 측에서 Client에게 먼저 접근하는 일이 없다. 일반적으로 요청/응답이 끝나면 연락이 끊어져서 client의 상태를 잊고 지내기 때문에, Stateless Server라고도 부른다. 보통 웹서버를 구현할 때는 Spring, nodejs 등 프레임워크를 이용하여 구현한다. 

<br/>

## **3. 게임 서버**

TCP Server, Binary Server, Stateful Server 등이 이에 해당된다. 러프하게 웹서버/게임서버로 나누어두긴 했지만 서비스 종류에 따라서 Stateless Server를 쓰는 게임 서버도 있다. 게임 서버의 정보의 요청/갱신/응답이 자주 발생가며 실시간 Interaction이 필요하다. Server가 언제든 client에게 접근 가능해야 하며, client와 연결되어 있는 동안 client의 상태에 대해 알고 있어야 한다. 따라서 이렇게 상태를 주시하는 실시간 서버를 두고 Stateful Server라고 부른다. 

게임 서버를 설계할 때는 최대 동시 접속자, 게임 장르, 채널링, 게임 로직, 네트워크, DB, 스레드 개수, 스레드 모델, 네트워크 모델, 반응성, 데이터 베이스 등을 고려하여 설계한다. 게임 서버의 경우 게임의 특성에 따라서 구성이 많이 달라지기 때문에 웹서버처럼 최적의 프레임워크라는 것이 존재하기 어렵다.

<br/>

## **4. 자료형**

```c++
using BYTE = unsigned char;
using int8 = __int8;
using int16 = __int16;
using int32 = __int32;
using int64 = __int64;
using uint8 = unsigned __int8;
using uint16 = unsigned __int16;
using uint32 = unsigned __int32;
using uint64 = unsigned __int64;
```

운영체제, 데이터 모델에 따라 자료형의 크기가 달라지기 때문에, 이로 인한 버그를 예방하기 위해 자료형을 따로 지정해서 사용하는 경우가 많다. WindowsIOCPServer 포스트에서도 위와 같은 자료형을 기반으로 사용하려고 한다. 

<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
