---
title: Procademy, Q2, 09) Network, TCP - send, recv
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. send**

## **1) TCP의 유실 대처**

가장 기본적인 초기 TCP에서는 유실이 생기면 유실된 것 이후로 쭉 받지 않으며, 그 값들은 전부 새로 재전송 해야 했다. 그러나 최근의 TCP는 옵션 헤더에 ACK의 대역을 설정할 수 있으며 이를 통해 유실된 대역을 제외하고 ACK를 보낼 수 있다. 이 옵션을 설정할 경우 유실된 것 이후 값들을 내부에 저장하고 있다가, 유실된 대역만 재전송 되면 이를 조립하는 방식으로 유실을 처리할 수 있다.

일반적으로는 트래픽이 과할 때 유실이 생기는데, 이때 여러 클라에서 비슷한 타이밍에 재전송을 여러번 시도할 경우 트래픽이 더 악화된다. 보통 재전송을 5-6회 해도 ack가 오지 않을 경우 TCP가 자체적으로 연결을 끊는다. 클라이언트 쪽의 네트워크 문제면 분산이 되어 있으므로 큰 문제로 이어지지는 않지만, 서버 측 장치에 문제가 있는 거라면 더 많은 문제가 생긴다. 

이는 하드웨어의 문제이므로 어플리케이션 측에서 적극적으로 해결하기는 어렵다. 코드의 문제라기 보다는 특정 라우터, 특정 장비의 문제이기 때문이다. TCP 재전송 횟수를 모니터링하다가 재전송 횟수가 너무 늘어났을 때 클라 측 패킷 전송을 줄이는 방법을 시도해볼 수는 있으나, 이는 근본적인 해결책은 아니다. 혹은 부하를 낮추기 위해 프레임을 떨구거나, 전송 장비를 분산시키는 방법도 있다. 

setsockopt로 recv buffer 등을 낮추거나 늘릴 수도 있다. 대신 연결 전에 설정해야 적용이 되며, 이미 연결된 소켓의 옵션을 수정할 순 없다. getsockopt으로 보면 적용된 것처럼 보이지만 패킷 캡쳐로 확인해보면 적용이 안된 것을 확인할 수 있다. 

## **2) Zero Window**

기존의 TCP 헤더로는 현재의 window size를 모두 표현할 수 없기 때문에, 이를 나타내기 위해 window scale을 이용한다. 3-way handshake를 할 때 헤더에 window scale값을 보내는데, 이후 전송되는 window size 크기에 이 값을 l-shift 하여 계산할 수 있다. 이는 대략적인 값으로, 패킷 캡쳐로 실제 전송할 수 있는 크기와 비교해보면, 실제로는 그 값보다 큰 window를 갖는 것으로 추정된다. 

상대 측에서 recv를 하지 않는데 send는 계속 이루어진다면 window size가 줄어든다. 하지만 패킷 캡쳐로 확인해보면 지속적으로 일관성 있게 줄어들지는 않는다. 이는 window가 처음에는 작은 크기로 있다가 부족해질 때마다 늘리기 구조로 이루어져있기 때문이다. 늘어날 만큼 늘어났는데도 계속 send가 들어온다면 남은 크기가 0이 되는 순간 Zero Window를 보낸다. 

Zero Window는 ACK를 필요로 하지 않는 메시지로, 남은 공간이 없으니 보내지 말라는 의미에서 TCP가 자동으로 주기적으로 전송한다. 간혹 ACK가 오기도 하는데 이는 Seq에 반영되지 않는, 그저 수신했다는 의미의 ACK이다. 그러다가 Window가 확보되면 TCP Window Update 메시지, 즉 Zero Window에서 벗어났으니 메시지를 보내도 된다는 의미의 메시지를 보낸다.  

## **3) Zero Window에서 계속 send할 경우**

며칠이 지나도 Zero Window 상태로 대기할 뿐 연결이 끊어지지는 않는다. 그러나 이 상태에서 계속 send()를 하게 되면, TCP는 송신을 안하는데 어플리케이션은 송신 버퍼에 계속 메시지를 넣기 때문에 송신 버퍼가 다 차게 된다. 이때 block 소켓일 경우 send()에서 block이 걸리고, non block일 경우 send()는 0을, GetLastError는 would block를 반환한다. 

would block은 수신측이 아직 준비가 되지 않아 송신측이 기다려야 함을 나타낸다. 따라서 이 예외는 대기 요청으로 해석하고 처리해야 한다. 이를 보완하기 위해서는 L7에 완충용 버퍼를 만들고 그 안에서 대기하도록 한 후, 그 버퍼조차 다 찼을 때 연결을 끊도록 할 수 있다. 그러나 Zero Window 자체가 일반적인 상황은 아니므로 would block이 몇 차례 떴을 때 바로 연결을 끊어도 무관하다. 

만약 recv가 안돼서 수신 버퍼가 계속 늘어날 경우 이론적으로는 Nonpaged pool 문제도 발생할 수 있다. 수신 버퍼는 기본으로 최대 2MB를 차지하는데 500명만 연결되어도 1GB가 된다. 이런 경우 서버의 OS가 Nonpaged pool 문제로 뻗어버릴 수도 있다. 하지만 의도적으로 connect를 하고 쓰레기 메시지를 때려 붓는 상황이 아니고서야 일반적으로 이런 상황은 생기지 않는다. 

<br/>

# **2. recv**

## **1) recv buffer**

recv시 recv buffer에 아무것도 없다면, block 소켓일 경우 block이 걸리고 non-block일 경우 would block을 반환한다. 보통 1byte라도 데이터가 있으면 읽어오며, 설정한 최대 사이즈까지 한번에 받을 수 있다. 최대 사이즈가 되면 다 전송되지 않았어도 일단 반환하므로 최대 사이즈보다 큰 데이터를 보낼 땐 L7에서 보관하다가 붙여주어야 한다. 그러나 이는 block이 걸리지 않은 상황에서의 이야기이다. 

block이 걸린 recv는 PSH가 와야만 block이 풀린다. PSH는 TCP 헤더에서 flush 동작에 의해 송신된 데이터임을 알리는 컨트롤 비트로, 어플에서 send 했던 단위로 설정된다. 즉, block이 걸렸을 땐 한 메시지 단위가 모두 전송되어야 메시지를 불러오는 것이다. 만약 100byte를 send 했다면 100번째 byte가 있는 메시지만 PSH로 전송되며, 그때 flush가 일어난다. non block에서는 PSH를 읽지 않는다.

이때, block이 걸린 상황에서는 recv buffer에 메시지를 저장했다가 복사하는 대신 사용자가 설정한 버퍼에 직접 값을 저장한다. 예를 들어 window size를 3000으로 두고 send(2MB)를 한다면, 2MB를 모두 받은 시점에서야 block이 풀리며 recv가 2MB를 반환한다. 버퍼 공간이 3000인데 2MB를 저장할 수 있는 상황이 이상하게 느껴지지만 이는 사용자 버퍼에 직접 값을 쓰기 때문에 가능한 것이다. 

recv()를 하면 보통 kernel mode로 전환이 돼서, kernel 스레드가 loop를 돌며 수신 버퍼에 있던 메시지를 사용자 버퍼에 복사하고 복사가 끝난 후 반환하지만, block이 걸린 상황에서는 다르게 동작한다. 이때는 수신 버퍼를 거치지 않고 TCP 장치가 사용자 버퍼에 다이렉트로 값을 넣고, PSH가 들어왔을 때 block을 풀고 recv를 return 한다. 실제로 ret 전에 실시간으로 메모리가 변화하는 것을 볼 수 있다. 

recv 자체가 시간이 오래 걸리는 작업이므로, 이후엔 가장 큰 단위로 한번에 받은 뒤 메시지 단위로 뜯어서 가져올 것이다. 이 과정에서 Validation 체크를 진행하여 문제가 있는 패킷으로 간주 되면 연결을 끊어야 한다. 한 프레임 당 recv 하는 크기는 컨텐츠가 요구하는 반응성에 따라 조정해야 한다. 3-4000 byte 이상이 올 일은 없지만 선생님의 경우 보통 10000 byte으로 잡는다고 하셨다. 

패킷이 이상한 경우 외에도 비정상적으로 자주 보낼 경우에도 연결을 끊는다. 악의적으로 많이 보내는 패킷까지 모두 받으면 문제가 될 수 있기 때문에 메시지 별로 기준을 잡는다. 예를 들어 이동의 경우 이동 쿨타임이 있어서 그 간격으로 send가 이루어지므로 만약 이동 메시지를 쿨타임보다 자주 보내는 유저가 있으면 핵으로 간주하여 연결을 끊는다. 

## **2) 동기/비동기 IO**

동기 IO는 함수 호출 시 바로 작업을 수행하는 것을, 비동기 IO는 일단 return 한 뒤 보장되지 않은 시점에 background에서 작업을 수행하는 것을 의미한다. 예를 들어 디스크에 파일 쓰기를 비동기 IO로 요청하면, 요청한 스레드는 이후 작업을 이어서 하고 background가 파일 쓰기를 해준다. 이때 background로 지칭되는 작업의 주체는 디바이스, 즉 디스크와 TCP 등의 장치이다.

이때 대부분의 경우 Block이 걸려야만 장치들이 작업할 수 있는 상황이 된다. 그래서 수신 버퍼가 비어있어야만 직접 복사가 일어나는 것이며, 모든 IO 시스템은 이런 방식으로 동작한다. 위 상황에서 대기하는 시간만 없애고 대신 통지하는 방식으로 우회한 것이 비동기 IO의 개념이다. 따라서 비동기 IO 역시 마찬가지로 일단은 Block 되는 상황, 조건을 만들어야 작동할 수 있다. 

네트워크 장비는 메시지를 물리 메모리에 무조건 상주시키기 위해 수신 버퍼를 Nonpaged pool로 사용한다. 반면 유저가 설정한 버퍼는 유저 영역이기에 Page out이 될 수도 있다. 따라서 이렇게 TCP 장치가 직접 건드리는 메모리는 Page lock을 걸어서 Page out 되지 않게 잠궈버린다. 이는 비동기 IO에서도 동일하게 상황이 만들어진다. 

Page lock은 물리 메모리 한계치가 있다. 위 상황은 동기 IO기 때문에 저 작업이 진행되는 동안은 다른 작업이 이루어지지 않아서 문제가 생기지 않는다. 그러나 비동기 IO에서 위와 같은 작업이 여러개가 동시에 일어날 경우, Page lock이 일어날 수 있는 크기가 한정되어 있으므로 공간 부족 문제가 생기지 않도록 유의해야 한다. 

만약 메모리(recv buffer, OS가 Cache한 파일 데이터)를 대상으로 read, recv가 이루어진다면 이는 동기적으로 처리된다. 커널 모드로 전환되어 recv buffer에서 사용자 버퍼로의 복사가 이루어지는 것이다. 반면 디바이스(tcp, Disk)를 대상으로 read, recv를 하면 디바이스에서 메모리를 거치지 않고 바로 사용자의 버퍼로 복사가 이루어지며 이는 비동기적으로 처리된다. 

TCP recv에서는 recv buffer가 메모리에 해당하며, 이를 비움으로써 block이 걸어 메모리를 쓰지 않도록 설정한 셈이다. 만약 Overlap IO로 송신을 할 경우 Zero Copy를 하게 되는데, 이는 송신 버퍼를 강제적으로 못쓰게 만듦으로써 디바이스(TCP)에서 사용자 버퍼로 직접 쓰도록 하는 것이다. 이때 이루어지는 작업은 block 진입 조건 충족 및 작동 방식 측면에서 앞서 말한 상황과 동일하다. 

파일 IO의 경우 "디바이스(disk)"를 대상으로 이루어지는 듯 보이지만 실제로는 OS가 Cache한 "메모리"를 대상으로 IO가 이루어진다. 이 경우 겉으로는 비동기처럼 보여도 실제로는 동기적으로 처리된다. 비동기적으로 처리하려면 OS의 메모리를 쓰지 않도록 별도의 작업을 거쳐야 한다. Block이 걸리는 IO의 경우 내부적으로는 다 이렇게 동작한다. 

다시 돌아와, recv는 복사 완료 후에 return이 되지만 커널 모드 전환이 되어 작업을 하는 게 아니다. 그저 반환하지 않을 뿐 내 스레드는 쭉 Block이 걸려있고 실제 작업은 TCP가 처리하는 것이다. 정확히는 내 프로세스와 아예 연관 없는 내부의 다른 커널 스레드가 디바이스 드라이버를 돌린다. 즉, 작업 주체의 관점에서는 비동기 IO일 때와 동일하게 동작한다.  

이때 디바이스에서 직접 복사한다고 해서 무조건 성능에 좋은 것은 아니다. 위험성도 있고 Page lock 등에서 추가적인 오버헤드가 많이 생기기 때문이다. 같은 이유로 Overlap IO라 해서 무조건 성능에 이로운 것도 아니다. 그래서 실제로는 Select/AsyncSelect와 같은 소켓 모델을 조금 더 많이 사용하게 된다. 

참고로 MS에서는 최근에 Overlap IO를 보완하기 위해 Registered IO를 만들었다. 소켓을 예시로 들자면, 기존에 Nonpaged pool을 사용했던 recv buffer를 처음부터 유저 영역 메모리에 만드는 것이다. 이 경우 Page lock을 걸어야 되는 건 동일하지만 이를 걸었다 풀었다 하는 과정을 없애고, lock을 건 채로 쭉 유지해서 오버헤드를 줄이겠다는 관점이다. 그러나 많이 사용되지는 않는다. 