---
title: Java, Iterator
categories: Java
tags: Java
toc: true
toc_sticky: true
---

📌 Iterator이란?

List, Set, Map, Queue 등의 Collection을 가져오거나 삭제할 때 사용한다. 모든 Collection Framework에 적용 가능하고 요소의 선택적 제거나 대체를 수행할 수 있다는 장점이 있다.

📌 Iterator의 메소드 

Iterator<T> iter = CollectionType.iterator(): Iterator 선언

iter.hasNext(): Iterator 안에 다음 값이 들어있는지 확인 들었으면 true, 안들었으면 false를 반환한다

iter.next()" iterator의 다음 값을 가져온다

iter.remove(): iterator에서 next() 시에 가져왔던 값을 컬렉션에서 삭제. 반드시 next() 후에 사용해야 함

📌 Iterator의 단점

처음부터 끝까지 단방향 반복만 가능하다

값을 삭제하는 것은 가능하나 변경하거나 추가하는 것이 불가능하다

대량의 데이터를 제어할 때 속도가 느리다. 

자바에서는 코드의 명확성을 위해 되도록 enhanced for을 권장한다. 


###### 출처

https://wakestand.tistory.com/247
