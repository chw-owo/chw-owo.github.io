---
title: MVC Design Pattern
categories: Web
tags: Web
toc: true
toc_sticky: true
---

## 📌 Design Pattern이란?

디자인 패턴은 건축으로치면 공법에 해당하는 것으로 소프트웨어의 개발 방법을 공식화 한 것이다. 소수의 뛰어난 엔지니어가 해결한 문제를 다수의 엔지니어들이 처리 할 수 있도록 한 규칙이면서, 구현자들 간의 커뮤니케이션의 효율성을 높이는 기법이다.

## 📌 MVC Design Pattern이란?

MVC란 Model View Controller의 약자로 에플리케이션을 세가지의 역할로 구분한 개발 방법론이다.

![image](https://user-images.githubusercontent.com/96677719/152674735-57baf5a5-4777-47b7-a004-00b41be3201e.png)

위의 그림처럼 사용자가 Controller를 조작하면 Controller는 Model을 통해서 데이터를 가져오고 그 정보를 바탕으로 시각적인 표현을 담당하는 View를 제어해서 사용자에게 전달하게 된다. 

## 📌 MVC 각 요소의 역할


Model

어플리케이션이 무엇을 할 것인지 정의한다. 내부 비즈니스 로직을 처리하기 위한 역할을 한다. 즉, 데이터 저장소(ex. DB)와 연동하여 사용자가 입력한 데이터나 사용자에게 출력할 데이터를 다룬다. 특히, 여러 개의 데이터 변경 작업(ex. 추가, 변경, 삭제)를 하나의 작업으로 묶은 트랜잭션을 다루는 일도 한다. 

View

최종 사용자에게 무엇을 화면(UI)로 보여준다. 화면에 무엇을 보여주기 위한 역할을 한다. 즉, 모델이 처리한 데이터나 그 작업 결과를 가지고 사용자에게 출력할 화면을 만든다. 만든 화면은 웹 브라우저가 출력한다.

Controller

Model과 View 사이에 있는 컴포넌트이다. Model이 데이터를 어떻게 처리할지 알려주는 역할을 한다. 클라이언트의 요청을 받으면 해당 요청에 대한 실제 업무를 수행하는 Model을 호출한다. 클라이언트가 보낸 데이터가 있다면, 모델을 호출할 때 전달하기 쉽게 적절히 가공한다. Model이 업무 수행을 완료하면 그 결과를 가지고 화면을 생성하도록 View에 전달한다. 즉, 클라이언트의 요청에 대해 Model과 View를 결정하여 전달하는 일종의 조정자로서의 일을 한다.

Model과 View는 다른 컴포넌트들에 대해 알지 못한다. 자기 자신이 무엇을 수행하는지만 알고 있다. 반면 Controller는 자기 자신 외에 Model과 View가 무엇을 수행하는지 알고 어플리케이션의 흐름을 제어한다. 

## 📌 MVC 패턴의 장단점

MVC 패턴을 사용하면 Model과 View가 다른 컴포넌트들에 종속되지 않아 변경에 유리하다는 장점을 가질 수 있다. 또 디자인 패턴 중 단순한 패턴인 축에 속한다.

그러나 View를 업데이트 하기 위해서는 결국 M-V 사이에 의존성이 생기게 된다. Model과 View 사이의 의존성이 여전히 존재한다고 볼 수 있다. 따라서 어플리케이션이 커지고 복잡해질수록 유지보수가 어려워진다. 

이를 극복하기 위해 Controller 대신 Presenter를 이용하는 MVP 패턴, ViewModel을 이용하는 MVVM 패턴 등이 존재한다. 

## 📌 Web과 MVC

위의 개념을 웹에 적용해보자. 

1. 사용자가 웹사이트에 접속한다. (Uses)
2. Controller는 사용자가 요청한 웹페이지를 서비스 하기 위해서 모델을 호출한다. (Manipulates)
3. 모델은 데이터베이스나 파일과 같은 데이터 소스를 제어한 후에 그 결과를 리턴한다.
4. Controller는 Model이 리턴한 결과를 View에 반영한다. (Updates)
5. 데이터가 반영된 VIew는 사용자에게 보여진다. (Sees)

코드 이그나이터의 문법을 바탕으로 Controller, MVC, View의 관계를 시각화하면 다음과 같다. 

![image](https://user-images.githubusercontent.com/96677719/152674820-99637aa0-dd62-4360-8704-46b583826851.png)


### 출처

https://opentutorials.org/course/697/3828

https://tecoble.techcourse.co.kr/post/2021-04-26-mvc/

https://brunch.co.kr/@oemilk/113