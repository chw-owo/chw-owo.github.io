---
layout: post
title: Java, Object์ Generic
date: 2022-01-20 19:00
categories: Java
tags: ์๋ฐ
toc: true
toc_sticky: true
---

๐ Object Class๋?

Object Class๋ ์๋ฐ์ ๊ธฐ๋ณธ์ ์ธ ํด๋์ค๋ค ์ค์์ ๊ฐ์ฅ ๋ง์ด ์ฌ์ฉ๋๋, ๋ชจ๋  ์๋ฐ ํด๋์ค์ ์กฐ์๊ฒฉ์ธ ํด๋์ค์ด๋ค. ๋ชจ๋  ํด๋์ค์ extends Object๊ฐ ์๋ต๋์ด ์๋์

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
์๋ฐ๋ ๋คํ์ฑ์ ๋๊ธฐ ๋๋ฌธ์ ์ด์ ๊ฐ์ด ํํํ  ์ ์๋ค. ๋จ, ์ค๋ธ์ ํธ๋ก ์ ์ธ๋์๊ธฐ ๋๋ฌธ์ ์ด๋ฅผ ๋ค์ Dog ์๋ฃํ์ ๋ฃ์ผ๋ ค๋ฉด Dog dogJohn = (Dog)John ์ฒ๋ผ Down casting์ด ํ์ํ๋ค.

๐ Object Class์ ๋ฐฐ์ด

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
Object Class๋ก ๋ฐฐ์ด์ ๋ง๋ค๋ฉด ๋ค์ํ ํ์์ ๊ฐ์ฒด๋ฅผ ํ ๋ฐฐ์ด์ ๋ด์ ์ ์๋ค. ๋จ, ํด๋น ๊ฐ์ฒด์ ์ํ์ ์ ๊ทผํ  ๋ ๋ค์ด์บ์คํ์ ํด์ผํ๋ค. 

๐ Generic์ด๋?

์์์ ์ฌ๋ฌ ๋ฐ์ดํฐ ํ์์ ๋ฌถ์ ๋ Object class๋ฅผ ์ฌ์ฉํ๋ค. ์ด๋, ์ ๋ค๋ฆญ์ ์ฌ์ฉํ๋ฉด ๋ค์ด์บ์คํ ์์ด๋ ์ฌ๋ฌ ๋ฐ์ดํฐ ํ์์ ์ฌ์ฉํ  ์ ์๋ค. 

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
