---
title: C++, STL 5) Set, Multimap, MultiSet
categories: cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part1: C++ 프로그래밍 입문> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 
## **1. Set**

**1) Set의 동작 원리**

Key와 Value가 똑같은 pair를 담는 연관 컨테이너이다. 

```c++
using namespace std;
#include<set>

int main()
{
    set<int> s;
    return 0;
}
```

<br/>

**2) Set의 문법**

```c++
s.insert(10);                               // 삽입
s.erase(10);                                // 삭제
set<int>::iterator findIt = s.find(10);     // 탐색
for(set<int>::iterator it = s.begin(); it != s.end(); ++it)
{
    cout << (*it) << endl;                  // 순회
}
```
단, Set은 Key와 Value를 함께 관리하는 컨테이너다보니 아래와 같은 문법은 사용할 수 없다. 

```c++
s[10];       // X
s[10] = 10;  // X
```
<br/>

## **2. Multimap**

**1) Multimap의 동작 원리**

중복 Key를 허용하는 Map을 의미한다. 

```c++
using namespace std;
#include<multimap>

int main()
{
    multimap<int, int> mm;
    return 0;
}
```

**2) Multimap의 문법**

```c++
mm.insert(make_pair(1,100));                        // 삽입
mm.erase(10);                                       // 삭제
multimap<int>::iterator findIt = mm.find(10);       // 탐색
for(multimap<int>::iterator it = mm.begin(); it != mm.end(); ++it)
{
    cout << (*it) << endl;                  // 순회
}
```
이때, erase의 경우 동일한 Key를 갖고 있는 모든 값을 삭제하고, 삭제된 요소의 개수를 반환한다. 삭제된 게 없을 경우 0을 반환한다. 또, 탐색의 경우 제일 처음 발견되는 요소의 iterator를 반환한다. 

```c++

multimap<int>::iterator findIt = mm.find(10);       // 탐색
if(findIt != mm.end())
    mm.erase(findIt);                               // 삭제
```

이와 같이 find로 찾은 iterator로 삭제할 경우, 가장 먼저 발견된 요소 한개만 삭제할 수 있다. 

단, Multimap은 Key가 중복될 수 있다보니 아래와 같은 문법은 사용할 수 없다. 

```c++
mm[10];       // X
mm[10] = 10;  // X
```

Multimap의 문법 중, equal_range()라는 문법도 존재한다. 예시 코드는 아래와 같다. 

```c++
mm.insert(make_pair(1,100));
mm.insert(make_pair(1,200));
mm.insert(make_pair(1,300));
mm.insert(make_pair(2,400));
mm.insert(make_pair(3,500));

pair<multimap<int, int>::iterator, multimap<int,int>::iterator> itPair;
itPair = mm.equal_range(1);
```
이 경우, Key 1이 시작하는 pair(1,100)부터 끝난 직후인 pair(2,400)까지의 값이 itPair.fist, itPair.second로 들어갈 것이다. 그 외에도 비슷한 역할을 하는 문법으로 lower_bound, upper_bound가 있다. 

```c++
multimap<int, int>::iterator itBegin = mm.lower_bound(1);  //pair(1,100)
multimap<int, int>::iterator itEnd = mm.upper_bound(1);    //pair(1,200)
```

전체 multimap의 시작과 끝의 경우, 이전처럼 mm.begin(), mm.end()로 나타낼 수 있다. 

```c++
multimap<int, int>::iterator itBegin = mm.begin();  //pair(1,100)
multimap<int, int>::iterator itEnd = mm.end();      //pair(3,500)
```
<br/>

## **3. MultiSet**

**1) MultiSet의 동작 원리**

중복 Key를 허용하는 Set을 의미한다. 

```c++
using namespace std;
#include<multiset>

int main()
{
    multiset<int> ms;
    return 0;
}
```

**2) MultiSet의 문법**

```c++
ms.insert(10);                                      // 삽입
ms.erase(10);                                       // 삭제
multiset<int>::iterator findIt = ms.find(10);       // 탐색
for(multiset<int>::iterator it = ms.begin(); it != ms.end(); ++it)
{
    cout << (*it) << endl;                          // 순회
}
```
Multimap에서 쓸 수 있는 문법 중 상당수가 MultiSet에도 존재한다. 

```c++
itPair = mm.equal_range(10);
itBegin = mm.lower_bound(10);  
itEnd = mm.upper_bound(10);    
```
단, Multiset은 Key가 중복될 수 있다보니 아래와 같은 문법은 사용할 수 없다. 

```c++
ms[10];       // X
ms[10] = 10;  // X
```
<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part1: C++ 프로그래밍 입문, Rookiss
