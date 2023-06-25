---
title: Procademy, Q1, 23 - 1) C++ - string, cast
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. String**

C++은 문자열을 하나의 객체로 만들어주는 String 클래스를 표준으로 제공한다. String을 사용할 경우 보다 편하게 문자열을 다룰 수 있다는 장점이 있으나, 내부적으로 버퍼를 할당, 확장하는 것을 자동으로 처리하기 때문에 성능상 저하가 생길 수 있다. 또, 문자열 길이에 대한 개념이 없어져서 실수하게 될 가능성이 높아진다. 요즘은 실무에서도 String을 많이 사용하지만, 위와 같은 이유로 기존의 char 사용법도 숙지하고 있어야 한다. 

<br/>

# **2. Cast**

## **1) C++의 cast**

C에서의 casting은 컴파일 에러를 피하는 것 외에는 아무 기능이 없는 반면 C++에서는 실수를 막기 위한 다양한 종류의 캐스팅 문법을 제공한다. 내부적으로 비교, 분기가 포함되기 때문에 약간의 성능 상 저하가 생길 순 있으나 적절히 사용할 경우 보다 안전한 작업이 가능해진다. 

## **2) static_cast**

```c++
class Car;
class Truck: public Car;

Car* c1 = new Car();
Truck* t1 = (Truck* ) c1;
```
위 캐스팅은 실수가 분명하지만 컴파일러에서 에러를 내지 않는다. 이후 t1을 이용해서 Truck에만 있는 멤버에 접근할 경우 잘못된 메모리 접근으로 큰 문제가 생길 수 있다. 

```c++
Car* c1 = new Car();
Truck* t1 = static_cast<Truck*>(c1);
```
static_cast를 사용하면 위와 같은 상황을 감지하여 컴파일 에러가 발생시킨다. 

```c++
Truck* t1 = new Truck();
Car* c1 = static_cast<Car*>(t1);
```
반대로 위와 같은 상황은 잘못된 메모리 접근으로 이어질 여지가 없기 때문에 에러 없이 통과시킨다. 

## **2) dynamic_cast**

```c++
Car* c1 = new Car();
Truck* t1 = dynamic_cast<Truck*>(c1);
```

dynamic_cast를 사용하면 런타임 중에 타입을 확인하여 잘못된 캐스팅일 경우 null을 반환한다. dynamic_cast이어도 int를 class로 캐스팅 하는 것과 같이 상속관계가 아닌 상태에서의 캐스팅은 컴파일 에러를 낸다. static_cast이 컴파일 시에, dynamic_cast는 런타임 시에 에러를 탐지한다. 

static_cast와 dynamic_cast 둘 다 상속 관계에서 실수를 탐지할 때 유용하다. dynamic_cast의 단점은 탐지를 위해 RTTI(런타임 형식 정보)가 계속 돌아가기 때문에 다소 느려지게 된다. 어셈블리를 확인해보면 의심 가는 문장만 대상으로 하여 타입 확인 명령어가 추가되는 것을 볼 수 있다. 

## **4) reinterpret_cast**

올바르지 않은 캐스팅처럼 보이더라도, 의도한 것이니 에러나지 말라는 의미로 C 스타일의 cast와 동일한 역할을 한다.

## **5) const_cast**

const 특성을 제거하는 cast이다.