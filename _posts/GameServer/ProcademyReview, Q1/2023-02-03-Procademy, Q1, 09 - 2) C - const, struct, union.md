---
title: Procademy, Q1, 09 - 2) C - const, struct, union
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. const**

```c++
int const a = 1;
int * aaa = (int*)&a;
*aaa = 1000;

printf(“%d\n”, a);
```

readonly, code 영역의 경우 읽기 전용이기 때문에 값을 수정할 수 없지만, const로 지역에 선언한 것은 값을 수정할 수 있다. 우리가 const 변수는 수정하지 않겠다고 약속을 한 것이지 어쨌든 stack 메모리에 올라가는 변수이기 때문에 우회하면 값을 바꿀 수 있다. 이를 출력해보면 a 주소의 값이 1000으로 바뀐 것을 볼 수 있다. 반면 콘솔에서 a을 출력해보면 여전히 1로 출력이 된다. a를 상수로 인식하기 때문에 매크로 사용하듯이 처음 초기화 했던 값으로 치환하여 출력해주는 것이다. 만약 a가 변수처럼 쓰일 수 밖에 없는 상황이라면 바뀐 값인 1000이 들어갈 것이고, 위처럼 상수로 치환할 수 있는 상황이라면 1로 치환되어 사용될 것이다. 이러한 동작 원리 덕분에, 상수를 선언할 때 #define 대신 const를 사용하는 경우도 종종 있습니다. 계산까지도 컴파일러가 런타임 이전에 해주기 때문입니다.

<br/>

# **2. struct**

구조체에 구조체를 대입하면 기본적으로 복사가 된다. 아래와 같은 구조체를 복사 대입한다면 mov 세번으로 깔끔하게 복사가 될 것이다. 

```c++
struct stSAMPLE
{
    int a;
    int b;
    int c; 
}	
```
엄청 큰 변수가 들어있는 경우 32bit에서는 dword ptr로, 64bit에서는 byte ptr로 복사 대입을 한다. byte단위로 이러한 과정을 실행하는 것은 성능 저하로 이어질 수 있으니 디스어셈블리를 통해 어떻게 이루어지고 있는지 확인하는 것이 좋다. 

<br/>

# **3. union**

가장 큰 용량의 멤버를 기준으로 멤버들을 겹쳐서 사용한다. 많이 쓰진 않지만 windows api, socket api에서는 간혹 사용할 일이 생긴다. 

union이 쓰인 대표적인 예시가 LARGE_INTEGER. 32bit에서는 8byte를 사용할 때 상위 / 하위 4byte씩 나누어 쓰기 위해 LARGE_INTEGER struct를 사용했고, 값을 초기화 할 때도 lowpart, highpart를 따로 초기화해주어야 했다. 그리고 64bit로 오면서 이를 union으로 만들어서 32bit와의 호환성을 보장해주었다. 또 다른 예시로, SockAddr이라는 IPv4를 표현하기 위해 union은 4byte짜리 in_addr 한개와 1byte짜리 char a1, b1, c1, d1 4개를 동시에 제공하고 있다. 

# **4. Memory 접근 함수**

```c++
memcpy, memccpy, memchr, memcmp, memicmp, memmove 
```

이 중 memset, memcmp, memcpy가 제일 자주 사용된다. 디스어셈블리를 보면 기본적으로 repeat mov를 바탕으로 동작한다. memset은 지정된 값으로 메모리를 초기화 하며, memcpy는 메모리 복사에 쓰인다. memcmp, memicmp는 input1이 input2보다 작으면 0 이하를, 크면 0 이상을, 같으면 0을 반환하는 비교 함수로, memcmp는 대소문자를 구분하고 memicmp는 무시한다. 

memcpy, memccpy, memmove 모두 메모리 복사에 쓰이는데 몇가지 차이점이 있다. memcpy는 null 검사 없이 n byte만큼 그대로 복사하며, memccpy는 복사 중 지정한 값이 나올 경우 그 값까지 복사한 뒤 복사가 끝난 dest의 다음 주소를 반환한다. 만일 지정한 값을 만나지 않은 경우 n byte를 복사하고 null을 반환한다. memmove는 겹치지 않는 메모리 영역부터 먼저 복사하고, 겹치는 영역은 src 값을 임시 저장한 뒤 복사하기 때문에 다소 느린 대신 두 메모리가 겹치는 상황에서도 쓸 수 있다.

