---
title: Procademy, Q2, 31) Etc - Log
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 게임 로그**

게임 운영에서 로그는 일반적으로 게임 로그와 시스템 로그로 나뉜다. 공식 명칭은 아니고 그냥 그런 느낌... 

게임 로그는 게임 운영 중 플레이 상황에 대해 남기는 로그로, 어떤 유저가 어떤 행동을 했는지 기록해두었다가 기획자, 개발자 등 모두가 사용한다. 플레이어, 화폐, 아이템을 추적하는 것이 주 목적이므로 보통 hp, 좌표 같은 자잘한 정보까지 저장하지는 않고, 특정 유저가 무엇을 했고 특정 아이템을 어디서 얻었는지 등에 대해 기록을 남긴다. 이를 바탕으로 디버깅 시 참고하거나, 유저의 문의에 대응하거나, 핵 및 금전 거래 같은 불법적인 상황을 추적한다. 

포션 같은 소비성 아이템은 고유 키를 부여하지는 않고 사용하는 것만 체크한다. 레어템의 경우 고유 키를 부여해서 로그에 남김으로써 아이템의 이동을 추적한다. 대부분의 운영사는 공식적으로 아이템, 화폐 거래를 금지하기 때문에 그런 걸 추적할 때도 사용한다. 작은 단위는 그냥 방치하기도 하지만, 큰 거래 내역은 본보기로 잡아내야 하기 때문에 게임 로그는 DB에 저장을 하여 최소 몇년 간 보관한다. 

게임 로그의 경우 모든 행동을 그때 그때 반영하는 대신 데이터를 DB에 저장하는 순간에 함께 저장한다. DB 저장 트랜젝션과 함께 처리하여 트랜젝션을 안하더라도 로그 쿼리는 전송한다. 롤백 내용은 DB에 저장되지 않기 때문에 로그 역시 저장되지 않는다. 매일 몇 GB에서 규모가 큰 경우 몇 백 GB까지 나오기 때문에 시간 순으로 저장한다. 64bit의 경우 보통 월 단위를 사용하며 32bit는 4GB 이상의 파일을 만들 수 없으므로 일 단위로 저장하는 경우가 많다.  

게임 로그는 보통 Server, Type, Code, AccountNo, Param1, Param2, Param3, Param4, ParamString 의 형태를 가진다. 이때 Code는 "돈 획득" 대신 "몬스터를 잡아서 돈 획득" 과 같이 쓰는 등 원인을 포함하여 상세하게 나눈다.  실수로 안남는 상황이 생기면 안되기 때문에 검증까지 시간이 오래 걸려서 신입에게는 잘 맡기지 않는다. 콘텐츠와 직접 연관이 되다보니 분리된 라이브러리로 만들기는 어렵다. 

<br/>

# **2. 시스템 로그**

시스템 로그는 함수가 return 되었을 때 100% 남아있어야 하는데 외부 원격지에 저장하는 방식은 이를 보장할 수 없다. DB에 남기는 것도 네트워크로 보내는 것도 다 유실 가능성이 있기 때문이다. 따라서 시스템 로그는 파일 저장을 기본으로 한다. 이 경우 플레이 내역을 보는 게 아니라 개발자들이 에러 추적할 때 참고하는 것이 목적이므로 상황 별로 분류하는 걸 권장한다. 이때 분산해서 저장한다면 저장 순서에 대한 카운터가 있어야 추합이 필요할 때 편하다. 

DEBUG, ERROR, SYSTEM을 기본으로 나누며, CRITICAL, WARNING, INFORM 등 더 나누기도 한다. ERROR LEVEL이라면 ERROR 이하 단계는 남지 않는다. 개발 땐 DEBUG로, 실제 운영 중일 땐 ERROR로 실행하는 것이 일반적이다. SYSTEM은 서버 가동, 스레드 생성, 파일 생성, DB 연결 등이 저장된다. 이를 통해 예외가 생겼을 때 어디까지 정상이었는지 확인할 수 있다. 이때 레벨은 런타임에서도 바꿀 수 있게끔 전처리기 대신 if 문으로 처리하는 것이 좋다. 전처리기로 하면 바꾸고 싶을 때마다 재빌드를 해야 되는데 현업에서는 재빌드 시 QA부터 새로 검사 해야되기 때문이다. 

시스템 로그는 어차피 문제가 생겼을 때 남는 것이지 정상 상황에서는 남지 않기 때문에 양과 성능을 고려하지 않고 100% 남는 것을 제 1목표로 만든다. 그러므로 open-write-close를 매번 하는 게 좋고, 그게 영 싫다면 fflush라도 해야 한다. 이때 읽고 쓰기는 OS에서 파일을 캐시하여 메모리에서 진행되는 것이지 디스크에 저장되는 게 아니기 때문에 fflush만으로는 디스크 저장을 보장할 수는 없다. 또, 서비스 중에 로그를 read 모드로 열어서 확인할 수 있도록 해야 하며 로그가 잘못 남는 경우를 대비해 삭제 기능도 넣는 것이 좋다. 

싱글톤을 쓸지 전역 함수으로 둘지는 상관 없지만, 콘텐츠와 네트워크 라이브러리 어디서든 호출할 수 있어야 한다. 또, FormatString/ SafeString/ 가변인자 등과 같이 변수와 문자열을 한번에 받을 수 있는 형태로 가야 한다. 로그에서는 예외가 나면 절대 안되니 되도록 예외를 던지지 않고 크기에 맞게 잘라주는 SafeString을 권장한다. 이때 문자가 잘린 것도 로그로 남겨서 파악할 수 있도록 해야 한다. 또, Hexa 로그로 메모리 주소 및 메모리 덩어리도 덤프 뜨듯이 남기는 것이 좋다. 

디버깅 말고 모니터링 용도로 쓸 수도 있는데 이때는 성능을 조금 고려해야 한다. 이것 때문에 스레드를 분리하기도 하는데 이 경우 프로세스가 죽어버린다면 통째로 날라갈 수 있다는 점에서 다소 위험하다. 따라서 디버깅 용도의 로그는 동기적으로 처리하고, 모니터링 용도의 로그만 따로 분리하여 유실을 조금 허용하는 비동기적 구조로 가는 것이 좋다. 이 경우 당연히 파일도 따로 나온다. 매번 합치는 건 부하가 크고, 하루 단위로 취합하는 프로그램을 만드는 것 정도는 괜찮다. 