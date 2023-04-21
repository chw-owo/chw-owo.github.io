---
title: Procademy, Q2, 06) Network, Basic - TCP-UDP, socket
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. TCP-UDP**

|TCP | UDP |
|----|-----|
| 연결형 프로토콜 | 연결 없이 통신 |
| 재전송으로 신뢰성 보장 | 신뢰성 보장되지 않음, 유실이 생겨도 재전송 x |
| 데이터 경계 구분이 없는 바이트 스트림 | 메시지 단위로 데이터 경계 구분이 있음 |
| 일대일 통신만 가능 | 일대다 통신 가능 |

L2는 peer to peer 연결이므로 전달 중 순서가 바뀔 일이 없지만 L3에는 라우팅이 들어간다. 만약 경로 중 한 라우터가 죽었다면 다른 라우터로 우회하기 때문에 순서가 보장되지 않는다. 이때, TCP는 패킷에 순서를 매겨서, 건너뛰고 패킷이 도착한 경우 이를 받지 않는 방식으로 순서를 보장한다. 반면 UDP는 유실을 허용하고 순서를 보장하지 않기 때문에 건너 뛰고 도착한 경우 유실된 것으로 간주하여 드랍한다.

동영상, 음악의 스트리밍 서비스는 끊기더라도 빠른 전송이 중요하므로 UDP를 사용한다. 반면 게임 서버에서 클라의 데이터를 받는 경우, 유실을 허용해선 안되므로 신뢰성 있는 통신이 가능하도록 L7가 보조해야 한다. 이를 Reliable UDP라고 한다. 일반적인 UDP의 경우 좌표 동기화를 위해 보조로 사용될 순 있지만 메인 프로토콜에 쓰기엔 안정적이지 못해 권장하지 않는다.  

UDP는 L3에서 쓰던 기능에 port만 추가했다고 봐도 된다. UDP 헤더는 매우 작은 용량으로 Src port, Dst port, Lenght, checksum 각각 2byte씩 총 8byte의 크기를 갖는다. 반면 TCP 헤더는 큰 용량의 가변 헤더로, 일반 통신 시 20byte 크기를, 3 Hand Shaking 시 32byte 크기를 갖는다. 여기도 checksum이 있는데 원래는 오류 검사 용이지만 L2에서 이미 검사 하기 떄문에 큰 역할은 없다. L7 계층에서 구현하는 오류 검사 역시 주로 안전한 메시지인지 확인하기 위함이다.

연결 하나 당 소켓 하나가 반환되는 TCP와 달리, UDP는 소켓 하나를 열면 그 곳으로 모든 클라이언트의 메시지가 오며 그 분류는 L7에서 해주어야 한다. 또, UDP는 Send 한 크기 그대로 전송이 되는 반면, TCP는 패킷이 뭉쳐져서 도착하며 데이터 경계를 L7에서 구분해야 한다. 예를 들면 10, 50, 100 byte 세개를 보냈을 때 160byte 한번, 80byte 두번, 60byte/ 100byte 등으로 도착하는 식이다.  

<br/>

# **2. socket**

## **1) 윈속이란?**

처음으로 등장한 소켓은 UNIX의 버클리 소켓이며, 이후 윈도우가 WinSock을 만들었다. IOCP, 비동기 IO 등 모두 WinSock을 바탕으로 구현되었으므로 윈도우를 사용한다면 WinSock을 따르는 게 좋다. ws2_32.lib를 프로젝트 속성 혹은 pragma에서 넣어야 사용할 수 있다.

## **2) 소켓 생성 문법**

```c++
int WSAGetLastError(void);
```

소켓은 에러의 이유가 다양하고 예측이 어렵기 때문에 반드시 위 함수를 이용해야 한다. 에러가 나고 printf 등을 호출하면 에러 코드가 성공 코드로 덮어 씌워지므로 반드시 예외 즉시 에러 코드를 호출하고 로그를 남겨야 하며, 절대 그 사이에 다른 코드를 실행해서는 안된다. 소켓은 성공했을 때 0을 return 하는 경우도 많아서 반환값을 잘 확인해야 한다. 

```c++
int WSAStartup (wVersionRequested, lpWSAData);
WSACleanup();
```
윈속 DLL을 초기화 한다. 클래스로 만들면 이걸 생성자에 넣기도 하는데 여러번 호출되어도 상관 없다. WSAData에서 관련 정보를 제공하지만 실질적으로 잘 사용되지 않는다. 

```c++
SOCKET socket ( int af, int type, int protocol );
int closesocket (SOCKET s);
```

파일은 파일을 열면 핸들이 나오지만, 소켓은 먼저 만들고 이후에 어떻게 쓸지 정하게 된다. IP를 쓴다면 af는 AF_INET를,  type은 TCP일 땐 SOCK_STREAM, UDP일 땐 SOCK_DGRAM을 지정한다. 이를 지정하면 protocol은 자동 결정되므로 0을 넣어도 정상적으로 동작한다. 이 함수는 소켓 핸들을 반환하며 이는 정수 값을 가진다. 

## **3) 소켓 주소 구조체**

```c++
typedef struct sockaddr {
	u_short	sa_family;
	char 		sa_data[14];
} SOCKADDR;
```

소켓에서는 어떤 주소 체계든지 받을 수 있도록 위와 같은 더미 구조체를 제공한다. 실제로 sockaddr 타입을 사용하진 않지만 sockaddr 타입으로 캐스팅 하여 전송하게 된다. 

```c++
typedef struct sockaddr_in {
	u_short	sin_family;
    u_short	sin_port;
    u_short	sin_addr;
	char 	sin_zero[8];
} SOCKADDR;
```

sin_zero[8]는 규격 맞추려고 넣은 변수이다. 최근에는 16byte를 초과하는 새로운 주소 체계들이 등장했으나, address의 family를 통해 계속 이전의 구조체로 소통하고 있다. 2byte의 family가 앞에 나온다는 건 어떤 주소 체계든지 동일하며, IPv6는 위 구조체 대신 sockaddr_in6을 사용한다. 

IP 주소를 저장하기 위한 in_addr, in6_addr의 경우 union을 사용한다. 이를 통해 조사식에서도 IP 주소를 간단하게 확인할 수 있다. 이는 u_long S_addr 가 직관적이지 못하기 때문에 제공되는 것으로 char 단위로 확인하면 쉽게 읽을 수 있다.  

## **4) 바이트 오더**

우리가 접하는 컴퓨터는 대부분 리틀 엔디안을 쓰지만 네트워크 장비는 빅 엔디안을 사용한다. 다른 데이터는 어차피 그대로 전송되기 때문에, 네트워크 장비가 쓰는 데이터인 주소 영역만 빅 엔디안으로 전송하면 된다. family는 소켓 API가 필요로 하는 구간이므로 리틀 엔디안이 들어가고, 포트와 주소에 대한 부분만 해당된다.  

```
htons: host (little) to network (big) short (2byte)
htonl: host (little) to network (big) long  (4byte)
ntohs: network (big) to host (little) short (2byte)
ntohl: network (big) to host (little) long  (4byte)
```
이를 위해 위와 같은 매크로를 제공한다. 그러나 일반적으로는 도메인 조회를 통해 얻은 주소를 문자열 형태로 받아서 사용하기 때문에 위 매크로는 많이 사용되지 않는다. 

```c++
unsigned long inet_addr(const char* cp);
char* inet_ntoa(struct in_addr in)
```

위의 문자열과 숫자 간 변환 함수는 WCHAR를 지원하지 않는다. 또, 두번째 함수의 경우 내부에서 동적할당을 해서 주는 것처럼 생겼는데 이를 외부에서 해제해야 하는지에 대해 명확하게 안내하지 않고 있다. 이는 소켓 API 내부에 있는 공간으로 추정되며, 메모리가 새지는 않지만 안정적인 구조라 보기는 어렵다. 이런 이유로 VS는 위 함수에 경고를 띄워주며 되도록 사용하지 않기를 권장한다. 

## **5) 도메인**

```c++
typedef struct hostent {
	
    char*   h_name;
    char**  h_aliases;
    short   h_addrtype;
    short   h_length;
	char**  h_addr_list;
} 
#define h_addr h_addr_list[0];
```

이는 도메인에 대한 정보를 반환하는 구조체이다. h_addr_list에서 ip 주소들을 링크드 리스트로 들고 있는데, 0번 index가 h_addr로 define 되어 있다 보니 보통은 h_addr를 사용한다. DNS가 매번 순서를 바꿔주기 때문에 문제가 되지 않는다. 

aliases는 도메인에 대한 별칭을 정하는 것으로, 소규모로 운영할 때는 필요가 없다. 하나의 서버에 여러개의 서비스를 올릴 경우 하위 도메인을 여러개로 나누어 사용해도 동일한 IP를 갖는다. 이러한 IP를 낱개로 관리하면 한번 바꿀 때 다 바꾸어야 한다. 이때 aliases 형 IP를 하나 두고 각 하위 도메인들의 IP를 aliases로 지정하면 쉽게 IP를 바꿀 수 있다.

```c++
struct hostent *gethostbyname (const char* name);
struct hostent *gethostbyaddr (const char* addr, int len, int type);
```

이들은 도메인 이름과 IP 주소를 상호 변환하는 함수로 hostent를 반환형으로 갖는다. 이 경우도 inet_ntoa와 동일한 문제가 있으므로 GetAddrInfo와 같은 다른 함수 사용을 권장한다. 이 함수 역시 포인터를 반환하긴 하지만, MS 공식 문서에 메모리 누수를 피하려면 freeaddrinfo를 해야 한다는 설명이 제공된다. 이처럼 할당하지 않은 포인터를 반환하는 경우엔 해제가 필요한 함수는 아닌지 살펴보는 것이 좋다. 

```c++
BOOL DomainToIP(WCHAR *szDomain, IN_ADDR *pAddr)
{
    ADDRINFOW   *pAddrInfo;
    SOCKAD_IN   *pSockAddr;
    if(GetAddrInfo(szDomain, L"0", NULL, &pAddrInfo) != 0)
    {
        return FALSE;
    }
    pSockAddr = (SOCKADDR_IN*)pAddrInfo->ai_addr;
    *pAddr = pSockAddr -> sin_addr;
    FreeAddrInfo(pAddrInfo);
    return TRUE;
}
```

GetAddrInfo를 활용한 예제는 위와 같다. GetAddrInfo의 경우 도메인 이름을 넣으면 *PADDRINFO를 반환하는데, 이는 주소 뿐 아니라 ServiceName, port와 같은 정보를 포함한다. 반환 값의 ai_next를 통해 다음 주소를 확인할 수 있으며 nullptr가 나올 때까지 출력하면 IP 전체 주소를 볼 수 있다.  이때 소켓 API에서 조회하는 건 내 OS가 캐싱해서 들고 있는 정보이므로 dnsflush 하지 않는 한 순서가 섞이지 않는다. 

<br/>

# **3. socket TCP**

## **1) TCP 연결 과정**

TCP는 listen, connect 이후에 send를 진행한다. 서버/클라의 차이는 listen-connect를 하는지, accept를 하는지에서만 드러나며, 연결 이후 통신 시에는 동일하게 동작한다. connect를 해서 연결이 되면 연결만을 위한 소켓을 하나 반환하는데, TCP는 1:1 연결이므로 5000명을 받고자 한다면 listen 소켓 하나와 클라쪽 소켓 5000개가 필요할 것이다. 서버 쪽 소켓의 이름은 listen_socket으로 통일하는 게 일반적이다. 

bind()는 내가 가진 IP, port 중 어떤 것을 소켓과 연결할지 지정하는 것이다. 소켓을 만들면 내가 가진 IP의 어떤 port를 쓸 지 결정해야 하는데, 이미 쓰고 있는 port면 에러가 난다. 이때 0.0.0.0:50000으로 설정했는데 내 PC의 이더넷이 여러개라면 모든 IP의 50000번 포트가 소켓에 바인딩 된다. 지정하고자 하면 구체적으로 하나의 주소를 지정해야 하며, 이 경우 여러개를 지정할 수는 없다. 일반적으로는 0.0.0.0을 사용한다. 

natstat을 보면 LISTENING 목록이 뜨는데 0.0.0.0이라면 연결된 모든 이더넷에서 받을 수 있다. 127.0.0.1은 보통 프로세스 간 통신을 위한 루프백 포트로 이 주소는 외부에서 접근할 수 없다. 일반적으로 서버에는 클라, DB, 관리용 서버, 모니터용 서버 등이 다 연결되는데 이때 클라와 중요한 연결은 분리된 이더넷을 통해 연결하면 조금 더 안전하게 사용할 수 있다. 클라는 공인 IP로, 중요한 연결은 사설 IP로 바인딩 하여 보호하는 것이다.

바인딩이 되고 나면 listen()으로 소켓을 listen 상태로 바꾸고, 그러면 클라에서 연결 가능한 상태가 된다. accept는 연결될 때마다 소켓을 반환한다. 따라서 bind, listen은 한번씩 하고 accept는 사람이 들어올 때마다 새로 해주어야 한다. 

## **2) 3-Way Handshake**

![image](https://user-images.githubusercontent.com/96677719/232972376-9c806f88-151d-4438-870b-ea324cc03848.png)

클라에서 최초 연결을 요청할 때 SYN가 1인 TCP 헤더를 하나 보낸다. 정상적으로 listen 중이었다면 서버는 SYN과 함께 회신 역할의 ACK를 보내고, 클라가 그에 대한 응답으로 ACK를 보내면서 연결이 수립 된다. 즉 SYN, ACK를 한번씩 주고받는 셈이다. 이 과정을 3-Way Handshake라고 부른다. 이는 listen 상태에서 connect() 요청이 들어오면 이후에 사용자가 추가적인 명령을 입력하지 않아도 TCP가 자동으로 진행시키며, "accept() 호출과 상관 없이" 연결이 성사된다.

즉, accept() 호출은 3-Way Handshake 및 연결 성사와는 아무 상관 없다. 받은 메시지는 backlog 큐 버퍼에 쌓이는데 accept()는 이 큐에서 Dequeue 하는 기능만을 수행한다. 따라서 연결이 안된다면 listen을 했는지, IP가 맞는지, 방화벽에서 막히지 않았는지 등을 확인해야지 accept를 확인할 필요는 없다. 참고로 accept() 전에 연결이 끊어져도 이미 backlog 큐에 들어간 메시지는 유지되며, 대신 accept()하자마자 연결이 끊어질 것이다.  

이 backlog 큐도 송수신 버퍼와 마찬가지로 Nonpaged pool을 차지한다. 따라서 backlog 큐가 늘어나면 Nonpaged pool도 같이 늘어나며, 65535를 다 채우면 거의 1GB까지 올라간다. 이는 송수신 버퍼와 backlog 큐 모두 하드웨어 인터럽트, 정확히는 이더넷 카드의 인터럽트를 사용하기 때문이다. 이더넷 카드 인터럽트를 바탕으로 L2, L3, L4가 차례대로 작동하기 때문에 TCP가 사용하는 내부적인 버퍼는 다 Nonpaged pool을 사용하게 된다. 

