---
layout: post
title: Java, Object와 Generic
date: 2022-01-20 19:00
categories: Java
tags: 자바
toc: true
toc_sticky: true
---

📌 Object Class란?

Object Class는 자바의 기본적인 클래스들 중에서 가장 많이 사용되는, 모든 자바 클래스의 조상격인 클래스이다. 모든 클래스에 extends Object가 생략되어 있는셈

```java

class Animal{   
    String name;
    Animal(String name){
        this.name = name;
    } 
}
class Dog extends Animal{
    String color;
    Dog(String name, String color){
        this.name = name;
        this.color = color;
    }    
}

public class Main {
    public static void main(String[] args) {

        Dog d1 = new Dog("John"); (o)
        Animal d2 = new Dog("Peter"); (o)
        Object d3 = new Dog("Wendy"); (o)
    }
}

```
자바는 다형성을 띄기 때문에 이와 같이 표현할 수 있다. 단, 오브젝트로 선언되었기 때문에 이를 다시 Dog 자료형에 넣으려면 Dog dogJohn = (Dog)John 처럼 Down casting이 필요하다.

📌 Object Class와 배열

```java

class Dog extends Animal{
    String name;
    Dog(String name){
        this.name = name;
    }    
}

class Cat extends Animal{
    String name;
    Cat(String name){
        this.name = name;
    }    
} 

public class Main {
    public static void main(String[] args) {

        Object[] obj = new Object[2];
        obj[0] = new Dog("John");
        obj[1] = new Cat("Peter"); 

        Dog d1 = (Dog)obj[0];
        Cat c1 = (Dog)obj[1];

        System.out.println(d1.name);
        System.out.println(c1.name);
    }
}

```
Object Class로 배열을 만들면 다양한 타입의 객체를 한 배열에 담을 수 있다. 단, 해당 객체의 상태에 접근할 때 다운캐스팅을 해야한다. 

📌 Generic이란?

위에서 여러 데이터 타입을 묶을 때 Object class를 사용했다. 이때, 제네릭을 사용하면 다운캐스팅 없이도 여러 데이터 타입을 사용할 수 있다. 

```java
class AnimalHotel<T>{
    T data;
}

...

public class Main {
    public static void main(String[] args) {

        AnimalHotel<Dog> d1 = new AnimalHotel<>();
        d1.data = new Dog("John");

        AnimalHotel<Cat> c1 = new AnimalHotel<>();
        c1.data = new Cat("Peter"); 

        System.out.println(d1.data.name);
        System.out.println(c1.data.name);
    }
}

```
