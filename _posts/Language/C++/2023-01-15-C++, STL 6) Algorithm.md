---
title: C++, STL 6) Algorithm
categories: Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part1: C++ 프로그래밍 입문> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 
## **1. STL Algorithm**

```c++
using namespace std;
#include <algorithm>

int main()
{
    ...
    return 0;
} 
```

컨테이너가 데이터를 저장하는 구조에 대한 것이었다면, 알고리즘은 데이터를 어떻게 사용할 것인지에 대해 정의해둔 것을 의미한다. STL에서 제공하는 알고리즘을 사용하기 위해서는 #include \<algorithm>을 해야한다. 자주 쓰이는 알고리즘의 예시는 아래와 같다. 

<br/>

## **2. find/find_if**

**find**

만약 vector에서 특정한 값을 찾아, 해당 값의 iterator를 반환하고 싶다고 해보자.
```c++
vector<int>::iterator it = find(v.begin(), v.end(), number);
```
이 코드는 아래 코드와 동일하게 동작한다. list, deque 등 iterator가 있는 다른 STL 컨테이너에서도 사용 가능하다. 

```c++
vector<int>::iterator it = v.end();
for(unsigned int i = 0; i < v.size(); ++i)
{
    if(v[i] == number)
    {
        it = v.begin() + i;
        break;
    }
}
```
훨씬 간단하고 가독성이 좋아진 것을 확인할 수 있다. 

<br/>

**find_if**

만약 11로 나누어지는 수의 iterator를 반환하고 싶다고 가정해보자. 이때 find_if를 사용할 수 있다. 

```c++
struct CanDivideBy11
{
    bool operator()(int n){ return (n % 11) == 0; }
};
vector<int>::iterator it = find_if(v.begin(), v.end(), CanDivideBy11());
```
이 코드는 아래 코드와 동일하게 동작한다. 그 외에도 bool 값을 반환하는 함수의 주소값이라면 함수 객체, 함수 포인터, lambda식 등 모두 가능하다. 

```c++
vector<int>::iterator it = v.end();
for(unsigned int i = 0; i < v.size(); ++i)
{
    if(v[i] % 11 == 0)
    {
        it = v.begin() + i;
        break;
    }
}
```
<br/>

## **3. count/count_if**

만약 홀수인 숫자의 개수를 반환하고 싶다고 해보자

```c++
struct IsOdd
{
    bool operator()(int n){ return (n % 2) == 1; }
};

int cnt = count_if(v.begin(), v.end(), IsOdd());
```
위와 같이 간단하게 적을 수 있다. 
<br/>

## **4. all_of/any_of/none_of**
```c++
bool all  = all_of(v.begin(), v.end(), IsOdd());
bool any  = any_of(v.begin(), v.end(), IsOdd());
bool none = none_of(v.begin(), v.end(), IsOdd());
```
이름에서 알 수 있듯이, 해당 컨테이너의 모든 값이 홀수면 all_of에서 참을, 하나라도 홀수면 any_of에서 참을, 모두 홀수가 아니면 none_of에서 참을 반환한다. 

<br/>

## **5. for_each**

범위로 지정된 컨테이너 안의 모든 요소들에 동일한 작업을 해달라는 의미이다. vector 안의 모든 숫자에 3을 곱한다고 해보자. 

```c++
struct Multiply
{
    void operator()(int& n){ n *= 3; }
};

for_each(v.begin(), v.end(), MultiplyBy3());
```

<br/>

## **6. remove/remove_if**

remove를 사용할 때는 유의해야 한다. 홀수인 데이터를 일괄 삭제한다고 해보자. 

```c++
remove_if(v.begin(), v.end(), IsOdd());
```
remove_if의 구조를 보면, 데이터를 찾자마자 삭제하는 방식으로 이루어지지 않는다. vector 공부할 때 봤듯이, 찾자마자 삭제하는 방식으로 동작하면 여러번 복사 이동이 일어나서 비효율적인 코드가 될 가능성이 높다. 따라서 불필요한 데이터를 삭제하기 보다는, 필요한 데이터를 복사하여 남겨두는 방식으로 동작한다. 예를 들자면 아래와 같다. 

```
Before remove_if : 1 4 3 5 8 2
After remove_if  : 4 8 2 5 8 2
```
앞에서부터 순차적으로 돌았을 때, 홀수가 아닌 값인 4, 8, 2만 앞에 복사해준 것을 볼 수 있다. vector에는 위와 같이 값이 들어가고, remove_if(...)의 경우 뒤에 오는 쓰레기 값(5, 8, 2)의 시작 주소를 반환한다. 그러나 이에 대해 모르는 상태에서 보면 그냥 의도한 것과는 다른 vector를 갖게 된 셈이 된다. 따라서 remove를 사용한 이후에는 뒤에 오는 쓰레기 값을 날리는 작업을 별도로 해주어야 한다. 

```c++
vector<int>::iterator it = remove_if(v.begin(), v.end(), IsOdd());
v.erase(it, v.end());
```
or
```c++
v.erase(remove_if(v.begin(), v.end(), IsOdd()), v.end());
```
이렇게 하면 의도했던 대로 홀수가 사라진 [4, 8, 2]의 vector를 구할 수 있다. 

<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part1: C++ 프로그래밍 입문, Rookiss
