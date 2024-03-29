---
title: Procademy, Q2, 32) Network - Stateless의 연결 유지
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Stateless의 연결 유지**

웹에서는 세션 유지를 위해 모든 메시지가 세션키를 들고 다니며, 그걸 통해 동일한 연결인지 확인한다. TCP 입장에서는 매번 새로운 연결이며 포트도 매번 바뀐다. 이때, http는 텍스트로 정보를 보내고 용량이 많은 것만 gzip으로 보내기 때문에 http 사이트에 접속하여 패킷을 캡쳐해보면 GET/HTTP 와 같은 정보 뿐 아니라 세션 정보까지도 확인할 수 있다. 웹 전용 디버깅 툴 피들러를 쓰면 조금 더 편하다. 이 경우 이더넷에 오가는 모든 패킷을 캡쳐하는 와이어 샤크와 달리 웹에 대한 정보만 걸러서 보여준다. 

url에서 http ~ php까지가 실제 주소고 그 이후는 파라미터로, 해당 php 파일에 파라미터를 전달하는 것을 의미한다. 아주 과거에는 /id=abc&password=1234 와 같이 id password를 url에 담아서 인증을 매번 새로 거치는 방법을 사용했는데, 위험하다는 의견이 나오면서 쿠키 방식으로 바뀌었다. 쿠키는 특정 도메인과 통신할 때마다 url 대신 브라우저 차원에서 담아 보내는 데이터이다. 익스플로러에서 도메인 별 유효한 쿠키 목록을 확인할 수 있다. 이 경우도 쿠키에 id, password가 그대로 담겨있기에 위험하다는 의견이 나왔으며 이후에 나온 게 세션이다.

세션은 언어마다 특징이 다르다. 예를 들어 php에서는 세션이 서버의 디스크에만 있으며 클라에 노출이 되지 않는다. 언어마다 다르기 때문에 세션을 구분할 수 있는 세션 ID를 보내고, 이 세션 ID를 바탕으로 연결을 유지한다. 서버 측 언어에 따라 세션 ID 키 이름이 조금씩 달라지며 php는 PHPSSESID로 전송된다. 첫 접속 시에는 Session ID가 오지 않으며 Set-Cookie에 Session ID를 담아 보낸다. 그러면 이후 접속부터 모든 메시지에 받았던 Session ID를 실어보내고, 서버는 이 ID를 바탕으로 서버의 디스크를 열어서 id, password, 장바구니 등의 개인 정보를 불러온다. 이때 메시지의 Session ID를 알면 개인 정보를 탈취할 수 있기 때문에 이에 대한 보완으로 나온 것이 HTTP를 암호화 한 HTTPS이다. 

# **2. Stateless와 보안**

모바일 환경이 되면서 IP가 자주 변하게 되었기에 최근에는 보안에 단계를 나누고 있다. 자동 로그인은 쿠키와 세션 유효 시간을 늘린 것이다.  원래는 브라우저를 닫으면 쿠키와 세션을 삭제하고 로그아웃 하는데, 브라우저 닫힘에 상관 없이 30일간 유지하고 30일 내에 재접속 하면 또 30일 연장하는 것이다.  

이때 https라고 해도 컴퓨터 혹은 핸드폰을 볼 수 있으면 쿠키와 session ID를 알아낼 수 있다. IP 대역 검사를 하지 않는 사이트라면,  session ID만 알면 IP 대역이 완전 달라도 다른 컴퓨터에서 로그인할 수 있다. 이 때문에 크롬의 경우 기기 이름과 함께 다른 IP에서 로그인 되었음을 알려주고, 기존에 로그인 했던 기기로 확인해야만 로그인 되도록 함으로써 보안 처리를 하고 있다. 

stateless 혹은 tcp가 아닌 상황에서의 연결 유지를 구현한다면 이런 연결 기능을 직접 구현해야 한다. 직접 session ID를 발행하고, 유지하고, 보안 처리까지 해야 된다. 완벽하게 노출을 막을 수는 없지만 제 3자가 갈취하여 악의적인 접근을 하려고 할 때 시간을 끌거나 실제 유저가 알아차릴 수 있게 하는 등의 보안 처리가 필요하다. 

과거에는 XSS, Cross-Site Scripting 공격으로 갈취를 하기도 했다. 스크립트를 게시글에 숨겨놓으면 해당 페이지에 접속했을 때 스크립트가 작동하면서 세션을 공격자에게 전송하는 등 공격자가 원하는 행동을 하도록 만드는 것이다. 자신이 관리하는 사이트라면 사용자가 올리는 게시글의 < >에는 역슬래시를 자동으로 넣는 방법으로 게시글이 프로그램으로 작동하지 못하도록 막을 수 있다. 
