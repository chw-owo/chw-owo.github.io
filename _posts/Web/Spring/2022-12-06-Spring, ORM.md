---
title: Spring, ORM
categories: Spring
tags: Spring
toc: true
toc_sticky: true
---

## 📌 ORM이란?

![image](https://user-images.githubusercontent.com/96677719/152674460-7eb6e086-0fdd-4be5-8bc8-9ac6bedb8473.png)


Object Relational Mapping, 관계- 객체 매핑

객체와 관계형 데이터베이스의 데이터를 자동으로 매핑(연결)해주는 것. 객체 지향 프로그래밍은 클래스를 사용하고, 관계형 데이터베이스는 테이블을 사용하기 때문에
객체 모델과 관계형 모델 간에 불일치가 존재한다. 이때, ORM을 통해 객체 간의 관계를 바탕으로 SQL을 자동으로 생성하여 불일치를 해결한다.

## 📌 ORM의 장점

객체지향적인 코드로 인해 더 직관적이고 로직에 집중할 수 있도록 도와준다.

- SQL문이 아닌 클래스의 메서드를 통해 데이터베이스를 조작할 수 있으므로 개발자가 객체 모델만 이용해서 프로그래밍을 하는 데 집중할 수 있다.

- 선언문, 할당, 종료 같은 부수적인 코드가 없거나 줄어든다.

- 객체마다 코드를 별도로 작성하기 때문에 코드의 가독성이 높아진다.

- SQL의 절차적이고 순차적인 접근이 아닌 객체지향적인 접근으로 인해 생산성을 높여준다.

재사용 및 유지보수의 편리성이 증가한다.

- ORM은 독립적으로 작성되어있고, 해당 객체들을 재활용할 수 있다.

- 매핑 정보가 명확하여, ERD를 보는 것에 대한 의존도를 낮출 수 있다.

DBMS에 대한 종속성이 줄어든다.

- 대부분 ORM 솔루션은 DB에 종속적이지 않기 때문에 구현 방법뿐만 아니라 많은 솔루션에서 자료형 타입까지 유효하다.

- 프로그래머는 Object에 집중함으로 극단적으로 DBMS를 교체하는 거대한 작업에도 비교적 적은 리스크와 시간이 소요된다.

- 자바에서 가공할 경우 equals, hashCode의 오버라이드 같은 자바의 기능을 이용할 수 있고, 간결하고 빠른 가공이 가능하다.


## 📌 ORM의 단점

완벽한 ORM 으로만 서비스를 구현하기가 어렵다.

- 사용하기는 편하지만 설계는 매우 신중하게 해야한다.

- 프로젝트의 복잡성이 커질경우 난이도 또한 올라갈 수 있다.

- 잘못 구현된 경우에 속도 저하 및 심각할 경우 일관성이 무너지는 문제점이 생길 수 있다.

- 일부 자주 사용되는 대형 쿼리는 속도를 위해 SP를 쓰는등 별도의 튜닝이 필요한 경우가 있다.

- DBMS의 고유 기능을 이용하기 어렵다. (하지만 이건 단점으로만 볼 수 없다 : 특정 DBMS의 고유기능을 이용하면 이식성이 저하된다.)

프로시저가 많은 시스템에선 ORM의 객체 지향적인 장점을 활용하기 어렵다.

- 이미 프로시저가 많은 시스템에선 다시 객체로 바꿔야하며, 그 과정에서 생산성 저하나 리스크가 많이 발생할 수 있다.


## 📌 ORM 예시

Java

```java
public class Person{
    private String name;
    private String height;
    private String weight;
    private String ssn;
    //implement getter & setter methods
}
```
iBatis

```xml
<select id="getPerson" resultClass="net.agilejava.person.domain.Person">
    SELECT name, height, weight, ssn FROM USER WHERE name = #name#;
</select>
```

해당 쿼리의 결과를 받을 객체를 지정해줄 수 있다. 즉, getPerson이라고 정의된 쿼리 결과는 net.agilejava.person.domain의 Person 객체에 자동으로 매핑되는 것이다.

Hibernate

JPA(Java Persistence API)는 자바의 ORM 기술 표준으로 인터페이스의 모음이다. 이러한 JPA 표준 명세를 구현한 구현체가 바로 Hibernate이다.

```xml
<hibernate-mapping>
    <class name="net.agilejava.person.domain.Person" table="person">
        <id name="name" column="name"/>
        <property name="height" column="height"/>
        <property name="weight" column="weight"/>
        <property name="ssn" column="ssn"/>
    <class>
</hibernate-mapping>
```

위 두개의 Framework의 예시를 보면 알 수 있듯이, setter 메소드가 있으면 객체에 결과를 set하는 작업을 따로 해주지 않아도 자동으로 해당 값이 할당된다. 물론 여기에 1:m 이나 m:1등의 관계들이 형성되면 추가적인 작업이 필요하긴 하지만, 일단 눈에 보이는 간단한 부분은 처리가 되는 것을 볼 수 있다. 물론 반대의 경우에도 객체를 던져주면 ORM Framework에서 알아서 get을 수행해 해당하는 column에 넣어주게 된다.


### 출처

https://eun-jeong.tistory.com/31

https://gmlwjd9405.github.io/2019/02/01/orm.html