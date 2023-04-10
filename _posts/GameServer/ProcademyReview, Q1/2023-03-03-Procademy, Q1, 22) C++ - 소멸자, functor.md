---
title: Procademy, Q1, 22) C++ - 소멸자, functor
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 전역 객체의 소멸자**

전역 객체는 프로세스 종료 시 소멸자가 알아서 호출되는데, 이는 내부적으로 atexit()를 통해 이루어진다. atexit()는 프로그램이 종료할 때 수행해야 하는 기능을 등록하는 함수로, 함수 포인터를 등록해두면 main 함수가 return 된 이후에 등록된 함수들이 자동으로 호출된다. 생성자는 직접 호출할 수 없지만 소멸자는 가능하기 때문에 사용자가 직접 atexit()로 이러한 기능을 등록할 수도 있다. 

```c++
Ctest g_test
void CtestDestroy
{
	g_test -> ~Ctest();
}
```
atexit가 진행하는 소멸자 호출 부분을 코드로 예시를 들자면 아래와 같다. 소멸자 호출은 thiscall이 들어가는 반면 atexit는 인자 없는 함수만 지원하는데, 처리하기 위해 컴파일러는 전역으로 존재하는 객체마다 그 전역 객체의 소멸자를 thiscall 하는 코드를 포함하여 함수를 만든 뒤 atexit에 등록한다. 전역 객체의 주소는 생성자가 호출되기도 전에 정해지기 때문에 이런 작업이 가능하다.

<br/>

# **2. functor**

괄호 연산자를 통해 클래스를 함수처럼 작동하게끔 오버로딩하여 호출 가능한 객체로 만든 것을 functor라고 한다. C++ STL 등을 보면 functor로 구현된 것이 많은데, C 스타일의 함수 포인터와 유사하게 사용된다. 간단한 예제 코드는 아래와 같다. 

```c++
class AddFunctor {

public:
	int operator()(int input){
		return total += input;
	}

private:
	int total = 0;
}

int main()
{
	AddFunctor addFunctor;
	addFunctor(1);
	addFunctor(2);
	int result = addFunctor(3);
}
```

functor은 객체이므로 생성자, 소멸자, 멤버 변수, 멤버 함수 등을 가질 수 있으며 template을 통한 추상화도 가능하다. 이러한 것들을 이용하면 디테일한 기능이 가능한 함수 객체를 만들 수 있다. 