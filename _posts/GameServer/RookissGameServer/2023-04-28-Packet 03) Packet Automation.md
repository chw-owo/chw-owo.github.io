---
title: Packet 03) Packet Automation
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Protocol Buffer의 적용**

Protocol Buffer는 구글에서 제공하는 패킷 처리 프로그램이다. 아래 예제와 같이 사용하며, 실제로도 많이 사용된다고 한다. 가변 길이 데이터 앞에는 repeat를 붙여서 표시한다. 그 외에도 다소 복잡한 기본 세팅이 필요한데, 사용할 일이 생길 때 공식 문서를 참조하는 게 좋을 것 같다. 

```c++

```

그 외에 제공되는 것으로는 Flat Buffer가 있다. Flat BUffer는 버퍼에 데이터를 바로 넣는 방식을 사용한다. 그 결과 복사 비용이 절약되어 좋은 성능을 보이는 대신, 가변길이 데이터가 위 예제처럼 중첩되어 나타나는 경우 사용자 입장에서는 다소 사용하기 복잡해진다. 

<br/>

# **2. 라이브러리와 콘텐츠 분리**

<br/> 

# **3. Packet Generator**

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
