---
title: Procademy, Q2, 16) Network - AsyncSelect
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. WSAAsyncSelect 특징**

select는 Pooling 방식으로 사용 가능한 set이 있는지 매번 체크해야 하는 반면, AsyncSelect 방식은 사용 가능한 set이 있을 경우 윈도우 메시지를 통해 자동으로 알림을 준다. 단, 윈도우 메시지를 사용하기 때문에 윈도우에서만 사용할 수 있다. 한 윈도우 프로시저에 전달되므로 멀티스레드를 사용하지 않고도 여러 소켓을 처리할 수 있다는 장점이 있다. 반면 AsyncSelect를 처리하는 코드가 WinMain Thread에 종속되기 때문에 멀티스레드를 사용할 수 없다는 단점이 있다. AsyncSelect와 함께 멀티스레드를 사용하려고 하면 경고가 뜬다. 

select와 달리 set을 한번만 등록하면 되며 소켓 개수 제한도 없다. wMsg에 지정한 유저 커스텀 메시지를 넣고, lEvent에는 소켓 별로 감시할 이벤트를 지정한다. 이때 FD_CONNECT로 connect를 감시할 수도 있다. select 모델에서는 Non-block socket일 때 connect에서 write set으로 반응이 오면 연결 성공으로, except set에 반응이 오면 예외로 처리했기 때문에 바로 connect를 시도하면 무조건 에러가 떴다. 반면 AsyncSelect에서는 FD_CONNET 이벤트를 통해 connect 신호를 감시할 수 있다.

WSAAsyncSelect 특성 상 Block Socket을 사용하면 아예 프로그램이 멈출 수 있기 때문에 자동으로 논블로킹 모드로 전환된다. listen 소켓은 accept를, accept에서 반환되는 소켓은 read, write를 감시해야 하는데, 이때 반환되는 소켓들은 다른 값으로 설정되어 반환되므로 직접 수정해야 한다. 또, 간혹 Would Block이 뜰 수 있으니 이에 대한 예외 처리가 필요하다. 

이때 메세지 별로 적절한 함수를 호출하지 않으면 동일 소켓에 대해 해당 메시지가 다시 발생하지 않는다. 폴링 방식인 select와 달리 이벤트 방식인 AsyncSelect는 상태가 아닌 상태 "변화"를 알려주는 것이기 때문이다. 예를 들어 소켓이 읽을 수 있는 상태임을 알린 이후에는 읽을 수 없는 상태로 바뀌어야 다시 읽을 수 있는 상태로의 “변화”가 생길 것이다. 만약 수신버퍼에 1000이 있는데 500만 recv 한다면 추가적인 수신이 오지 않아도 적절한 처리를 해준 것이기 때문에 바로 FD_READ가 뜨고 이를 다시 수신할 수 있게 된다.

FD_WRITE 역시 상황이 바뀌었을 때, 즉 보낼 수 없었는데 보낼 수 있게 될 때 한번만 메시지를 보내며 계속 보낼 수 있는 경우에는 보내지 않는다. 따라서 연결에 성공했을 때와 Would Block이 뜨다가 해결되었을 때만 메시지가 온다. 따라서 이 모델에서는 FD_WRITE에 의존하지 않고 보낼 게 있을 때 그 자리에서 보내는 구조를 채택하게 된다. 평상시에는 그냥 send를 호출하며 송신 버퍼가 가득 찬 상황 등에만 FD_WRITE를 이용한다.  



<br/>

# **2. WSAAsyncSelect 예제**

네트워크 이벤트가 발생하면 wParam으로 소켓이, lParam에는 이벤트와 오류 관련 데이터가 들어오며 이를 형변환 해서 사용한다. 이때 이벤트는 중첩해서 오지 않기 때문에 일반적으로는 switch-case 문을 활용한다. 

```c++
LRESULT CALLBACK WndProc(...)
{
	switch (message)
    {
	case UM_SOCKET:

		if(WSAGETSELECTERROR(lParam))
		{
			HandleError();
		}

		switch (WSAGETSELECTEVENT(lParam))
		{
			case FD_CONNECT:
				g_bConnect = true;
				break;

			case FD_READ:
				ReadEvent();
				break;

			case FD_WRITE:
				WriteEvent();
				break;

			case FD_CLOSE:
				// 종료 처리
				break;
		}
    }
} 
```

```c++
void ReadEvent()
{
    char buffer[MAX];
    ret = recv(..., buffer, ...);
    RecvRingBuffer.Enqueue(buffer, ret);
    while(RecvRingBuffer.GetUseSize() >= 0)
    {
        if(RecvRingBuffer.GetUseSize() < HEADER_LEN)
            break;

        HEADER header;
        RecvRingBuffer.Peek(&header, HEADER_LEN);
        if(header.len + HEADER_LEN > RecvRingBuffer.GetUseSize())
            break;

        DRAW_PAKCET packet;
        RecvRingBuffer.MoveRear(HEADER_LEN); 
        RecvRingBuffer.Dequeue(&packet, Header);
        // To-do: ...
    }
}
```

header의 type을 바탕으로 분기를 태워서 받은 메시지에 해당하는 적절한 처리를 해주면 된다. 

이때, select는 loop를 돌며 recv, send를 반복하지만 AsyncSelect는 loop를 돌지 않는다. 따라서 send 해야 할 데이터를 그 자리에서 다 보내지 않는다면 다음에 send 할 데이터가 생기기 전까지 전송되지 않는다. 따라서 SendRingBuffer가 비거나 WouldBlock이 뜰 때까지 loop를 돌리며 send를 처리해야 한다.