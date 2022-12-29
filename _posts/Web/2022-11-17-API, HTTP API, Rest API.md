---
title: API, HTTP API, Rest API
categories: Web
tags: API
toc: true
toc_sticky: true
---

## **1. API**

**📌 API란?**

🔎 Application Programming Interface

어플리케이션 소프트웨어를 구축하고 통합하기 위한 프로토콜 세트 🎁

간단히 말해서, 정보 제공자와 정보 사용자 간에 정보를 어떻게 주고 받을지에 대한 것을 담은 계약.🔗

<br/>
<details>
<summary style="color:gray">조금 더 친절한 설명</summary>
<div markdown="1">   

![image](https://user-images.githubusercontent.com/96677719/149865104-b8750d9e-0f39-4577-9506-3ac204e7f57f.png)

사람이 서로 대화하기 위해서는 언어를 사용해야 한다.🗣 


![image](https://user-images.githubusercontent.com/96677719/149865072-111eb520-381e-446f-9967-39b745a44f7a.png)


또 둘 이상이 모인 사회에서 질서가 유지되기 위해서는 규칙이 필요하다. 👩‍👩‍👧‍👦

![image](https://user-images.githubusercontent.com/96677719/149865086-2dd909ba-2f44-4fe3-b296-7fb42c34a691.png)

🌍컴퓨터들이 모여서 소통하는 네트워크에서도 마찬가지로 데이터를 주고 받기 위해서는 같은 언어와 규칙을 공유해야 한다. 이러한 규칙들을 프로토콜이라 부른다. 같은 언어를 사용해야 대화가 가능한 것처럼 같은 프로토콜 조합을 사용해야 소통할 수 있다.

</div>
</details>
<br/>

<details>
<summary style="color:gray">다양한 API를 확인해보자</summary>
<div markdown="1">   

네이버 OpenAPI: https://developers.naver.com/docs/common/openapiguide/apilist.md

API store: 국내 최초 API 유통 플랫폼 https://www.apistore.co.kr//main.do

REST API: https://restfulapi.net/

</div>
</details>
<br/>

컴퓨터 내부에서, 또는 컴퓨터 사이에서 데이터가 교환될 때는 어떤 데이터 형식을 어떤 순서와 절차, 방법을 이용해 교환할 것인지 정해야 한다. 프로토콜은 이에 대해 정의한 규칙들의 집합을 말한다. 간단한 예시를 만들자면 날씨 서비스용 API를 가정해보자면, 해당 API는 사용자가 우편 번호를 제공하며 날씨 정보를 요청(requset)📤	하면 생산자는 날씨 정보로 응답(response)📥하도록 지정할 수 있다. 대표적으로 웹문서를 주고 받을 때는 HTTP 프로토콜을, 파일을 주고 받을 때는 FTP. 메일은 SMTP를 이용하는 등 전송 유형에 따라 다양하게 만들어져 있다. 

이처럼 API를 사용하면 주고 받는 자원의 검색 방법, 자원의 출처에 대한 지식이 없어도 정보를 주고 받을 수 있다. 이를 통해 제품 또는 서비스가 서로 커뮤니케이션할 수 있으며 애플리케이션 개발을 간소화하여 시간과 비용을 절약할 수 있다. 💸



## **2. HTTP API**

**📌 HTTP란?**


🔎 HyperText Transfer Protocol

![image](https://user-images.githubusercontent.com/96677719/149865641-a8bec2ec-3f40-40e8-9405-c1127c4550ac.png)


웹에 내재된 프로토콜로, 인터넷에서 데이터를 주고 받을 수 있는 프로토콜. 웹(Web)은 WWW(World Wide Web)의 줄인 말로, WWW에서는 HyperText 전달 프로토콜에 기초하여 데이터를 주고 받는다. 

![image](https://user-images.githubusercontent.com/96677719/149865876-b3a7f3d6-c98b-4970-8ba9-62bb06a1b84f.png)

HyperText: 문서 내부에 또 다른 문서로 연결되는 참조를 집어 넣음으로써 웹 상에 존재하는 여러 문서끼리 서로 참조할 수 있는 기술. 이때 문서 내부에서 또 다른 문서로 연결되는 참조를 하이퍼링크라고 부른다. 

```
<a href="../signup.html">회원가입 하기</a>
<a href="https://chw-owo.github.io/">조혜원 블로그로 가기</a>
```
=> 웹 페이지: HTML 언어를 사용하여 작성된 ✌하이퍼텍스트 문서✌

=> 웹 사이트: 서로 관련된 내용으로 작성된 웹페이지들의 집합

HyperText 방식으로 텍스트, 그림, 소리, 영상 등의 멀티미디어 정보를 전달하기 위한 규칙이 바로 HTTP이다. 웹 페이지를 방문할 때마다 컴퓨터는 HTTP를 사용하여 인터넷 어딘가에 있는 다른 컴퓨터에서 해당 웹 페이지를 다운로드 한다.

<br/>
<details>
<summary style="color:gray"> 인터넷 != 웹</summary>
<div markdown="1">   

인터넷: ( != 웹 ) 컴퓨터가 서로 연결되어 통신을 주고 받는 컴퓨터끼리의 네트워크를 통칭하는 말. 웹 뿐만 아니라 모바일 앱, 전자메일, 토렌트 등의 파일 공유 시스템, 웹캠, 동영상 스트리밍, 온라인 게임 등이 여기에 포함된다.

웹: WWW(World Wide Web) 인터넷 상에서 동작하는 서비스 중 하나로 인터넷에 연결된 사용자들이 서로의 정보를 공유할 수 있는 공간의 일종. 웹은 HTTP를 기반으로, HyperText 방식으로 텍스트, 그림, 소리, 영상 등의 멀티미디어 정보를 전달한다. 

</div>
</details>
<br/>


## **3. RESTful API**

📌 **REST란?**

🔎 Representational State Transfer

자원의 representation에 의한 상태 전달

![image](https://user-images.githubusercontent.com/96677719/149866039-aba9369d-493a-4804-862b-8efa179d1468.png)


REST API란 HTTP API를 기반으로 하여 REST 아키텍쳐의 제약 조건을 준수하는 웹 API로, 이때 REST은 프로토콜이 아니라 네트워크 아키텍처 원칙, 일종의 가이드라인들을 모아놓은 것을 의미한다. 📋 


http의 주요 저자 중 한 사람인 Roy fielding이 http 설계의 우수성이 제대로 활용되지 못하는 것이 안타까워 웹의 장점을 최대한 활용할 수 있는 REST를 발표했다. 

REST는 다른 일부 API (ex SOAP, XML_RPC)등이 엄격한 프레임워크를 부과하는 것과 달리 REST API는 거의 모든 언어를 사용하여 개발이 가능하며, 다양한 데이터 포맷을 지원한다. 예를 들어 데이터 요청이 http를 통해 REST API로 전송되면 REST용으로 설계된 API, 즉 REST API가 HTML, XML, 텍스트, JSON과 같은 다양한 형식으로 메세지를 반환할 수 있다.   

REST는 가이드 라인이기 때문에 개발하는 서비스의 특징, 환경 등을 고려하여 설계되어야 한다. 이러한 REST API를 제공하는 웹 서비스를 RESTful하다고 한다. 


<br/>
<details>
<summary style="color:gray">네트워크 아키텍쳐란?</summary>
<div markdown="1">   

아키텍처, architecture란 시스템의 전체 구조를 의미한다.


![image](https://user-images.githubusercontent.com/96677719/149722685-da4366fb-0e8d-437a-a645-55ac21358764.png)

출처: https://better-together.tistory.com/65


네트워크는 수많은 컴퓨터들이 연결되어 데이터를 전송하는 복잡한 시스템이다. 이렇게 복잡한 시스템을 프로토콜의 조합으로 단순하게 만든 것을 네트워크 아키텍쳐라 부른다. 

</div>
</details>
<br/>


📌 **REST의 구성**

REST는 resource(자원), Verb(행위), Representation(표현) 세가지로 구성된다. 

참고자료 - 카카오톡 API: https://developers.kakao.com/docs/latest/ko/message/rest-api

자원은 서버에 존재하는 데이터의 총칭으로 모든 자원은 고유의 URI를 가지며 클라이언트는 이 URI를 지정하여 해당 자원에 대해 CRUD 명령을 수행할 수 있다. (ex: https://chw-owo.github.io/recipe/TeokPokki)

"떡볶이에 대한 요리 방법을 책갈피 페이지에 담는다"고 했을 때, POST방식으로 아래와 같은 json을 보내게 된다. 

```
{
    "recipe":{
        "food": "떡볶이"
    }
} 
```

이때 https://chw-owo.github.io/recipe/TeokPokki가 REST의 resource(자원), 책갈피 페이지에 담는 것(POST)가 REST의 verb(행위), 응답으로 보내는 json파일이 REST의 representation(표현)에 해당한다. 

**1) resource(자원)**

REST는 ROA, Resource Oriented Architecture, 리소스 지향 아키텍처로, 모든 것을 리소스 단위로 표현한다. 따라서 REST API는 리소스 단위로 API를 설계하길 권장하며 API에 사용된 리소스는 명사로, 그리고 복수형으로 사용하길 권장한다. 

ex1) gmail.googleapis.com/users/messages/1

ex2) GET /members/delete/1 과 같이 동사 형태가 사용되는 것 보다는 DELETE /members/1과 같이 사용되는 것이 좋다 

+) 그 외에도 가독성을 위해 _ 대신 -를 이용하는 것, 맨 마지막에 /를 넣지 않는 것, 대문자를 쓰지 않을 것 등의 권장사항이 있다. 

이러한 권장사항은 공식 홈페이지에서 더 많이 볼 수 있다. https://restfulapi.net/resource-naming/

**2) Method(행위)**

http에는 다양한 메소드가 존재하나 REST는 그 중 CRUD에 해당하는 4가지 메소드, POST, PUT, UPDATE, DELETE만 사용한다.

![image](https://user-images.githubusercontent.com/96677719/149866583-b045e2fd-e309-47b4-bf12-4248f70b5bec.png)

<br/>
<details>
<summary style="color:gray"> Idempotent </summary>
<div markdown="1">  

메소드를 수행할 때는 해당 메소드가 Idempotent인지 아닌지 주의를 기울여야 한다. Idempotent는 여러번 수행해도 결과가 동일한 성질을 의미한다. POST 메소드를 호출할 때 마다 해당 항목의 count를 증가시키는 경우는 Idempotent하지 못한 것이다. 만약 서로 연관된 여러 API가 동시에 사용될 때 하나의 API 동작이 실패하게 되면 Idepotent하지 않은 API에 대해서는 트랜잭션과 같은 부분을 추가적으로 고민해봐야 한다. (트랜잭션: DB에 서비스 요구시 응답하기 위한 상태 변환 과정의 단위)

</div>
</details>
<br/>


<br/>
<details>
<summary style="color:gray"> + 또 다른 대표적인 API, SOAP </summary>
<div markdown="1">  

![image](https://user-images.githubusercontent.com/96677719/149866740-5b334977-c11a-46ef-9f01-e7426bbea944.png)

SOAP: Simple Object Access Protocol, 데이터 포맷이 xml로 한정되어 있고 엄격한 프레임워크를 가지고 있기 때문에 웹에 최적화된 REST와 달리 브라우저들 간의 호환성에 있어서 약하다. 대신 신뢰성을 보다 잘 보장할 수 있다.

</div>
</details>
<br/>

📌 **제약 조건**


REST API는 HTTP API에 몇가지 제약 조건, 일종의 가이드라인을 붙인 것. 따라서 HTTP에서도 통용되는 사항들이 있다. 

(REST를 만든 Roy fielding는 이걸 다 지키지 않으면 REST API가 될 수 없다고 못박아뒀으나 현실은 그냥 HTTP API면 REST API라고 부르는 경우가 많음. ~~Roy fielding가 싫어합니다~~)

**1) Client-Server 구조 (HTTP와 동일)**

클라이언트, 서버, 리소스로 구성된 ✌클라이언트-서버 구조✌가 필요하다. 

![image](https://user-images.githubusercontent.com/96677719/149789876-0bda9452-8d2a-4da2-871e-dd7dd338d0f1.png)

클라이언트가 서버에 요청을 보내면 서버는 그에 대한 응답을 보내는 구조가 요구된다. 서버는 API를 제공하고 클라이언트는 사용자 인증, context(세션, 로그인 정보) 등을 관리하는 구조로 역할을 구분하여 각자 개발할 내용을 명확히 하고 서로간의 의존성을 줄인다.


**2) Stateless 무상태성 (HTTP와 동일)**

![image](https://user-images.githubusercontent.com/96677719/149789943-42cd9506-bf3a-46a0-9291-c8992e18c01d.png)


서버는 들어온 request만을 단순하게 처리하며 클라이언트의 상태에 대한 정보를 따로 저장하지 않는다. 이를 통해 서버 확장성이 높아지고, 서버에서 불필요한 정보를 관리하지 않음으로써 성능이 개선된다. 그러나 클라이언트가 매번 추가 데이터를 전송해야 하기 때문에 다른 수단 (쿠키, 세션, JWT 등)을 함께 이용한다. 


**3) Cacheable 케시 이용 가능(HTTP와 동일)**

![image](https://user-images.githubusercontent.com/96677719/149866916-05f3d601-b2a3-4fb2-9fd9-c7a1f941316f.png)

클라이언트-서버 간 상호작용을 줄여줄 수 있도록 캐싱이 가능해야 한다. 캐싱은 주어진 리소스의 복사본을 저장하여 임시적으로 갖고 있다가, 해당 리소스에 대한 요청이 들어오면 서버에 있는 원본 대신 제공하는 기술이다. 원래의 서버로부터 다시 다운로드 하는데에 걸리는 시간을 줄여주어 서버 부하를 완화하고 성능이 더 향상된다. rest APIsms http 웹표준을 그대로 사용하기 때문에 http가 가진 캐싱 기능이 적용된다. 

**4) Layered System 계층화된 시스템**

![image](https://user-images.githubusercontent.com/96677719/149722859-c6b1d2b7-acb1-413f-b8ad-c25403e13971.png)

복잡한 네트워크 시스템을 프로토콜의 조합으로 단순화한 것을 네트워크 아키텍쳐라 한다. 조금 더 자세히 설명하자면, 

추상화, abtraction: 복잡한 시스템을 쉽게 이해할 수 있도록 핵심적인 기능들을 뽑아 단순화시키는 방법

모듈화, nodularity: 복잡한 네트워크 시스템이 데이터를 전송하는 과정을 단계로 분할하여 각 단계 마다 핵심적인 기능을 모듈이라는 단위로 만드는 것. 이때 모듈은 다른 모듈과 상관 없이 독립적으로 고유한 기능을 수행하며, 독립적인 모듈들이 인터페이스로 연결되어 하나의 시스템으로 동작하게 된다. 따라서 모듈들을 연결할 수 있는 균일한 인터페이스가 필요하다. 

계층화: 모듈 간에 시간 개념을 부여하여 일정한 순서를 매긴 것을 계층이라 부르며, 이를 배열한 것을 계층화라 한다. 이를 통해 각 계층 간에 일정한 정보의 흐름이 있는 시스템을 설계할 수 있다. 

 이와 같은 계층화를 통해 네트워크를 추상화시킨 것을 네트워크 아키텍쳐라고 한다. 각 계층이 독립적으로 역할을 수행하기 때문에 한 계층에서 오류가 났을 때 해당 계층만 변경하면 되는 등 시스템 설계에 유연성과 개방성을 부여한다. 

**5) self-descriptiveness(자체 표현 구조)** 🛠

REST API 메세지만 보고도 쉽게 이해할 수 있어야 한다. 
구체적으로, 리소스와 메소드를 이용해서 어떤 메소드에 무슨 행위를 하는지 알 수 있으며, 메세지 포맷 역시 json을 이용해서 직관적으로 이해 가능하게 해야한다.


**6) Uniform Interface(균일한 인터페이스)**

인터페이스가 균일하다. 

URL을 쓰는 동시에 한정된 인터페이스로써 사용하여 한정된 자원으로 메세지에 접근할 수 있다. 예를 들자면

https://chw-owo.github.io/admin 라는 하나의 리소스를 두고

GET - 관리자 페이지 조회, 

PUT - 관리자 페이지 수정, 

POST - 관리자 페이지 생성, 

DELETE - 관리자 페이지 삭제 

이와 같이 4가지 인터페이스로 한정지어서 해당하는 리소스에 접근하는 것이다. 이를 통해 다양한 방법으로 설명이 될 수 있는 표현 방식을 갖게 된다. 또, URL 주소 길이가 짧아질 뿐더러 하나의 URL이 많은 표현(Representation)을 나타내게 된다. 

균일한 인터페이스를 위해서는 4가지 제약 조건이 있는데 이 부분이 지켜지지 않으면 REST API라고 할 수 없다(고 Roy field가 말했지만 이 부분을 지키지 않으면서 REST API라고 부르는 경우도 많다.)

ex)

기존의 URL: http://chw-owo.github.io/page?user=guest/menu=login

REST를 적용한 URL: http://chw-owo.github.io/user/login, HTTP Method: POST


📌 **Uniform Interface의 4가지 측면**

REST를 처음 제시한 Roy Fielding은 이를 REST 아키텍쳐 스타일과  다른 네트워크 기반 스타일을 차별화 하는 핵심적 기능이라고 설명했다. 

**1) Resource Identifier (URI)**

리소스는 하나 이상의 유일한 특정 주소인 URI (Uniform Resource Identifier)을 반드시 가져야한다. REST에서는 리소스를 특정지을 수 있는 명사형 Identifier로 URI를 정하도록 권장하고 있으며 웹 상에서 URL (Uniform Resource Locator)를 통해 제한된 메소드와 함께 접근할 수 있다. 

<br/>
<details>
<summary style="color:gray"> URI? URL? </summary>
<div markdown="1">  

![image](https://user-images.githubusercontent.com/96677719/149862624-8b71fc78-e631-4071-852e-e8ec00d9e953.png)

URI: Uniform Resource Identifier, 인터넷에 있는 자원을 나타내는 유일한 주소

URL: Uniform Resource Locator: 파일 식별자, 네트워크 상에서 자원이 어디있는지 알려주기 위한 규약, 웹사이트 주소 뿐 아니라 네트워크 상의 자원을 모두 나타낼 수 있다. 

URN: Uniform Resource Name: urn:scheme를 사용하는 URI의 역사적인 이름, 영속적이고 독립적인 자원을 위한 지시자로 사용하기 위해 정의되었다. 

예를 들어 http://chw-owo.github.io:3000/main?id=HTML&page=12 라고 되어있는 주소가 있다.

여기서 http://chw-owo.github.io:3000/main 여기까지는 URL이고(URI이기도 한)
http://chw-owo.github.io:3000/main?id=HTML&page=12 이 것은 URI라고 할 수 있다. (URL은 아닌)

이유는 URL은 자원의 위치를 나타내 주는 것이고 URI는 자원의 식별자인데
?id=HTML&page=12 이 부분은 위치를 나타내는 것이 아니라 id값이 HTML이고 page가 12인 것을 나타내주는 식별하는 부분이기 때문이다

</div>
</details>
<br/>


**2) Resource Representation** 

![image](https://user-images.githubusercontent.com/96677719/149866230-31c4116d-73b7-4db2-bba0-c199b768d3ba.png)


클라이언트 서버 구조에서 리소스를 직접 주고 받는 것이 아니라, 그 리소스에 대한 상태나 정보를 설명하는 Representation을 주고 받는다. 리소스는 하나 이상의 Representation을 가질 수 있다. Representation의 형태는 content-type으로 결정되며 클라이언트마다 동일 Resource에 대해 다른 Representation을 가질 수 있다. 예를 들자면 https://chw-owo.github.io/hi 라는 같은 uri로 접근했을 때 전송한 요청의 Content-Type에 따라 응답이 달라진다.

```
Content-Type: text/plain
Content-Language: en

hello
```
```
Content-Type: text/plain
Content-Language: ko

안녕하세요
```


Representation Formats

- text/html
- application/xml
- application/json
- application/x-www-form-urlencoded (for input: PUT,POST)
- Others : image(svg,jpg,png,etc.), txt/css, text/javascript

리소스는 Representation에 대한 접근을 위한 urL을 가지고, HTTP 4가지 메소드 (POST, GET, PUT, DELETE)를 통해 생성, 조회, 갱신, 삭제될 수 있다. 

**3) 자기 기술적 (self-descriptive) 메세지** 

잘 지켜지지 않는 부분 1

REST는 Stateless, 상태 정보를 유지하지 않기 때문에 클라이언트-서버 간의 메세지는 자신에 대한 상태 정보를 설명해야 한다. 즉, API 문서가 REST API 응답 본문에 존재해야 한다. API 문서 전체를 응답에 넣는 것은 불가능 하지만 적어도 API 문서가 어디 있는지 알려주어야 한다. 이를 지키면 REST 메세지가 요청 작업을 수행하기 위해 충분한 정보를 별 다른 문서 없이  HTTP 메소드, 상태 코드, 헤더 등을 활용하여 그 자체로 확인할 수 있게 된다. 

ex) 
```
GET / HTTP /1.1
```
self-descriptive 한가? No, 목적지가 빠져있기 때문에 안된다. 
```
=> GET / HTTP /1.1, Host: www.test.co.kr
```
<br/>
<br/>
```
HTTP/1.1 200 OK

[ {"op": "remove", "path": "a/b/c/" }
```
self-descriptive한가 ? No, Client가 해석을 못한다.
```
=> HTTP/1.1 200 OK

Content-Type: application/json-path+json  

[ {"op": "remove", "path": "a/b/c/" } 
```
Content-Type의 application/json-path+json 명세를 통해서 body 내용을 해석 할 수 있기 때문에 이는 self-descriptive하다

**4) HATEOAS** 

잘 지켜지지 않는 부분 2

Hypermedia as the Engine of Application State

![image](https://user-images.githubusercontent.com/96677719/149864135-c0911a82-513f-46bf-b18e-ec9c3c5300be.png)

하이퍼링크 메세지를 통해 어플리케이션의 상태를 전이시켜야 한다.  

리소스를 할당한 후 REST 클라이언트가 하이퍼 링크를 통해 현재 사용 가능한  모든 작업을 찾을 수 있어야 한다. 그러나 많이 지켜지지 않는다. 

ex)

GET https://my-server.com/article 

이 게시글을 조회했을 때 사용자가 다음으로 할 수 있는 행동이 다음 게시물 조회, 게시물을 내 피드에 저장, 댓글 달기 세가지라고 하자. 이러한 각각 행동을 상태라고 부르는데 이것들에 해당하는 링크를 Hypermedia 링크로 reponse 본문에 모두 넣어주어야 한다. 


```
{
  "data": {
    "id": 1000,
    "name": "게시글 1",
    "content": "HAL JSON을 이용한 예시 JSON",
    "self": "http://localhost:8080/api/article/1000", // 현재 api 주소
    "profile": "http://localhost:8080/docs##query-article", // 해당 api의 문서
    "next": "http://localhost:8080/api/article/1001", // 다음 article을 조회하는 URI
    "comment": "http://localhost:8080/api/article/comment", // article의 댓글 달기
    "save": "http://localhost:8080/api/feed/article/1000", // article을 내 피드로 저장
  },
}
```

3, 4번의 항목은 구현하는데 드는 노력 대비 큰 성능의 향상이 없기 때문에 잘 지켜지지 않는다.

## 4. 장점 정리

1) HTTP 프로토콜의 인프라를 그대로 사용하므로 REST API 사용을 위한 별도의 인프라를 구출할 필요가 없다.

2) HTTP 프로토콜의 표준을 최대한 활용하여 여러 추가적인 장점을 함께 가져갈 수 있게 해준다.

3) HTTP 표준 프로토콜에 따르는 모든 플랫폼에서 사용이 가능하다.

4) 서버와 클라이언트의 역할을 명확하게 분리한다.

5) REST API 메시지가 의도하는 바를 명확하게 나타내므로 의도하는 바를 쉽게 파악할 수 있다.


<br/>

## 5. 출처 

프로토콜: https://developer.mozilla.org/ko/docs/Glossary/Protocol

아키텍쳐: https://better-together.tistory.com/65

API: https://digitalbourgeois.tistory.com/52

REST API: https://www.redhat.com/ko, 
https://meetup.toast.com/posts/92, https://tibetsandfox.tistory.com/19

캐시: https://developer.mozilla.org/ko/docs/Web/HTTP/Caching

HTTP: https://hanamon.kr, https://victorydntmd.tistory.com/286, https://joshua1988.github.io/web-development/http-part1/

인터넷과 웹: https://seunghyun90.tistory.com/40, https://all-young.tistory.com/19

Uniform Interface: https://programmer7895.tistory.com/28, https://midnightcow.tistory.com/102, https://doitnow-man.tistory.com/96

URL URI : https://velog.io/@jch9537/URI-URL

representation: https://blog.npcode.com/2017/04/03/rest%EC%9D%98-representation%EC%9D%B4%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80/

<br/>

+)정말 좋았던 두 자료!

https://restfulapi.net

https://www.youtube.com/watch?v=RP_f5dMoHFc
