---
title: Procademy, Assignment 02) 난수 생성기
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---
## **과제**

**-** C 런타임 라이브러리의 rand()를 분석하여 동일한 원리로 동작하는 난수 생성기를 구현한다.

**-** C 런타임 라이브러리 rand()와 직접 구현한 rand()가 동일한 결과값을 출력해야 한다. 

<br/>


## **과제 과정**

우선 디스어셈블리로 rand 함수가 어떻게 동작하는지 봤다. (vs2022, Release Mode, x86 기준) 코드 분석과 관련없는 부분들은 일부 생략했다. 

```c++
int main()
{
    int r;
    srand(time(NULL));
    r = rand();
}
```

**1) main**
```
00AF1020  push        ebp  
00AF1021  mov         ebp,esp  
00AF1023  push        ecx  

srand(time(NULL));

00AF1024  push        0  
00AF1026  call        time (0AF1000h)  
00AF102B  add         esp,4  
00AF102E  push        eax  
00AF102F  call        dword ptr [__imp__srand (0AF20C8h)]  
00AF1035  add         esp,4  

int r = rand();

00AF1038  call        dword ptr [__imp__rand (0AF20C4h)]  
00AF103E  mov         dword ptr [r],eax  
}
00AF1041  xor         eax,eax  
00AF1043  mov         esp,ebp  
00AF1045  pop         ebp 
```
main 함수의 디스어셈블리는 단순하다. Null 값으로 0을 push한 뒤 time을 호출하고 eax에 time의 결과 값을 담아서 srand를 호출한다. 그 후 rand함수를 호출, eax에 그 결과 값을 담아서 r에 대입한 뒤 스택을 정리한다. 

<br/>

**2) __imp__srand (0AF20C8h)**

```
call __acrt_getptd
```
일단 time을 call 하고 나면 eax에 time(null)의 결과물인 1673970771가 들어있다. 이를 push 해서 ebp의 상대주소 형식으로 사용하게 된다. 그리고 __imp__srand를 통해 이 seed 값을 저장하여 이후 rand() 호출 시에 계속 사용할 수 있게끔 만든다. 

<br/>

**3) __imp__rand (0AF20C4h)**

rand에서 난수 생성과 연관있는 명령어만 살펴보았다. 

```
imul        eax,dword ptr [ebx+18h],343FDh  
add         eax,269EC3h  
mov         dword ptr [ebx+18h],eax  
shr         eax,10h  
and         eax,7FFFh
```
ebx + 18h의 위치에 기존 srand()의 결과값, 즉 seed값인 1673970771가 저장되어있다. 읽기 쉽게 십진수로 바꾸자면 아래와 같다.

```
eax = 1673970771 * 214013

eax = eax + 2531011 (이는 다음 seed 값이 된다)

seed = eax

eax = eax / 65,536 (2의 16승)

eax = eax & 2047 (비트 연산)
```

이 과정을 거쳐 난수인 rand()의 결과값인 28,145가 나오게 된다. 이후의 연산들 역시 지금 seed값으로 저장해둔 값을 불러와 같은 과정을 반복하게 된다. 

<br/>

## **과제 코드**

```c++
#include <iostream>
using namespace std;

class CustomRand 
{
public:
    CustomRand()
    {
        _seed = 1;
    }

    CustomRand(int seed)
    {
        _seed = seed;
    }

    int operator()()
    {
        int random; 
        random = _seed * 0x343FD;
        random += 0x269EC3;
        _seed = random;
        random >>= 0x10;
        random &= 0x7FFF;

        return random;
    }

private:
    int _seed;
};

void GetOriginRand()
{
    srand(time(NULL));

    for(int i= 0; i < 10; i++)
        printf("origin: %d\n", rand());
}

void GetCustomRand()
{
    CustomRand customRand(time(NULL));

    for (int i = 0; i < 10; i++)
        printf("custom: %d\n", customRand());
}

int main()
{
    GetOriginRand();
    printf("\n");
    GetCustomRand();
}
```

seed 값 저장을 위해 함수 객체를 사용했으며 seed 값 디폴트는 기존 rand()에서와 마찬가지로 1로 설정해주었다. int operator() 안의 내용은 위에서 분석했던 어셈블리어 코드를 C++로 한줄씩 변환하여 작성했다. 

결과는 다음과 같다. 

![cmd](https://user-images.githubusercontent.com/96677719/212955949-f18d5a61-8d78-4306-8e10-52fd71452ae1.png)

<br/>

## **과제를 통해 느낀 것**

라이브러리에 있는 함수를 혼자 어셈블리로 뜯어본 것은 처음이었는데 내가 원하는 것과 관련되는 부분이랑 관련되지 않은 부분을 구분하는데 생각보다 시간이 오래 걸렸다. 조금 눈에 익으니까 어떤 건 넘기고 어떤 걸 봐야할지 눈에 들어오긴 했는데 (특히 이번 과제의 경우 그냥 mul, add, shr 같은 연산자 위주로 보면 해결이 되는 거였다) 자연스럽게 분석하기 위해서는 아직 연습이 많이 필요할 것 같다. 

<br/>

## **과제 피드백**

설정이나 컴파일러 버전에 따라 알아보기 힘들 게 출력되는 경우도 있다. 그때 호출 스택을 같이 보면 더 해석하기 쉽다. 호출 스택을 참고할 경우 내가 보는 어셈블리가 어떤 함수의 어셈블리인지 알 수 있다. 이때 유의할 점은 외부 코드 표시 활성화가 안되면 내가 짠 코드만 나오고 외부 라이브러리의 어셈블리가 나오지 않는다. 또, 기호로드를 켜놓고 진행하는 것이 좋다. 기호 로드를 켜야 어셈블리와 매핑 시키는 과정에서 함수 이름을 불러줌. 이걸  켜두어야 디버깅용 코드인지 rand()를 위한 코드인지 명확히 알기가 더 수월하다. 

<br/>

## **질의응답**

**Q. rand 범위가 short인데 부족하지 않나요?**

부족한 경우도 많습니다. 'rand 범위는 short니까 rand의 호출 횟수를 최소화하여 이 범위를 확장시키세요' 라는 문제도 출제되고는 합니다. 이 문제에서 가장 흔한 접근은 더하기인데 이는 너무 비효율적입니다. 그 다음 흔한 접근은 곱하기인데, 이는 배수가 되기 때문에 값이 비게 되어 결함 그 자체라고 볼 수 있습니다. 따라서 이런 경우 bit 단위로 접근하는 게 좋습니다. 15bit짜리 rand니까 left shift하면 30bit rand로 만들 수 있습니다 

**Q. 런타임 라이브러리가 무엇인가요?**

C, C++을 편하게 사용하기 위해 제공하는 라이브러리로, C, C++의 표준이기 때문에 어떤 컴파일러든 간에 똑같이 작동됨을 보장하는 라이브러리입니다. 모든 함수와 문법의 작동을 표준대로 따르게 됩니다. 내부(어셈블리)는 조금 다를 수도 있는데 그럼에도 결과는 동일하게 나올 것임을 보장합니다.

<br/>

