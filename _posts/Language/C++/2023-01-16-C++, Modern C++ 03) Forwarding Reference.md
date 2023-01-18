---
title: C++, Modern C++ 03) Forwarding Reference
categories: Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part1: C++ 프로그래밍 입문> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 
## **1. 전달 참조란?**

전달 참조, Forwarding Reference

기존에는 보편 참조, Universal Reference라는 단어로 쓰였다. C++17로 넘어오면서 용어가 바뀌었지만 둘 다 의미는 같다. 

```c++
template<typename T>
void ForwardingRef(T&& param)
{
    ...
}

int main()
{
    Knight k1;
    ForwardingRef(k1);              // O 
    ForwardingRef(std::move(k1));   // O
    return 0;
}
```
얼핏 보았을 때는 R-value 참조와 동일하게 보이지만, R-value 참조는 R-value만 받을 수 있는 것과 달리 전달 참조는 L-value도 받을 수 있다. Visual studio에서 확인해보면 ForwardingRef(k1) 경우 L-value로, ForwardingRef(std::move(k1)) 경우  R-value로 넘어가고 있지만 둘 다 잘 동작함을 확인할 수 있다. 

```c++
int main()
{
    Knight k1;
    auto&& k2 = k1                  // O 
    auto&& k3 = std::move(k1)       // O
    return 0;
}
```
auto의 경우에도 template를 사용했을 때와 마찬가지로 L-value, R-value 모두 잘 받을 수 있는 것을 확인할 수 있다. 단, template의 && 인자 앞에 const를 붙이면 R-value 참조로 고정된다. 

이처럼 형식 연역 (type deduction)이 발생하는 상황에서 &&로 R-value 참조를 사용해주면 전달 참조가 된다. L-value를 넣으면 L-value 참조로, R-value를 넣으면 R-value 참조로 동작한다. 번거롭게 함수를 여러개 선언하지 않고도 한번에 처리할 수 있는 것이다. 

문제는 R-value 참조 함수와 L-value 참조 함수가 전혀 다르게 동작한다는 것에 있다. 따라서 넘겨준 참조값이 R-value인지 L-value인지에 따라서 forwarding을 따로 해주어야 한다. 

<br/>

## **2. std::forward**

```c++
void Copy (const Knight& knight) { ... }
void Copy (Knight&& knight) { ... }

template<typename T>
void CopyByRRef(T&& param)
{
    Copy(std::forward<T>(param));
}

int main()
{
    Knight k;
    CopyByRRef(k);              // Copy(const Knight& knight)
    CopyByRRef(std::move(k));   // Copy(Knight&& knight)    
    return 0;
}
```

이렇게 forward로 명시해주면, L-value 참조일 경우 복사 연산자인 Copy(const Knight& knight)를, R-vlaue 참조일 경우 이동 연산자인 Copy(Knight&& knight)를 자동으로 호출해준다. 

<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part1: C++ 프로그래밍 입문, Rookiss
