---
title: Java, HashMap
categories: Java
tags: 자료구조, Java
toc: true
toc_sticky: true
---

## 1. Hash function Algorithm 

임의의 길이의 입력 데이터를 정해진 길이의 데이터로 매핑하는 함수. 결과값으로는 기존 입력 데이터를 알 수 없다. 매핑 과정을 Hash라고 부르며 이때 입력 데이터는 Key, 결과값은 Value가 된다. 일관된 규칙으로 변환하는 것이 아니기 때문에 출력값은 랜덤이며 따라서 동일한 값이 나올 가능성 역시 존재한다. 이러한 경우를 해시 충돌이라고 부른다.


## 2. HashTable

#### HashTable이란?

HashTable은 해쉬함수를 이용하여 key를 Hash값으로 매핑하고 이 Hash값을 index 삼아서 value를 key와 함께 저장하는 자료구조이다. 데이터가 저장되는 곳을 버킷, 또는 슬롯이라고 부르며 삽입, 삭제, 탐색이 가능하다. 

value: 저장소인 버킷, 슬롯에 최종적으로 저장되는 값으로 hash와 매칭되어 저장된다. 

key: 고유한 값. 하나의 hashTable에는 중복된 key가 들어갈 수 없다.  키 값을 저장할 경우 키의 길이가 다양하면 메모리 낭비가 생기므로 Hash function을 이용해 정해진 길이의 데이터로 매핑한다. 이처럼 key는 hash function의 input으로 들어가 고정된 길이의 hash로 반환된다.

#### HashTable의 장점

key - value가 1:1로 매핑되므로 삽입, 삭제, 검색이 이루어질 때 평균 O(1)의 시간 복잡도를 가지게 된다.

#### HashTable의 단점

Hash 충돌의 가능성이 있다.

데이터의 순서가 보장되지 않는다.

hash function이 복잡하다면 연산 시간이 증가된다.

데이터가 저장되기 전에 미리 저장공간을 확보하므로 공간 효율성이 떨어진다.


## 3. HashMap 


Java 2에서 추가된 Java Collection Framework의 API. HashTable과 같은 기능을 제공하나 몇가지 차이점이 있다.

HashMap은 보조 해쉬 함수를 사용하여 해쉬 충돌이 일어날 가능성을 줄였다.

HashMap은 동기화를 지원하지 않아서 HashTable에 비해 조금 더 빠르다. 동기화가 필요한 멀티스레드 환경에서도 프로그래밍 상의 편의성 때문에 Map m = Collections.synchronizedMap(new HashMap(...)); 와 같은 형태를 많이 쓴다.

#### HashMap의 주요 메소드

boolean containsKey(Object key)	지정된 key가 포함되어 있는지 여부를 반환한다.  

boolean containsValue(Object value)	지정된 value가 포함되어 있는지 여부를 반환한다.

Set entrySet()	저장된 키와 값을 엔트리(키와 값의 결합)의 형태로 Set에 저장하여 반환한다. 

Set keySet()	저장된 모든 key를 Set에 저장하여 반환한다. 

void clear()	저장된 모든 객체(key, value)를 제거한다. 

Object remove(Object key)	지정된 key에 해당하는 value를 제거한다. 

Object getOrDefault(Object key, Object defaultValue)	지정된 키의 값을 반환한다. 키가 없을 경우, default Value로 지정된 데이터를 반환한다. 

void putAll(Map map)	Map에 저장된 모든 요소를 HashMap에 저장한다. 

Object replace(Object key, Object value)	지정된 키의 값을 지정된 value로 대체한다. 

boolean replace(Object key, Object oldValue, Object newValue)	지정된 키와 값(oldValue)가 모두 일치하는 경우에만 새로운 값으로 대체하며, 일치 여부를 반환한다. 

#### HashMap에 반복문 적용하기

###### For-Each Loop 사용

반복문 안에 key-value 값이 전부 필요할 때 사용한다. 대신 Null Point Exception을 발생시킬 수 있으므로 null 체크를 해야한다.

```java

for (Map.Entry<Integer, Integer> entry : map.entrySet()) {

    System.out.println(entry.getKey(), entry.getValue());

}
```

key, value 값 중 하나만 필요할 떄는 Entry 없이 다음처럼 사용할 수 있다. 이럴 경우 entrySet을 사용할 때보다 약 10% 빨라진다.

```java
for (Integer key : map.keySet()) {
    System.out.println(key);
}
 
for (Integer value : map.values()) {
    System.out.println(value);
}
```

###### Iterator

Generic을 사용하는 방법과 사용하지 않는 방법이 있다. 

Generic 사용
```java
Iterator<Map.Entry<Integer, Integer>> entries = map.entrySet().iterator();

while (entries.hasNext()) {

    Map.Entry<Integer, Integer> entry = entries.next();

    System.out.println(entry.getKey(), entry.getValue());
}
```

Generic 미사용

```java
Iterator entries = map.entrySet().iterator();

while (entries.hasNext()) {

    Map.Entry entry = (Map.Entry) entries.next();

    Integer key = (Integer)entry.getKey();
    Integer value = (Integer)entry.getValue();

    System.out.println(key , value);
}
```
Java 버전이 낮아도 사용할 수 있다는 점과 iterator.remove()를 통해 반복문 도중에 항목들을 삭제할 수 있는 유일한 방법이라는 점이 장점이다.

###### 출처

https://d2.naver.com/helloworld/831311

https://moonong.tistory.com/5

https://fullstatck.tistory.com/10

https://velog.io/@adam2/%EC%9E%90%EB%A3%8C%EA%B5%AC%EC%A1%B0%ED%95%B4%EC%8B%9C-%ED%85%8C%EC%9D%B4%EB%B8%94

