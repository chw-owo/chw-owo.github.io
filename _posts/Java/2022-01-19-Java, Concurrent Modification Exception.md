---
layout: post
title: Java, Concurrent Modification Exception
date: 2022-01-19 19:00
categories: Java
tags: Java
toc: true
toc_sticky: true
---


알고리즘 문제(링크)를 풀던 중 ConcurrentModificationException이 발생했다. ConcurrentModificationException은 리스트, 셋, 맵 등을 for문에 직접 넣고 돌리며 remove를 호출할 때 발생하는 에러다. element의 index가 실시간으로 변할 때, 즉 loop가 일어나던 도중 length가 변경되어서 해당 인덱스 값이 null이 되면 발생한다. 이를 해결하기 위한 몇가지 방법이 있다. 

📌

## Iterator을 사용하여 순회할 때마다 iterator.remove를 해주는 방법

before

```java
public class Test() {
    private ArrayList<A> abc = new ArrayList<A>();

    public void doStuff() {
        for (A a : abc) 
        a.doSomething();
    }

    public void removeA(A a) {
        abc.remove(a);
    }
}
```

after

```java
List<String> list = new ArrayList<String>();
...
for (Iterator<String> iterator = list.iterator(); iterator.hasNext(); ) {
    String value = iterator.next();
    if (value.length() > 5) {
        iterator.remove();
    }
}
```

이때 remove() 메서드보다 next() 메서드가 먼저 호출되어야 에러가 발생하지 않는다. 더 효율적이고 쓸데없는 코드들이 없으므로 이 방법을 주로 권장한다. 

📌

## 삭제해야 할 리스트(여기서 remove)를 따로 만들고 그 리스트(remove)를 for문에 넣어서 기존 list를 제거하는 방식

```java
ArrayList<String> list = new ArrayList<String>();
~ ~ ~
            
ArrayList<String> Remove = new ArrayList<String>();
for (String name : list) 
    Remove.add(name);
for (String name : Remove) 
    list.remove(name);

```

📌

## 원칙적으로 for문 안의 자료가 remove되는 걸 막는 클래스를 사용하는 방법

```java
List<Integer> list = new CopyOnWriteArrayList<>();
list.add(1);
list.add(2);
list.add(3);

for(Integer i : list) {
System.out.println("값:" + i);
if( i == 2) {
list.remove(i);
    }
}

```
예시로 CopyOnWriteArrayList나 내가 사용한 ConcurrentHashMap 등이 있다. 콜렉션에 수정 작업이 일어나는 경우 자동으로 새 콜렉션을 만들어 수정하고 자동으로 동기화 해주는 방식이다. 위처럼 반복문 내부에서 add를 할 경우 내부적으로 복사된 새 arraylist에 더해지므로 에러가 발생하지 않는다. 기존 코드에서 수정하기 간편하여 사용하였으나, 내부적으로는 for문 안의 값을 수정할 때마다 같은 크기의 콜렉션이 새로 만들어지는 방식이기 때문에 성능이 좋지 않다. 

출처: https://jy2694.tistory.com/16 

https://imasoftwareengineer.tistory.com/85

https://aljjabaegi.tistory.com/533

https://stackoverflow.com/questions/8104692/how-to-avoid-java-util-concurrentmodificationexception-when-iterating-through-an
