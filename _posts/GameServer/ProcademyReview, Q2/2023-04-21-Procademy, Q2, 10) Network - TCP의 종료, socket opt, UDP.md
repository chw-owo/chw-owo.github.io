---
title: Procademy, Q2, 10) Network - TCP의 종료, socket opt, UDP
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. TCP의 종료**

## **1) closesocket()**

```c++
ret = recv(...);
if(ret == 0)
{
    closesocket(...);
}
```
일반적인 TCP 설계에서는 recv 반환 값이 0이면 closesocket을 호출한다. closesocket은 소켓 리소스를 반환하며 TCP일 경우 연결 해제 절차, 즉 Fin 메시지 전송도 함께 수행한다. 이때 반환과 TCP 연결 해제는 별도의 작업이므로 소켓이 반환되었어도 연결은 유지되고 있을 수 있다. 이 두 작업의 진행 시점은 이후 설명할 socket option으로 설정한다. 

## **2) 4-Way Handshake**

위 예제에서 recv가 0을 반환하는 것은 Fin이 도착했다는 의미이며, closesocket을 호출하면 상대에게도 Fin 메시지를 전송한다. Fin에 대한 Ack는 TCP가 자동으로 전송한다. 이때 Fin1 - Ack1, Fin2 - Ack2 네 단계를 거쳐 연결이 해제되므로 종료 과정을 4-Way Handshake라 부른다. TCP는 종료에 대한 제어권을 사용자에게 주기 위해 이렇게 설계되었다. 

![image](https://user-images.githubusercontent.com/96677719/233758143-8d532b36-5a40-4ba4-ac56-76df19d0e60c.png)

3-Way와 달리 Ack1과 Fin2이 분리되었으며 이는 연속적으로 이루어지지 않는다. 종료 통지를 받은 측에서 closesocket을 호출해야만 Fin2이 보내진다. 이런 이유 때문에 통신 문제로 Fin1에 대한 Ack1가 오지 않으면 FIN_WAIT_1에, 종료 통지 수신 측에서 closesocket을 하지 않으면 FIN_WAIT_2에 머무르며 다음 단계로 진행되지 않는다. 

이는 Fin이 "연결을 종료하겠다"는 의미 대신 “나는 더 이상 보낼 게 없다. (그러나 너는 보낼 게 있으면 보내도 된다.)” 라는 의미이기 때문이다. 즉, 더 이상 recv() 할 게 없음을 통지하는 것이므로 이후에도 send()는 할 수 있다. closesocket은 FIN_WAIT_2 상태에서 30초가 지나면 OS가 연결을 해제하지만, shutdown 함수를 통해 Fin만 보낼 수도 있다. 

TCP는 정상 종료와 강제 종료를 구분하여, 정상 종료일 경우 이와 같은 4-Way 방식으로 종료를 처리한다. 반면 강제 종료일 경우 RST로 종료를 수행하는데 이 경우 FIN_WAIT 혹은 TIME_WAIT 등 없이 즉각 연결 해제 및 리소스 반환을 수행한다. 정상 종료 절차는 TCP에서 제공하는 것이므로 UDP와 같은 타 프로토콜에서는 항상 강제 종료 방식을 사용한다. 

더 이상 쓰지 않을 소켓이 FIN_WAIT에 머무르는 것은 Nonpaged pool을 낭비하는 것이므로 빨리 처리해야 한다. 만약 악의적으로 많은 소켓을 연결하고 closesocket을 하지 않는다면 30초만으로도 큰 문제가 생길 수 있다. 이때 강제 종료 메시지인 RST 조차도 Established일 때만 전송이 되기 때문에 상대가 한번 FIN_WAIT 상태가 되면 대응할 방법이 없다. 

## **3) TIME_WAIT**

TIME_WAIT은 "연결 해제는 되었지만 이 포트에 한동안 바인딩 못하게 하겠다, 그리고 상대도 이 포트를 못쓰게 하겠다"는 목적의 상태이다. IP 계층은 TCP처럼 순서 보장을 하지 않기 때문에 연결 종료 이후 port를 재사용했는데 뒤늦게 패킷이 도착할 경우 자신에게 온 패킷이 아님에도 오해가 생길 수 있다. 따라서 TIME_WAIT로 즉각적인 port 재사용을 막는다.

그러나 TIME_WAIT이 걸리면 해당 리소스를 쓰고 있지 않음에도 쓸 수 없는 상태가 되므로 낭비가 생긴다. 이러한 이유로 종료 통지를 한 측에만 이 상태가 남는다. 둘 중 한쪽만 TIME_WAIT을 해도 재연결을 막을 수 있는데, 만약 연결 끊김 당한 쪽에 TIME_WAIT가 남아 리소스 낭비가 된다면 이를 악용한 상황이 생길 수 있으므로 이렇게 설계된 것으로 추정된다. 

물론 TCP에서는 Seq가 동일하지 않으면 어차피 패킷을 폐기한다. 우연히 그것조차 일치할 경우를 대비해 이렇게 설계되었지만 이는 거의 불가능한 확률이다. 따라서 레지스트리 키에서 이 값을 수정하거나, 4-Way 종료 절차를 밟는 대신 RST를 보내는 등의 방법으로 TIME_WAIT을 남지 않게 하는 것이 효율적이다. 두번째 방법은 추후 설명할 소켓 옵션으로 설정할 수 있다. 

## **4) shutdown**

shutdown는 리소스 반환이나 강제적 종료 없이 Fin만 보내서 송수신을 제어할 수 있도록 한다. 만약 shutdown(SD_SEND)를 하면 Fin은 가지만 소켓은 반환되지 않으므로 상대는 계속 send() 할 수 있다. 이렇게 전송할 게 없다는 사실만 통지하고, 일방적으로 종료하지 않는 종료 방식을 두고 graceful shutdown 라고 한다. 

이는 멀티 스레드에서 종료 시점과 반환 시점을 분리할 때 사용되기도 한다. accept 스레드를 따로 만들면 closesocket 하자마자 accept가 해당 포트를 즉시 재사용하는 경우가 생긴다. 이때 다른 스레드에서 해당 소켓을 쓴다면 기존 유저 대신 다른 유저가 연결될 수도 있다. shutdown으로 Fin 전송과 closesocket을 분리하면 별도의 동기화 없이 이를 막을 수 있다. 

단, shutdown을 사용하면 Fin을 보내기 때문에 반드시 4-Way로 종료 절차를 밟으며 TIME _WAIT이 남는다. RST는 Established 상태에서만 전송되므로 이후에 즉시 종료로 소켓 옵션을 설정하고 closesocket을 하더라도 아무 효과가 없다. 또, 앞서 언급한 상대가 반응이 없는 상황도 대비할 수 없게 되므로 권장하지 않는다. 

Overlap IO를 쓸 경우 이는 더 큰 문제가 된다. TCP 연결이 남아 있어도 CloseHandle로 해당 IO 핸들을 없애면 사용하던 IO에서 다 예외를 던지며 해당 recv, send의 IO를 모두 종료시킬 수 있다. 그러나 shutdown으로 Fin을 보내는 경우 상대방이 ACK 응답 혹은 closesocket을 하지 않으면 더 이상 진행되지 않으므로 이를 사용할 수 없다.

<br/>

# **2. socket option**

## **1) option의 종류**

setsockopt으로 소켓 옵션을 설정할 수 있다. 이때 optval은 상황에 따라 넣어야 하는 데이터 형태가 다르기 때문에 char *로 캐스팅 해서 넣도록 되어 있다. getsockopt으로 현재 소켓의 옵션을 확인할 수 있는데, 송수신 버퍼 값은 제대로 반영되지 않는 것으로 보인다. IP 옵션은 잘 쓰지 않으며, 아래 옵션들도 SO_KEEPALIVE, SO_LINGER 외에는 잘 쓰지 않는다. 

SO_BROADCAST: LAN 안에서만 작동되며 라우터를 빠져나가지 못한다. 온라인 게임에서 사용할 일은 없지만 LAN 게임에서는 간혹 사용될 수 있다. 

SO_KEEPALIVE: TCP가 지금 연결된 게 맞는 상태인지 주기적으로 확인한다. 헤더를 보내서 회신이 오는지 확인하고 몇 회 이상 오지 않을 경우 연결을 끊는다.

SO_SNDBUF, SO_RCVBUF: 소켓 송수신 버퍼의 크기를 설정한다. 잘 사용되지 않는다.

SO_SNDTIMEO, SO_RCVTIMEO: block 소켓에서 일정 시간 이상 없을 때 예외를 반환 하도록 초 단위로 설정할 수 있다. non-block 소켓을 쓴다면 사용하지 않는다. 

SO_REUSEADDR: 주소 재활용 옵션으로 윈도우에서는 작동하지 않는다. 

TCP_NODELAY: 네이글 알고리즘 사용 여부를 결정한다. 


## **2) SO_KEEPALIVE**

TCP에서 연결 확인 패킷을 주기적으로 보낸다. 상대가 응답하지 않거나 RST를 보내면 소켓을 닫는데, 이는 리소스 반환이 아니라 TCP 연결 해제를 의미한다. 이를 통해 주기적으로 끊어진 연결을 해제하여 낭비를 줄일 수 있다. 기본 2시간 간격이지만 윈속에서는 초단위로 설정 가능하다. ESTABLISHED만 대상이 되며 이미 4-Way 과정에 돌입한 소켓에는 의미가 없다. 

게임 서버에서는 잘 사용하지 않는다. 만약 클라가 무한루프에 빠졌거나, 소켓을 여러개 연결하고 가만히 있는 경우 TCP 연결은 정상이므로 이걸로 감지할 수 없다. 이런 상황을 막기 위해서 어차피 L7에서는 주기적으로 정상 상태인지 확인하는 하트 비트 기능을 구현해야 한다. 따라서 SO_KEEPALIVE까지 이중으로 사용할 이유가 없다.

일반적으로 클라에서 일정 주기로 하트 비트를 쏘게 설계한 뒤, 하트 비트가 오지 않을 경우 서버에서 연결을 끊는다. 이 경우 별도의 응답을 구현할 필요가 없으며, 일정 이상 통신이 없을 때 끊기만 해도 L4의 이상 상황까지도 대응할 수 있다. 대신 서버가 일정 간격으로 하트 비트 체크를 수행해야 하는데, 응답 없이 체크만 하므로 큰 부담이 되진 않는다.  

로그인으로 인증된 계정이라면 연결이 끊기는 게 빈번한 일은 아니니 3-5분 간격으로 하트 비트를 보내도 문제가 없다. 그러나 악의적으로 대량 연결을 하는 경우 아주 짧은 시간에도 서버가 다운될 수 있다. 따라서 로그인 전단계의 연결은 로그인에 걸리는 시간을 기준으로 짧은 간격으로 체크하고, 로그인 된 유저는 좀 더 넓은 간격으로 체크하는 것이 좋다.

## **3) SO_LIGNER**

```c++
struct linger {
    u_short l_onoff;
    u_short l_linger;
};
typedef struct linger LINGER;
```

이는 closesocket 호출 시 송신 버퍼에 남은 데이터를 어떻게 할지, 언제 함수를 반환할지 등을 지정한다. onoff 0은 Linger 비활성화를 의미하는 디폴트 값으로, 바로 함수를 반환하고 남은 데이터 전송은 Background(TCP)에게 맡긴다. 이 경우 4-Way로 종료 절차가 이루어지며, 윈도우 부족 등으로 전송이 안된 것은 안된 채로 둔다.

onoff가 0보다 크면 linger 시간 동안 대기한 후 함수를 반환한다. 남은 데이터 전송을 시간 내에 완료한 경우 정상 종료(4-Way 종료)를, 못한 경우 조건에 따라 정상 종료 혹은 강제 종료(RST)를 수행한다. onoff를 1로, linger을 0로 설정할 경우 별도의 조건 없이 바로 RST 방식으로 종료하며 TIME_WAIT도 남지 않는다. 

<br/>

# **3. UDP**

## **1) UDP**

UDP는 connect, listen, accept가 없고 bind 하는 순간부터 송수신이 가능하다. netstat도 UDP 연결 상대 정보는 알 수 없으며, 필요하다면 L7에서 확인하거나 패킷 캡쳐를 해야 한다. 따라서 UDP를 쓴다면 L7에서 IP-port 테이블을 직접 만들어야 한다. TCP에서 커널이 하던 작업을 사용자가 하는 것뿐이므로 작업의 총량이 늘어나거나 느려지는 것은 아니다.

UDP는 1:1, 1:다수 모두 가능한데, 한 소켓으로 여러명을 받는다고 느려지는 것은 아니다. L3까지는 UDP, TCP 구분 없이 동일하게 데이터를 전달하다가, L4에서 다르게 분배해주는 것이기 때문이다. 그러나 한 프로세스 안에서 여러 소켓을 만들어 분배하는 등 소켓 단위 멀티 스레드를 한다면 병렬로 받는 것이나 마찬가지이므로 성능에 영향을 줄 수 있다.
 
unsigned short가 UDP 헤더의 최대 크기이므로 크기를 62235까지 설정할 수 있다. 그러나 MTU 이상으로 보내면 IP가 알아서 패킷을 자르기 때문에 L7에서 그 이하로 조각내는 작업을 해주어야 한다. 또, UDP는 데이터그램이라 수신 버퍼보다 전송된 데이터가 크면 남는 데이터는 버려지므로 충분히 큰 버퍼를 써야한다. 

UDP는 처음으로 sendto 한 데이터가 도달할 때 클라이언트 바인딩이 된다. 이때 바인딩 된 포트를 계속 사용하기 때문에 바로 recvfrom을 해도 받을 수 있으며, 이때 peeraddr로 송신측의 IP, port를 알 수 있다. UDP는 아무에게서 메시지를 받을 수 있으므로 주소 비교가 필요하다. 공유기 NAT를 통한다고 해도 포트포워딩 된 포트 번호로 우연히 들어올 수 있다.

## **2) Reliable UDP**

모바일 환경으로 오면서 IP, port로 연결을 유지하는 것이 어려워진 것, TCP 헤더가 너무 큰 것 등의 이유로 최근에는 Reliable UDP를 만들어서 사용하기도 한다.

Reliable UDP를 구현하려면 순서 보장과 재전송을 어떻게 처리할지, SEQ, ACK와 회신을 어떻게 구현할지, MSS와 연결 단위는 어떻게 고려할지, 데이터를 어떻게 뜯을지, 이들을 위해 최소 몇바이트의 헤더가 필요할지 등에 대해 고려해야 하며 트래픽 방지를 위해 반응이 느릴 때 통신 속도를 점점 느리게 하는 것 역시 필요하다. 

또, Reliable UDP가 활성화 된 배경을 고려하면 IP, port 주소가 바뀌는 상황을 대비해 둘만 아는 키(세션 ID 혹은 세션 키)를 넣어서 구분하는 등 연결에 대한 재정의가 필요하다. 이때 세션 키 탈취 위험 문제가 생기는데 TCP는 이에 대한 안전 장치로 Seq를 사용한다. 따라서 우리가 구현할 때도 Seq를 활용하거나 세션 키를 매번 바꾸는 등의 장치가 필요하다.

모바일 MMO에서 TCP를 사용할 경우, IP가 바뀔 때마다 메시지 창을 띄우거나 자동으로 버퍼링과 함께 재연결 하는 등의 방법을 쓴다. 이때마다 DB 백업을 하면 서버 부담이 커질 것이다. 따라서 일반적으로는 연결이 끊길 경우 일단 메모리 상에 데이터를 보관해두다가, 일정 시간 내 재접속이 된다면 그대로 사용하고 안된다면 종료 절차를 밟는다. 

참고로 Stateless는 요청마다 연결-해제를 반복하지만, 그럼에도 로그인 정보는 페이지에서 계속 유지되며 이는 서버가 바뀌어도 보장된다. 이는 L7의 HTTP 프로토콜에서 처리하는 것으로, 첫 연결 시에 32byte 정도의 고유 string 키를 부여하여 이를 바탕으로 각 사용자를 구분하는 방법을 사용한다. 최근에는 이 키의 탈취 위험을 없애기 위해 HTTPS를 사용한다.