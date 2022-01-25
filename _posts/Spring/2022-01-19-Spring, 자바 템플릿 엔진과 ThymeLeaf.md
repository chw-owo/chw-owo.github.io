---
layout: post
title: Spring, 자바 템플릿 엔진과 ThymeLeaf
date: 2022-01-25 19:00
categories: Spring
tags: Spring
toc: true
toc_sticky: true
---

# 📌웹 템플릿 엔진이란?

템플릿 양식과 특정 데이터 모델에 따른 입력 자료를 합성하여 결과 문서를 출력하는 소프트웨어를 말한다. 그 중 웹 템플릿 엔진은 웹 문서가 출력되는 템플릿 엔진으로 웹 템플릿과 웹 컨텐츠 정보를 처리하기 위해 설계된 소프트웨어이다. 이는 view code인 html과 data logic code인 db connection을 분리해주는 역할을 한다. 

# 📌템플릿 엔진의 종류

서버 사이드 템플릿 엔진 

서버에서 구동되는 템플릿 엔진으로 JSP, thymeleaf 등이 있다. 서버에서 가져온 데이터를 미리 정의된 템플릿에 넣어 html을 그린 뒤 클라이언트에게 전달한다. html에서 고정되어있는 부분은 템플릿으로 만들고 동적으로 생성되는 영역만 템플릿에 소스 코드를 끼워넣는 방식으로 동작한다. 

클라이언트 사이드 템플릿 엔진

html 형태로 코드를 작성하며 동적으로 DOM을 그리게 해주는 역할을 한다. 브라우저 위에서 작동하며 react, Vue.js 등이 있다. 서버에서는 Json Xml의 데이터만 전달하고 서버에서 벗어난 상태에서 화면을 구성한다 

# 📌ThymeLeaf

스프링부트가 자동 설정을 지원하는 템플릿 엔진. 서버사이드 자바 템플릿 엔진의 한 종류. JSP와 달리 Servlet Code로 변환되지 않기 때문에 View와  비즈니스 로직을 보다 잘 분리할 수 있다. 타임리프의 주 목표는 유지관리가 쉬운 템플릿 생성 방법을 제공하는 것이며, 실제로 템플릿에 영향을 주지 않는(기존 HTML 코드를 변경하지 않고 덧붙이는) 방식을 사용한다.

프로젝트를 하던 중 RestController - ModelAndView 방식을 이용했는데,html에서 model을 쓰려다보니 Thymeleaf가 필요하더라.

# 📌ThymeLeaf 표현식

더 구체적인 표현식은 여기서 확인 가능

https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html

### 간단한 표현

변수(Variable) 표현식                     : ${...}

선택 변수(Selection Variable) 표현식 : *{...}

메세지(Message) 표현                    : #{...}

링크 URL(Link URL) 표현식              : @{...}

단편(Fragment) 표현식                   : ~{...}
 
### 리터럴

텍스트 리터럴 : 'one text', 'Another one!', ...

숫자 리터럴 : 0, 34, 3.0, 12.3, ...

부울 리터럴 : true,false

널 리터럴 : null

리터럴 토큰 : one, sometext, main, ...


### 텍스트 작업

문자열 연결 : +

리터럴 대체 : |The name is ${name}|

'|' 을 사용 (shift + \)
 
### 산술 연산

이진 연산자 : +, -, *, /,%

빼기 기호 (단항 연산자) : -
 
### Bool 연산

이항 연산자 : and,or

부울 부정 (단항 연산자) : !,not

### 비교 및 평등

비교기 : >, <, >=, <=( gt, lt, ge, le)

같음 연산자 : ==, !=( eq, ne)
 
### 조건부 연산자

if-then : (if) ? (then)

if-then-else : (if) ? (then) : (else)

Default : (value) ?: (defaultvalue)

# 📌JSP가 권장되지 않는 이유

JSP를 사용하고 JAR 패키징을 하는 경우 JSP가 실행되지 않는다. 따라서 WAR 패키징을 해야만 하는데 WAR 패키징엔 여러가지 문제점이 존재한다. 

JAR

Java Archive, 자바 프로젝트를 압축한 파일로 리소스, 속성 파일, 라이브러리 등이 포함되어있다. Path 정보를 유지한 상태로 압출하며 원하는 구조로 구성이 가능하다. JRE만으로도 실행할 수 있다.

WAR

Web Application Archive, servletm hsp 컨테이너에 배치할 수 있는 웹 어플리케이션 압축 파일로 웹 관련 자원만을 포함한다. JAR과 달리 WEB-INF, META-INF 디렉토리로 사전 정의된 구조가 필요하며 실행하기 위해서도 Toncat과 같은 웹 서버, 웹 컨테이너가 따로 필요하다. JAR보다 더 넓은 압축 범위를 가지고 있지만 스프링으로 만든 프로젝트를 압축할 경우 JAR이 권장된다. 

### 출처

https://dev-gorany.tistory.com/302

https://velog.io/@dsunni/Spring-Boot-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%9B%B9-MVC-Thymeleaf