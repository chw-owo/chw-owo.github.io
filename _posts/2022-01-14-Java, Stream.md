---
layout: post
title: Java, Stream
date: 2022-01-15 14:00
categories: Java
tags: 자바
toc: true
toc_sticky: true
---
배열의 원소를 가공할 때 다음과 같이 사용할 수 있다.

1. stream.map()

예시
```java
stream(target_array).map(s->s.toUpperCase() 혹은 String::toUpperCase)
```

요소들을 특정 조건(ex: toUpperCase)에 해당하는 값으로 변환해줌 

```java
System.out.println(list.stream().map(s->s.toUpperCase()).collect(Collectors.toList()));
```
collect를 이용해 결과를 바로 리턴받을 수 있고

```java
list.stream().map(String::toUpperCase).forEach(s -> System.out.println(s));
forEach를 이용해 결과를 print할 수 있다.
```

2. stream.filter()

예시
```java
list.stream().filter(t->t.length()>5)
```
요소를 특정 조건에 따라 걸러냄, 리턴, print 할 때는 위와 동일

3. stream.filter()

```java
list.stream().sorted()
```
ascii 코드 순으로 요소를 정렬
![ascii_code](https://user-images.githubusercontent.com/96677719/149602458-b62971bd-2cd2-4de4-8bb5-3daeb159d7be.png)


출처: https://dpdpwl.tistory.com/81
더 많은 메소드를 보고 싶을 떄 참고할 자료: https://futurecreator.github.io/2018/08/26/java-8-streams/