---
title: Java, Collection
categories: Java
tags: 자바
toc: true
toc_sticky: true
---

#### **📌 Collection VS 배열**

배열: 정적인 자료형. 연속된 공간이 필요하고 처음에 크기를 함께 선언해야 한다. 따라서 공간 낭비가 생길 수 있고 데이터를 넣는 것이 힘들지만 요소들에 접근할 때는 빠르게 할 수 있다. 

Collection: 가변적, 동적인 자료형. 빈 공간에 데이터를 넣기 때문에 크기를 정해두지 않고 데이터를 추가할 수 있다. 따라서 데이터를 넣을 때는 편하지만 일렬로 데이터가 들어가있지 않기 때문에 요소들에 접근할 때는 시간이 더 소요된다. 

#### **📌 Collection과 자료형**

Collection은 동적인 자료형이기 때문에 참조 자료형만 넣을 수 있다. 즉 int boolean char 등 값을 가지는 기본 자료형은 들어갈 수 없고 String, 커스텀 자료형 등 주소를 가지는 레퍼런스 자료형만 쓴다. 이를 보완하기 위해 Integer, Boolean Character 등 기본 자료형에 해당하는 래핑 클래스가 존재한다. 

#### **📌 Collection의 Interface**

![image](https://user-images.githubusercontent.com/96677719/150907794-8feb32cd-737a-4d6f-9fea-f5bdb52163e9.png)

기본 인터페이스로는 List, Set, Map이 있다. 

List: 중복을 허용하고 순서가 유지되는 데이터 집합 ArrayList, LinkedList, Vector, Stack

Set: 중복을 허용하지 않는 집합으로 순서가 유지되지 않는다,  HashSet, TreeSet

Map:key-value 쌍으로 이루어진 데이터의 집합으로 순서가 유지되지 않는다. HashMap, TreeMap, HashTable, Properties

출처:

https://gangnam-americano.tistory.com/41

최주호 데어프로그래밍 자바
