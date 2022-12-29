---
title: Spring, DI, IoC, Bean
categories: Spring
tags: Spring
toc: true
toc_sticky: true
---

## 📌 DI, IoC, Bean란?

뜻을 풀어쓰면 다음과 같다

DI = Dependency Injection, 의존성 주입
IoC = Inversion of Controll, 제어의 역전
Bean = 스프링 프레임워크가 관리해주는 객체

단어 풀이만으로는 무슨 뜻인지 감이 안잡히니 차근차근 다시 보자

## 📌 강한 결합

우선 DI, IoC 의 필요성을 알기 위해 강한 결합이 무엇인지 먼저 살펴봐야 한다.

```java
// 1. Controller1이 Service1 객체를 생성하여 사용
public class Controller1{
    private final Service1 service1;

    public Controller1(){
        this.service1 = new Service1();
    }
}
// 2. Service1이 Repository1 객체를 생성하여 사용
public class Service1 {
    private final Repository1 repository1;

    public Service1 (){
        this.repository1 = new Repository1();
    }
}
//3. Repository1 객체 선언
public class Repository1{...}
```

위의 예시는 상한 결합에 해당한다. 만약 이러한 상황에서 Repository1 객체 생성 시 DB 접속 id, pw 를 받아서 DB 접속 시에 사용하도록 변경이 된다면 아래와 같이 생성자에 DB 접속 id, pw를 추가해야 할 것이다.

```java
public class Repository1{
    public Repository1(String id, String pw{
        Connection connection = DriverManager.getConnection("jdbc:h2:mem:springcoredbb:, id, pw);
        })
    }
}
```
이때 문제가 하나 생긴다. Controller 객체를 5개 생성할 경우 그 각각의 객체가 Service1 객체를 각각 하나씩 생성하여 사용하게 되는데 Repository1 생성자 변경에 의해 모든 Controller와 모든 Service의 코드를 변경해야 하게 된다. 

각 객체에 대한 객체 생성은 딱 1번만 이루어지고 이렇게 생성된 객체를 모든 곳에서 재사용하도록 처리하는 방법을 통해 이러한 문제를 해결할 수 있다. 이러한 방법론에 따라 수정된 코드는 다음과 같다.

```java
public class Repository1{...}

//1. Repository1 객체 생성
Repository1 repository1 = new Repository1();
```
```java
// 2. Service1이 Repository1 객체를 생성하여 사용
Class Service1 {
    private final Repository1 repository1;

    public Service1 (Reposotory1 repository1){
        //this.repository1 = new Repository1();
        this.repository1 = repository1;
    }
}

Service service1 = new Service1(repository1);
```
새로운 repo 객체를 생성하는 대신 기존 repo1 객체를 재사용 한 뒤 그것을 바탕으로 Service1 객체를 생성한다.

```java
public class Controller1{
    private final Service1 service1;

    public Controller1(Service1 service1){
        this.service1 = service1;
    }
}
```
Controller1 객체 역시 같은 방법으로 생성해준다.

이렇게 변경된다면 생성자에 DB 접속 id, pw를 추가하여도 Repo1 객체 생성시에만 코드를 다음처럼 수정해주면 된다.

```java
public class Repository1{
    public Repository1(String id, String pw{
        Connection connection = DriverManager.getConnection("jdbc:h2:mem:springcoredbb:, id, pw);
        })
    }
}
String id = "anonymous";
String pw = "";
Repository1 reposituory1 = new Repository1(id, pw);
```
이러한 방식을 느슨한 결합이라고 표현한다. 이러한 느슨한 결합을 통해 코드를 보다 효율적으로 구현할 수 있다.

## 📌 IoC, Inversion of Controll, 제어의 역전

이때, 강한 결합과 느슨한 결합이 진행되는 흐름을 비교해보면, Controller1 => Service1 => Repository1 순서로 객체를 생성하며 접근하는 강한 결합과 달리 느슨한 결합에서는 Repository1을 먼저 구현한 뒤 이를 Service, Controller가 순서로 가져다가 사용하는 것을 볼 수 있다. 이처럼 프로그램의 제어 흐름이 뒤바뀐 것을 제어의 역전, Inversion of Controll, IoC이라고 한다.

## 📌 DI, Dependency Injection, 의존성 주입

위의 예시처럼 용도에 맞게 필요한 객체를 가져다가 사용하는 것을 의존성 주입, Dependency Injection, DI라고 부른다. 이렇게 구현되는 경우 사용할 객체가 어떻게 만들어졌는지 알 필요 없이 가져다가 쓰면 되기 때문에 기존 객체가 변경되어도 코드를 크게 수정하지 않아도 된다. 

## 📌 Bean과 스프링 IoC 컨테이너

DI를 사용하려면 객체 생성이 우선 되어야 한다. 그리고 이러한 객체를 생성, 관리하는 역할을 스프링 프레임워크가 대신 해준다. 이때 스프링이 관리하는 객체를 Bean, 이러한 Bean을 모아두는 공간을 스프링 IoC 컨테이너라고 부른다. 

### Bean 등록하기

Bean을 등록하기 위한 방법에는 @Component를 이용하는 것과 @Bean을 이용하는 것이 있다. 클래스 선언 위에 @Component를 선언하면, 스프링 서버가 뜰 때 스프링 IoC 컨테이너에 해당 클래스의 객체, Bean을 저장해준다. 반면 @Bean을 사용하면 직접 객체를 생성, 빈으로 등록을 요청할 수 있다. 이 역시 마찬가지로 스프링 서버가 뜰 때 스프링 IoC 컨테이터에 빈이 저장된다. 

### Bean 사용하기

멤버 변수 선언 위에 @Autowired를 붙일 경우 스프링에 의해 DI가 이루어진다. 단 Autowired는 스프링 IoC 컨테이너에 의해 관리되는 클래스에 한해 적용 가능하다. 그리고 생성자 선언이 1개일 때는 생략이 가능하다. 


### 출처

스파르타코딩클럽 Spring 심화반 강의
인프런 최주호 스프링부트 개념 정리 강의