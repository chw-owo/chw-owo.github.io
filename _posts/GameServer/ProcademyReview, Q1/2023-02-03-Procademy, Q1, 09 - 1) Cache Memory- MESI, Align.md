---
title: Procademy, Q1, 09 - 1) Cache Memory - MESI, Align
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. MESI**

## **1) MESI와 Cache miss**

Intel에서는 일반적으로 L1, L2 캐시는 따로 사용하고 L3만 공용으로 사용한다. 이때 a라는 변수가 코어 1, 2의 두 L1에 모두 올라가 있고, 코어 1에서 해당 변수를 수정한다면 코어 2의 L1에 있는 a는 어떻게 될까? 동일한 데이터인데 서로 값이 달라질 것이다. 이러한 상황을 막기 위해 캐시 메모리 간에 통신이 필요하다.

현재 가장 많이 사용하는 캐시 통신 프로토콜은 MESI 방법이다. MESI는 Modified Exclusive Shared Invalid 의 약자로 이는 캐시 메모리의 일관성을 유지하기 위해 캐시 메모리가 가질 수 있는 4가지 상태를 의미한다.  MESI 방법을 사용할 경우 위 예시 상황에서는 오래된 데이터, 즉 코어 2 - L1의 a를 Invalid(무효화) 상태로 변경하고 코어 1 - L1의 a는 modified 상태로 변경한다. 동시에 들고 있을 경우 Shared로, 단독으로 사용하고 있으면 Exclusive 상태를 갖는다. 

만약 한 변수를 share하고 있을 때, 나는 가만히 있었음에도 다른 코어에서 해당 변수를 바꿨다면 내 코어에서 해당 변수를 담고 있는 캐시 라인은 전체가 Invalid가 되므로 캐시 미스가 나게 된다. 따라서 이처럼 한 데이터를 코어들이 공유하며 계속 수정하는 상황에서는 캐시 미스의 확률이 높아질 수 있다. 아무 상관 없는 변수를 사용하는데도 같은 캐시 라인을 사용한다는 이유로 계속해서 캐시 미스가 나는 것이다. 

## **2) 동기화 문제**

멀티스레드를 배우게 되면 out of ordering 개념이 등장하는데 이는 명령어를 순서대로 동작시키는 대신 성능을 위해 CPU가 비순차적으로 명령어 수행의 순서를 지정하는 것을 의미한다. 이에 대해 검색해보면 “캐시 메모리 동기화 이전에 순서가 달라져서 이전의 데이터를 사용하게 된다” 와 같은 설명을 마주하게 될텐데, 이는 사실 틀린 설명이다. 캐시 메모리는 시시각각 통신하기 때문에 어떤 상황에서도 이전 값을 갖고 있지는 않다. 이러한 문제는 스토어 버퍼 단계에서 발생하는 문제이다. 

예를 들어, mov [a] eax를 한다면 시간을 절약하기 위해 CPU에서 직접 wirtethrow 하는 대신 store buffer에 바뀔 값을 넣게 된다. 이후 CPU는 마저 할 일을 하고, 그와는 별개로 store buffer에서 L1으로 writethrow하며 최종적으로 값이 수정된다. 근데 CPU에서 store buffer에 넣는 것과 store buffer에서 L1에 넣는 것 사이에 시간 차가 생길 수 있는데 여기서 동기화 문제가 발생하는 것이다. 즉 캐시와 store buffer 사이의 동기화 문제인 것이지 cache 사이의 동기화 문제는 아닌 것임을 알고 있어야 한다. 

<br/>

# **2. Align**

```c++
// 전역, 지역의 경우
alignas(64); 

// 동적 할당의 경우 
_aligned_malloc(sizeof(-), 64);
```

이 둘은 Modern C++에서 등장한, 주소 경계를 맞춰주는 키워드이다. 이전에는 __declspec(align(64))를 사용하여 이 작업을 했는데 이것은 C++ 표준이 아니다. alignas 혹은 _aligned_malloc을 활용하면 캐시라인을 분리하여 Cache miss 문제를 일부 보완할 수 있다. 이때, 지역 변수에 이를 사용하려면 애초에 ebp 위치가 해당 경계에 있어야 하기 때문에, 2-3년 전까지는 지역변수에서는 이 키워드가 제대로 동작하지 않았다. 그러나 2023년 기준 현재는 이 부분이 패치가 되어서 지역 변수에서도 bp 마스킹을 통해 경계를 맞춰준다.

```c++
struct stSAMPLE
{
int a;
int b;
alignas(64) int c; 
}	
```

이렇게 구조체 및 클래스 내부에 alignas 키워드를 이용해 변수를 선언할 경우, 해당 구조체 전체를 64 경계에 세우고, 또 int c도 64 경계에 세우게 된다. 따라서 위 구조체의 경우 총 128 byte의 자료형이 된다. 

위에서 살펴봤듯이 변경이 잦은 변수(ex 좌표값)를 변경이 적은 변수(ex 회원 번호)와 같은 캐시라인에 두는 것은 비효율적일 수 있다. 혹은 동기화 목적으로 lock을 많이 사용해야 하는 변수가 있다면 이 경우에도 다른 변수들과 거리를 두는 것이 더 나을 것이다. 혹은 첫 선언 이후 캐시 히트가 100%여야 하는 구조체가 있다면 크기를 64byte로 맞춘 뒤 시작점을 align 해주면, 컨텍스트 스위칭이 일어났을 때를 제외하면 큰 효과를 볼 것이다. 물론 64byte 이상이거나, 변수가 너무 작을 경우 큰 의미가 없을 것이다. 

반면, 이러한 캐시라인 분리를 너무 자주 사용하는 건 캐시메모리의 낭비가 될 수 있으며 상황에 따라서는 오히려 캐시 미스가 더 많아지기도 한다. 예를 들어 한 코어에서 두 변수를 모두 다 자주 접근한다면 (ex 좌표와 회원 번호를 함께 읽어오는 상황 등) 오히려 다른 캐시라인을 사용하는 게 캐시 미스를 유도할 수도 있다. 따라서 이 변수가 어떻게 사용되는지 논리적으로 잘 따져서 조심스럽게 trade-off 한 뒤 결정하는 것이 좋다. 실제로 이러한 이유로 자주 사용되지는 않는다. 따라서 alignas를 사용할 것이라면 주석으로 의도를 명확히 밝히는 것이 좋다.

<br/>

# **3. 질문**

**Q.** 두 코어에서 동시에 한 변수를 건드리면 어떻게 되나요?

clock이 들어갈 때 두 코어가 동시에 동작하는 것은 맞지만, Cache의 통신에 있어서는 순서가 정해지기 때문에 동시에 값을 쓸 일은 없습니다. 단, 두 코어에서 다른 코어들의 해당 변수를 invalid로 바꾸라는 신호를 동시에 보낸다면 조금 달라질 수도 있습니다. 그런 상황에서 어떻게 동작하는지는 좀 더 알아본 후 알려드리겠습니다.

**Q.** 그럼 Invalid 상태가 되면 해당 변수에 접근할 수 없게 되나요?

접근할 수는 있지만 Modified 변수에 저장된 값을 불러올 것입니다. 

**Q.** 그래도 멀티 코어는 L3로 통신이 가능한데, 멀티 프로세서 (멀티 CPU)의 경우에는 어떻게 통신하나요? 

멀티 CPU 환경에서는 RAM을 완전히 분리해버리기 때문에 양쪽 CPU가 한 데이터에 동시에 접근하는 경우가 없습니다. 다른 RAM의 데이터를 가져오려면 요청을 통해 가져와야 합니다.
