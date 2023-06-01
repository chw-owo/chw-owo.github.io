---
title: Procademy, Q2, 15) Network - Ring Buffer
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Ring Buffer의 적용**

send, recv는 시간이 오래 걸리는 작업이므로 필요할 때마다 매번 호출하는 것보다는, Ring Buffer를 통해 저장해두었다가 한번에 처리하는 것이 성능상 이롭다. 이를 위해 세션에 send 전용 Ring Buffer, recv 전용 Ring Buffer를 각각 들고 있으면서 send, recv 대신 RingBuffer.Enqueue, RingBuffer.Dequeue를 호출할 수 있다.

```c++
char buffer[RECV_RINGBUFFER_MAX]; 
int recvRet = recv(s, buffer, pSesion->RecvRingBuffer.GetFreeSize() , 0); 
int recvdSize = pSesion->RecvRingBuffer.Enqueue(buffer, ret);
if (recvRet != recvdSize) HandleError();
```
recv 전용 RingBuffer를 통해 받은 메시지는 header의 len을 바탕으로 필요한 길이만큼 읽어서 적절하게 처리한다. 

```c++
int sendRet = pSesion->sendRingBuffer.Enqueue(...);
if (sendRet ...) HandleError();
```

send 전용 RingBuffer의 경우 위처럼 보내야 할 메시지가 있을 때마다 메시지를 버퍼에 넣어둔 뒤, select에서 한번에 send 함으로써 성능을 높인다. 

```c++
fd_set wset;
FD_ZERO(&wset);
for(...)
{
    FD_SET(socket, &rset);
    if(Session->sendRingBuffer.GetUseSize() > 0 )
    {
        FD_SET(Session->socket, &wset);
    }
}

...
select(...);
...

for(Session in SessionList)
{
    if (FD_ISSET(Session->socket, &wset))
        SendProc(...);
}
```

wset은 송신 버퍼에 여유가 있을 때마다 select 되기 때문에 위처럼 send 전용 RingBuffer에 보낼 것이 쌓인 상황에만 FD_SET을 호출하는 것이 적절하다. 이때 Blocking 소켓은 내가 보낸 인자만 반환되지만 Non-Blocking 소켓은 일부만 전송되는 상황도 생길 수 있다. 이 경우 Dequeue 했던 것을 다시 Enqueue 할 수는 없기 때문에, 희망하는 크기를 모두 Dequeue 하는 대신, Peek을 이용하는 것이 더 안전하다. 

```c++
// SendProc
char buffer [MAX];
int peekRet = sendRingBuffer.peek(buffer, max);
int sendRet = send(..., buffer, peekRet, ...);
if (sendSize ==SOCKET_ERROR) HandleError();
SendRingBuffer.MoveFront(sendRet);
```

예제 코드는 위와 같다. Peek을 통해 일단 버퍼 안의 값을 보존한 채 전송한 후, send의 return 값만큼 MoveFront를 호출하여 전송에 성공한 만큼 front 값을 바꾼다.


