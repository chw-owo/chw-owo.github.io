---
title: C++, Modern C++ 02) R value Reference
categories: Cpp
tags: 
toc: true
toc_sticky: true
---
## **1.L-value vs R-value**

**L-value**

단일식을 넘어서 계속 지속되는 개체

**R-value**

L-value가 아닌 나머지 개체. 임시 값, 열거형, 람다, i++ 등. 

```c++
a = 3;      // O 
3 = a;      // X
(a++) = 3   // X
```
a는 Left, Right 양측 어디에든 올 수 있지만 3, a++ 같은 경우는 Right에만 올 수 있다. 이러한 개체들을 R-value라고 부른다.  

## **1.R-value Reference**

```c++
void CopyByRef (Knight& knight) { ... }
void CopyByConstRef (const Knight& knight) { ... }

int main()
{
    CopyByRef(Knight())         // X
    CopyByConstRef(Knight())    // O
    return 0;
}
```
위의 상황에서 Knight()는 R-value이기 때문에 CopyByRef는 에러가 나고 CopyByConstRef는 통과하게 된다. CopyByConstRef가 통과하는 이유는 const를 통해 해당 값을 읽기만 하고 수정하지는 않겠다고 명시했기 때문이다. 이때, R-value Reference를 사용하면 const를 사용하지 않고도 참조 형식으로 넘겨줄 수 있다. (포인터도 마찬가지)

```c++
void CopyByRRef (Knight&& knight) { ... }

int main()
{
    Knight k
    CopyByRRef(k)                // X
    CopyByRRef(Knight())         // O
    return 0;
}
```
&&은 참조의 참조가 아니라, R-value를 받을 수 있는 또 다른 의미의 표기로 봐주는 것이 좋다. R-value는 기존에 받지 못했던 R-value는 받을 수 있는 반면 일반적인 L-value(위 예시의 경우 k)를 받을 수 없는 특징을 갖는다. 어셈블리어를 까보면 CopyByRef, CopyByConstRef, CopyByRRef 세 경우 모두 크게 다르지 않은 것을 확인할 수 있다. 셋 다 원본을 넘기고 있으므로, R-value 참조에서도 아래와 같은 수정이 가능한 것이다. 

```c++
void CopyByRRef (Knight&& knight) 
{
    knight._hp = 20;
}
```
Knight()는 임시 개체인데 이렇게 바꿀 수 있도록 새로운 문법을 추가했다는 게 의미없게 보일 수 있다. 그러나 R-value는 이러한 개체만 받을 수 있는 것은 아니다. 

```c++
void CopyByRRef (Knight&& knight) { ... }

int main()
{
    Knight k
    CopyByRRef(static_cast<Knight&&>(k));  
    return 0;
}
```
캐스팅을 거쳐서 위와 같이 넘겨줄 수도 있다. 그럼 이게 기존의 CopyByRef (Knight& knight)와 무슨 차이가 있을까? 우선 R-value reference는 기본적으로 전달 받은 값을 사용한 뒤, 원본 데이터를 재사용하지 않을 것이라 가정하고 동작하게 된다. 즉, 인자로 전달 받은 원본 데이터를 보다 자유롭게 조작할 수 있게 된 것이다. 어떤 상황에서 R-value 참조가 필요한지 코드로 예를 들어보자. 

```c++ 
Knight()
{
public:
    void operator=(Knight&& knight) noexcept
    {
        _pet = knight._pet;
        knight._pet = nullptr;
    }
public:
    Pet* _pet;
}

int main()
{
    Knight k1;
    k1._pet = new Pet();
    ...

    Knight k2;
    k2 = static_cast<Knight&&>(k1);
}
```

R-value 참조를 이용하면, 위처럼 상대 Player(k1)의 pet를 그대로 뺏어오는 상황도 만들 수 있게 된다. 다른 기능이다보니 그대로 비교할 수는 없지만, 이 경우 새로 pet을 만들어야 하는 깊은 복사에 비해 더 빠른 연산이 가능하다. 이러한 R-value reference를 이용한 연산자, 생성자를 이동 대입 연산자, 이동 생성자라고 부른다.

```c++
int main()
{
    Knight k1;
    k1._pet = new Pet();
    ...

    Knight k2;
    k2 = std::move(k1);
}
```

캐스팅 문장을 길게 적을 필요 없이, std::move로도 동일한 동작을 할 수 있다. 원본 데이터를 날려도 되는 상황이 빈번하지는 않지만, 간혹 쓰이므로 알아두는 것이 좋다. 

unique_ptr를 이용해 복사할 수 없는 포인터를 만든 상황에서도 쓰일 수 있다. 

```c++
std::unique_ptr<Knight> uptr1 = std::make_unique<Knight>();
std::unique_ptr<Knight> uptr2 = uptr1               // X
```
위 코드는 실행할 수 없다. unique_ptr의 내부로 들어가보면 복사와 관련된 연산자의 사용을 막아두었기 때문이다. 

```c++
std::unique_ptr<Knight> uptr1 = std::make_unique<Knight>();
std::unique_ptr<Knight> uptr2 = std::move(uptr1)    // O
```
하지만 이러한 상황에서도 move, 즉 R-value 참조를 사용하면 uptr을 넘길 수 있다. 이 값은 더 이상 사용하지 않을 것이니 마음대로 사용하라는 의미를 담고 있기 때문이다. 

<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part1: C++ 프로그래밍 입문, Rookiss
