---
layout: post
title: Spring, Controller, Service, Repository
date: 2022-01-24 19:00
categories: Java
tags: Java
toc: true
toc_sticky: true
---

![image](https://user-images.githubusercontent.com/96677719/150688504-7f6d3050-6d0f-40aa-9e91-88a4485f044a.png)


![image](https://user-images.githubusercontent.com/96677719/150688464-f84d46d3-7443-4c0e-bc75-d1aa02943631.png)

![image](https://user-images.githubusercontent.com/96677719/150688446-3e2afb0e-2e49-41cc-a2d0-994d5cc3ea9d.png)


###### 📌Controller
주소를 찾아서 사용자의 요청에 맞게 데이터를 전달한다. 더 자세히 말하자면, 사용자의 request를 처리한 후 지정된 뷰(html, jsp 등 화면에 띄우는 파일)에 모델 객체(Object, Dto 사용자로부터 받은 데이터)를 넘겨주는 역할을 한다. 

###### 📌Service
회원 가입, 조회 등 비즈니스단의 기능들 혹은 중복 검사 등이 수행된다. DB에서 유저에게, 혹은 그 반대로 데이터를 가공하여 전달하는 역할을 한다. repository를 거치기 때문에 직접 DB에 접근하지는 않는다.

###### 📌Repository
저장소라는 의미로 DB에 접근하는 역할을 맡는다. 일종의 인터페이스로, JpaRepository를 상속 받으면 기본적인 CRUD가 자동으로 생성되어 이를 쉽게 처리할 수 있다. 

###### 📌출처

https://snepbnt.tistory.com/312

https://it-recording.tistory.com/31?category=971893
