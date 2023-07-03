---
title: Procademy, Q2, 14) Network - L7 프로토콜, Ring Buffer
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. L7 프로토콜**

L7 프로토콜의 헤더에는 우선 필수적으로 페이로드 길이와 타입이 들어간다. 또, 추가적으로 체크섬을 넣기도 하는데, 이는 악의적 조작을 감지하기 위한 안전장치 혹은 암호화에 대한 검증 용도로 사용된다. 그러나 암호화를 해도 패킷을 그대로 복사해서 여러번 다시 보내는 매크로를 만들어 악용할 수 있다. 이걸 막기 위해서는 한번 쓴 메시지는 두번 사용하지 못하도록 시간값 혹은 매번 변하는 고유 번호 값을 헤더에 넣어서 비교해야 한다.  

흔하진 않지만 맨 앞에 특정 값을 넣어서 쓰레기 패킷을 거를 수 있다. 맨 앞에 len이 오면 그 크기만큼 메시지를 기다리며 낭비가 생기므로 이를 막기 위한 것이나, 크게 의미 있는 작업은 아니다. 이러한 프로토콜 값은 보안을 위해 일주일에 한번 정도의 간격으로 변경해야 한다. 같은 맥락에서 전역 변수 위치 등도 비슷한 간격으로 바꾸어 악의적인 분석에 혼란을 준다. 단, PC 온라인 게임과 달리 모바일 게임의 경우, 플레이스토어에 반영되기까지 시간이 걸리므로 버전별로 분기처리를 하는 등의 방법으로 하위 호환성을 보장해야한다. 

<br/>

# **2. Ring Buffer**

## **1) Ring Buffer 구성**

매번 recv, send를 하는 것은 성능 저하가 심하므로 큰 버퍼를 통해 한번에 읽어온 뒤 L7 헤더의 len을 바탕으로 데이터를 잘라서 사용하는 게 일반적이다. 이 과정을 반복하기 위해 byte 단위의 원형 큐인 Ring buffer를 사용한다. recv용 Ring buffer는 TCP에서 패킷을 분리하여 가져오기 위해 필요하다. UDP의 경우 송신 크기 그대로 전달되므로 없어도 된다. send 용 Ring buffer는 wouldblock에 대한 대처를 위해, 그리고 잦은 send로 인한 트래픽을 줄이기 위해 사용된다. 

```c++
class RingBuffer
{
public:
    RingBuffer(void);
    RingBuffer(int iBufferSize);

    void Resize(int size);
    int GetBufferSize(void);
    int GetUseSize(void);
    int GetFreeSize(void);

    int Enqueue(char* chpData, int iSize);
    int Dequeue(char* chpData, int iSize);
    int Peek(char* chpDest, int iSize);
    int DirectEnqueueSize(void);
    int DirectDequeueSize(void);

    void ClearBuffer(void); 
    int MoveRear(int iSize);
    int MoveFront(int iSize);
    char *GetFrontBufferPtr(void);
    char *GetRearBufferPtr(void);
}

```

Ring Buffer의 필수 멤버 함수는 위와 같다. 버퍼가 크면 메모리 낭비가, 작으면 트래픽 낭비가 생기니 한 프레임에 송수신 되는 메시지를 넉넉히 담을 수 있는 크기로 잘 조율해야 한다. Peek은 버퍼는 그대로 두고 데이터를 읽기만 할 때 사용하는 것으로, 이때 front는 움직이지 않는다. 이걸로 헤더만 먼저 확인을 하고, 그걸 바탕으로 필요한 len 만큼 받은 게 확인되면 버퍼에서 데이터를 가져온다. 

외부에서 rear, front의 위치를 강제로 옮기고 가져오는 것은 객체지향 관점에서는 잘못된 것이지만 성능을 위해서 허용한다. 포인터를 직접 얻어서 직접 쓰기를 하고, MoveRear, MoveFront를 함으로써 불필요한 복사를 줄이면서도 Enqueue의 효과를 낼 수 있다. 콘텐츠 단에서 사용할 때는 Enqueue, Dequeue를 사용하지만 데이터를 send, recv 할 때는 성능을 위해 이렇게 할 수 있다. 

## **2) Buffer가 가득 차는 상황**

Resize는 런타임 중 버퍼가 다 찼을 때 크기를 변경하는 것으로, 메모리 할당과 카피가 일어나니 최대한 피하는 게 좋다. 게임은 접속하는 순간 제일 많은 데이터를 보내므로 수용 가능 인원을 초과하고 이후에 접속한 사람들이 캐릭터 생성 메시지로 인해 버퍼 초과를 겪을 수 있다. 이동이나 공격 메시지에서 이런 일이 생길 가능성은 거의 없다. 

위 경우 외에도 recv 버퍼에 온 헤더의 len이 잘못됐을 때, 상대의 tcp 수신 버퍼가 다 차서 내 tcp 송신 버퍼에 이어 send 버퍼까지 찼을 때도 위 상황이 생긴다. 이때 앞서 말한 수용 가능 인원 초과 상황을 포함, 이들을 모두 잘못된 상황으로 간주하여 Resize를 호출하는 대신 연결을 끊을 수도 있다. 어느 쪽을 택하든 버퍼가 다 차는 것은 일반적이지 않은 상황이므로 파악을 위해 기록을 남겨야된다. 

버퍼가 다 찼을 때 중간에 send, recv를 하는 것도 가능한 방법이지만 그럴 경우 많이 느려지기 때문에 일반적으로 사용하는 방법은 아니다. 

## **3) Ring Buffer Test**

데이터가 잘 들어가는지, 포인터 위치가 적합한지 등을 확인하기 위해 랜덤 크기의 Enqueue, Dequeue를 여러 번 실행하여 검토하는 테스트를 거쳐야 한다. 문제가 생겼다면 코드를 보며 짐작하는 대신 front, rear 값을 바탕으로 문제를 찾아야 한다. 이때 random seed를 상수로 해야 이후 재현이 용이하다. 또, 보통 버퍼 경계에서 문제가 생기니 버퍼 크기는 작게 하는 게 좋다. 