---
title: Java, Stream
categories: Java
tags: 자바
toc: true
toc_sticky: true
---

#### **📌 Stream란?**

🔎 A sequence of elements supporting sequential and parallel aggregate operations.

순차, 병렬 작업을 지원하는 element들의 연속. 자바8에서 추가된 람다를 활용할 수 있는 기술 중 하나로 단어 그대로 데이터의 🌊흐름🌊을 의미한다.  

#### **📌Stream의 장점**

자바8 이전에 배열 또는 콜렉션 인스턴스를 다룰 때는 for 혹은 foreach로 요소 하나씩 꺼내야 했다. 이러한 방법은 로직이 복잡해질 수록 코드 양이 많아져서 여러 로직이 섞이에 되고, 메소드를 나누게 되면 루프를 여러번 돌게 된다. 그러나 stream을 사용할시 아래와 같은 이점으로 이를 보완할 수 있다. 

###### 1. 배열 또는 콜렉션에 메소드를 조합해서 원하는 결과를 출력할 수 있도록 한다.
###### 2. 람다를 이용하여 코드의 길이를 줄이고 간결하게 표현할 수 있다
###### 3. 배열과 콜렉션을 함수형으로 처리할 수 있게 된다.  
###### 4. 간단하게 Multi Threading, 병렬처리를 할 수 있다. 

</div>
</details>
<br/>

<details>
<summary style="color:gray">MultiThreading, 병렬처리란?</summary>
<div markdown="1">   

![NaverAPI](https://user-images.githubusercontent.com/96677719/149865490-f0ca1dcb-bddb-4fe7-b8da-c199e92177e6.png)

네이버 OpenAPI: https://developers.naver.com/docs/common/openapiguide/apilist.md

![APIstore](https://user-images.githubusercontent.com/96677719/149865317-81816b5a-8cc8-4275-941e-937f74900a47.png)

API store: 국내 최초 API 유통 플랫폼 https://www.apistore.co.kr//main.do

REST API: https://restfulapi.net/

</div>
</details>
<br/>


#### **📌Stream의 동작 원리**


stream의 대표적인 특징은 중간 연산 방법을 쓰는 것이다. 반환 값으로 연산의 결과(데이터)가 아닌 스트림을 반환하다가 최종 연산 과정에서 이전 중간 연산들을 모두 합치고 이를 이용해 최종 연산을 한다.

```java
List<String> result = list.stream()
        .filter(x -> { // 중간 연산 1
            System.out.println(x + " in filter method");
            return x.length() >= 1;


        }).map(x -> { // 중간 연산 2
            System.out.println(x + " in map method");
            return x.toUpperCase();


        }).limit(2) // 중간 연산 3
        .collect(Collectors.toList()); // 종결 연산

```


3개의 요소에 대해 메서드를 수행하기 때문에 3번의 호출이 이루어질 것 같지만 실제로는 스트림의 모든 요소가 중간 연산인 filter를 수행하는 것이 아니라 요소 하나씩 모든 파이프라인을 수행한다. 

![image](https://user-images.githubusercontent.com/96677719/150052355-2fefaa41-7b81-413b-9437-dd6015d2bda2.png)

이처럼 filter로 걸러진 값들끼리만 그 다음 연산을 수행하는 것이다. 

이렇게만 보면 연산이 빨라야 할 것 같다. 그러나 실제로는 중간 연산마다 데이터를 저장, 반환하여 넘기는 것이 아니라 연산의 파이프라인을 return 하게 되기 때문에 for-loop를 이용해서 반환값을 필터링 한 후 넘기는 것과는 다르다. 

![image](https://user-images.githubusercontent.com/96677719/150052989-4eafbb4b-f53a-4d5c-9098-c5711c0df024.png)

즉, 연산의 결과가 연산이 종결되기 전에는 stream 타입을 반환하는 것. 스트림 연산의 연속 호출은 여러개의 스트림이 연결되어 스트림 파이프라인을 형성한다. 이러한 동작 원리 때문에 스트림을 사용하기 적절할 때와 적절하지 않을 때가 나뉜다. 

#### **📌Stream을 사용할 때 주의할 점**

stream API는 대체로 for-loop보다 느리다. for-loop는 40년 이상 사용되었다보니 최적화가 잘 되어있는 반면 스트림은 2015년부터 도입되었기 때문에 아직 컴파일러가 최적화를 제대로 하지 못했기 때문이다. 따라서 가독성, 작성의 편안함 측면에서는 stream이 유리할 수 있지만 성능을 고려하면 for-loop가 나을 수 있다. 

📌 적절한 상황:

순서가 있는 원소들을 일관성 있게 변환하거나, 필터링 할 때

순서가 있는 원소들을 연산 후 결합 할 때

순서가 있는 원소들을 Collection에 모을 때

📌 부적절한 상황:

char 값을 처리할 때 - 자바에서는 char용 스트림을 제공하지 않기 때문에 명시적으로 형 변환을 하지 않으면 int 값이 반환된다.)

지역변수(함수 내부에 있는 변수, 함수가 종료되면 값이 사라진다)를 읽거나 수정해야 하는 경우 - 스트림은 일종의 람다 함수인데 람다에서는 지역변수를 수정하는 게 불가능하다. 사실상 final 혹은 final인 변수만 읽을 수 있다. 

+) final: entity가 한번만 할당될 수 있는 것, 즉 한번만 초기화 가능하고 오버라이드, 상속이 불가능 한 것

+) entity(개체): 객체지향 프로그램 언어에서 특정 속성 값을 가진 데이터를 의미
!= object(객체): 데이터 entity와 그 데이터에 관련된 동작을 모두 포함한 것
return, continue, break, exception을 수행해야 하는 경우 - 수행할 수 없다.


#### **출처** 

https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html

https://futurecreator.github.io/2018/08/26/java-8-streams/

https://coding-factory.tistory.com/574

http://www.tcpschool.com/java/java_stream_creation

https://private-space.tistory.com/103
