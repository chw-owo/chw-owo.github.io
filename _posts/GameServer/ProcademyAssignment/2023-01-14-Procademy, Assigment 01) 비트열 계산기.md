---
title: Procademy, Assignment 01) 비트열 계산기
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

비트 연산에 익숙해지기 위한 과제. 소인수분해, 포인터 등을 사용하지 않고, 비트 연산자를 사용하여 구현하는 것이 과제의 조건이었다. 이 코드는 32bit Window 환경에서 작성되었으며, short, int의 size가 달라질 경우 다르게 동작할 수 있다. 

## **1. Bit Translator**

**1)** unsigned char 변수의 값을 비트 단위로 찍어주는 코드를 작성하라. 

**2)** 지역 변수에 특정한 값 하나를 넣고 (input으로 받는 게 아님), 비트 단위로 분해하여 0이면 0, 1이면 1을 출력한다.  

```c++
#include <iostream>

void DecimalToBinary(unsigned char input)
{
	unsigned char flag = 0;
	printf("%u 의 바이너리 : ", input);

	for (short i = (sizeof(input) * 8) - 1; i >= 0; i--)
	{
		flag = 1 << i;

		if (input & flag)
			printf("%u", 1);
		else
			printf("%u", 0);
	}
	printf("\n");
}

int main()
{
	DecimalToBinary(40);
}
```

```c++
unsigned char flag = 0;
printf("%u 의 바이너리 : ", input);

for (short i = (sizeof(input)*8)-1; i >= 0; i--)
{
    flag = 1 << i;
    ...
}
```
우선 bitflag를 만든다. 그 후 (sizeof(input) * 8) - 1 부터 시작하여 0까지, 1씩 줄여가며 1을 left shift 한다. input이 char이라면 char의 크기는 1이므로 (1 * 8) - 1, 7부터 0까지 총 8번 반복문을 시행하며 그 과정을 통해 아래와 같은 flag들이 만들어진다. 

```
10000000    // 1 << 7
01000000    // 1 << 6
00100000    // 1 << 5
00010000    // 1 << 4
00001000    // 1 << 3
00000100    // 1 << 2
00000010    // 1 << 1
00000001    // 1 << 0
```

```c++
for (short i = (sizeof(input)*8)-1; i >= 0; i--)
{
    flag = 1 << i;

    if (input & flag)
        printf("%u", 1);
    else
        printf("%u", 0);
}
```

그리고 bitflag 와 input 값을 비트 연산자 &로 비교해서, input의 해당 위치가 1이면 1을, 0이면 0을 출력한다. 

결과는 아래와 같이 나온다.

![cmd](https://user-images.githubusercontent.com/96677719/212535050-13a7641d-a677-42c7-9233-0a3be03eb990.png)

<br/>

## **2. Bit Controller**

**1)** unsigned short (16bit) 변수의 각 비트를 컨트롤하는 코드를 작성하라.

**2)** unsigned short 변수 = 0으로 초기값을 가진다.

**3)** 키보드로 1~16의 비트 자리 입력을 받는다. 그 후 \[0/1]을 사용자로부터 입력 받는다.

**4)** 지정된 자리의 비트를 입력 받은 \[0/1]로 바꾸어준다. 이때, 지정된 자리 외의 다른 자리의 기존 비트 데이터는 보존이 되어야 한다. 

```c++
#include <iostream>

void BitController()
{
	unsigned short value = 0;
	unsigned short flag = 0;
	unsigned char index, input;

	while (true)
	{
		printf("비트위치: ");
		scanf_s("%u", &index);
		
		printf("OFF/ON [0,1] : ");
		scanf_s("%u", &input);

		if (index < 1 || index > sizeof(value) * 8)
		{
			printf("비트 범위를 초과하였습니다.\n\n");
			continue;
		}

		if (input == 0)
			value &= ~(1 << (index - 1));
		else
			value |= 1 << (index - 1);

		for (char i = 0; i < sizeof(value) * 8; i++)
		{
			flag = 1 << i;

			if (value & flag)
				printf("%u 번 Bit : ON\n", i+1);
			else
				printf("%u 번 Bit : OFF\n", i+1);
		}
		printf("\n");
	}
}

int main()
{
	BitController();
}
```

전체 코드는 위와 같다. 

```c++
unsigned short value = 0;
unsigned short flag = 0;
unsigned char index, input;

while (true)
{
    printf("비트위치: ");
    scanf_s("%u", &index);
    
    printf("OFF/ON [0,1] : ");
    scanf_s("%u", &input);

    if (index < 1 || index > sizeof(value) * 8)
    {
        printf("비트 범위를 초과하였습니다.\n\n");
        continue;
    }
    ...
}
```
우선 bitflag로 사용할 flag, 컨트롤 하는 데이터가 될 value 값을 unsigned short로 만들어준다. 비트 위치는 index에, on/off 값은 input에 받는다. 이 둘은 0~255을 벗어날 일이 없기 때문에 메모리를 아끼기 위해 unsigned char로 선언해주었다. 만약 비트 위치가 1~16(sizeof(short) * 8) 범위를 벗어날 경우 continue로 입력부터 다시 받도록 한다. 

```c++
	while (true)
	{
        ...
		if (input == 0)
			value &= ~(1 << (index - 1));
		else
			value |= 1 << (index - 1);
		...
	}
```
만약 input이 0이라면 1을 L shift 해준 값을 not-and 한다. 예를 들어, 1011에서 세번째 1만 0으로 바꾸고자 한다고 가정해보자. 0010을 not 하여 1101로 만든 뒤, 1011 and 1101 연산을 거치면 결과 값은 1001으로, 나머지 값은 유지하고 해당하는 값만 바꿀 수 있다. 만약 1001이어서 바꿀 필요가 없었다고 하더라도, and 연산이기 때문에 원하는 대로 해당 부분은 0으로 유지할 수 있다.  

만약 input이 1이라면 1을 L shift 해준 값을 or 한다. 예를 들어, 1001에서 세번째 0만 1로 바꾸고자 한다고 가정해보자. 0010을 or 하면 나머지 값은 유지하고 해당하는 값만 바꿀 수 있다. 만약 1011이어서 바꿀 필요가 없었다고 하더라도, or 연산이기 때문에 원하는 대로 해당 부분은 1로 유지할 수 있다.  


```c++
	while (true)
	{
        ...
		for (char i = 0; i < sizeof(value) * 8; i++)
		{
			flag = 1 << i;

			if (value & flag)
				printf("%u 번 Bit : ON\n", i+1);
			else
				printf("%u 번 Bit : OFF\n", i+1);
		}
		printf("\n");
	}
```
Assignment 1에서 사용했던 대로, short의 비트수만큼 flag를 하나씩 밀어주며 value 와 flag를 and 연산 한다. 이번에는 value가 1이면 ON을, 0이면 off를 출력한다.  

출력 결과는 아래와 같다. 

![cmd](https://user-images.githubusercontent.com/96677719/212536246-6095e3fb-b142-41b2-95d0-8226fcd4ead1.png)

<br/>

## **3. Byte Controller**

**1)** unsigned int (32bit) 변수를 바이트 단위로 컨트롤하는 코드를 작성하라 

**2)** unsigned int 변수는 0의 초기값을 가진다.

**3)** 키보드로 1~4의 바이트 위치를 입력 받고, 해당 위치에 넣을 데이터 0~255를 입력 받는다. 

**4)** 입력받은 위치에 해당 값을 넣되, 그 외 위치의 기존 바이트 데이터 값은 보존이 되어야 한다. 

**5)** 해당 변수를 바이트 단위로 쪼개서 출력하고 마지막으로 4byte 16진수로 출력한다. 

```c++
#include <iostream>

void ByteController()
{
	unsigned int value = 0;
	unsigned char index, input, tmp;

	while (true)
	{
		input = 0;

		printf("위치 (1~4) : ");
		scanf_s("%u", &index);

		printf("값 [0~255] : ");
		scanf_s("%u", &input);

		if (index < 1 || index > sizeof(value))
		{
			printf("위치 범위를 초과하였습니다.\n\n");
			continue;
		}

		if (input < 0 || input > 0xff)
        {
            printf("데이터 범위를 초과하였습니다.\n\n");
            continue;
        }

		
		value &= ~(0xff << ((index - 1) * 8));
		value |= input << ((index - 1) * 8);

		for (char i = 0; i < sizeof(value); i++)
		{
			tmp = value >> (i * 8);
			printf("%u 번째 바이트 값 : %u \n", i+1, tmp);
		}

		printf("\n전체 4바이트 값 : 0x%p \n\n", value);
	}
}

int main()
{
	ByteController();
}
```
전체 코드는 위와 같다. 

```c++
unsigned int value = 0;
unsigned char index, input;

while (true)
{
    input = 0;

    printf("위치 (1~4) : ");
    scanf_s("%u", &index);

    printf("값 [0~255] : ");
    scanf_s("%u", &input);

    if (index < 1 || index > sizeof(value))
    {
        printf("위치 범위를 초과하였습니다.\n\n");
        continue;
    }

	if (input < 0 || input > 0xff)
	{
		printf("데이터 범위를 초과하였습니다.\n\n");
		continue;
	}

    ...
}
```
컨트롤 하는 데이터가 될 value 값을 unsigned int로 만들어준다. 바이트 위치는 index에, 0-255 사이의 값은 input에 받는다. 이 둘은 0~255을 벗어날 일이 없기 때문에 메모리를 아끼기 위해 unsigned char로 선언해주었다. 만약 바이트 위치가 1~4(sizeof(int)) 범위를 벗어날 경우 continue로 입력부터 다시 받도록 한다. 

```c++
while (true)
{
    ...
    value &= ~(0xff << ((index - 1) * 8));
    value |= input << ((index - 1) * 8);
    ...
}
```
255는 unsigned 바이트가 가질 수 있는 최대 크기로, 비트로 나타내면 11111111이다. 11111111을 index 크기만큼 L shift 한 뒤, not - and를 통해 덮어쓰려는 영역의 bit 값을 0으로 초기화한다. 덮어쓰려는 영역은 0과 그 외의 영역은 1과 and 연산이 되기 때문에 그 외의 영역은 유지한 채 덮어쓰려는 영역만 초기화할 수 있게 된다. 그 후, char (0~255)로 받은 input 값을 index 크기만큼 밀어서 value와 or 연산을 한다. 그러면 다른 영역은 유지한 채, 원하는 영역의 bit 값만 input 값으로 바꿀 수 있다.

```c++
while (true)
{
    ...
    for (char i = 0; i < sizeof(value); i++)
    {
        printf("%u 번째 바이트 값 : %u \n", i+1, static_cast<unsigned char>(value >> (i * 8)));
    }

    printf("\n전체 4바이트 값 : 0x%08x \n\n", value);
	
}
```

반복문을 돌며 R Shift한 후, unsigned char 크기에 맞게 캐스팅 하여 각 바이트의 값을 출력한다. value가 기본적으로 unsigned int 형이기 때문에 unsigned char로 캐스팅 하지 않으면 255를 초과하는 값이 나올 수 있다. 마지막으로 %p 형식 지정자를 통해 value를 16진수로 출력해준다. 만약 000000A2라는 값이 있을 경우, 그냥 %x(16진수 형식 지정자)를 쓰면 A2가 출력되므로 앞의 빈 공간도 확인하고 싶다면 08x를 사용하여 전체를 출력하는 것이 좋다. 

```c++
printf("\n전체 4바이트 값 : 0x%p \n\n", value);
```
%p(포인터 형식 지정자)를 써도 000000A2로 출력된다. 그러나 %p는 포인터를 출력하는 것이 본래 목적이기 때문에 %08x를 사용하는 것이 더 적합하다.  

결과는 아래와 같이 출력된다. 

![cmd](https://user-images.githubusercontent.com/96677719/212537368-ea2bc7e7-7778-4100-b178-e50dfaf3fce3.png)

<br>

## **과제를 통해 배운 것**

**1. cin vs scanf**

scanf는 형식 지정자를 통해 입력받을 데이터의 형식을 지정해주는 반면, cin은 컴파일러가 변수 타입을 알아서 판단하여 입력받는다. 또, cin은 LF, whitespace를 무시하지만 scanf는 이를 거르지 않고 모두 입력으로 받는다. 결정적으로, cin, cout은 C 라이브러리의 stdio buffer과 동기화 하는 과정이 포함되며 iostream, stdio의 버퍼를 모두 사용하기 때문에 delay가 발생하게 된다. 이러한 이유들로 일반적으로는 prinf, scanf가 cout, cin에 비해 2배 이상 빠르다. 

```c++
std::ios::sync_with_stdio(false);
std::ios_base::sync_with_stdio(false);
std::cin.sync_with_stdio(false); //std::cout.sync_with_stdio(false);
```
cin, cout을 사용하기 전에 위와 같은 함수를 사용해주면 C의 stdio와의 동기화를 끊고 C++만의 독립적인 버퍼를 생성하여 속도가 향상된다. 대신 C의 버퍼와 병행하여 사용할 수 없게 된다. 

코드 출처: https://jm19.tistory.com/4

<br/>

**2. scanf vs scanf_s**

scanf는 지정된 버퍼 크기보다 더 많은 양의 문자를 넣을 수 있기 때문에 Buffer Overflow에 취약하다. 이를 보안한 것이 scanf_s로, 문자나 문자열을 입력받는 경우 인자값으로 크기를 입력해주어야 한다. 

<br/>

**3. nand/nor**

이전에 전공 수업 때 배웠던 nand 연산에 대해 잊고 있었는데, 이 과제를 구현하고 나서 보니 내가 nand 연산을 썼음을 발견하고 다시 떠올리게 되었다. nand, nor 의 진리표는 아래와 같다. 

![image](https://user-images.githubusercontent.com/96677719/212537913-f7eaabbb-495c-4178-a080-5843df1ae0d4.png)

![image](https://user-images.githubusercontent.com/96677719/212537922-0a96be2a-9bad-467b-b251-dcaf535cfc58.png)

사진 출처: 두산백과

<br/>

## **질의응답**

**1) char 자료형에 정수 받기**

Q. 불필요한 공간을 줄이고 싶어서 index, input등 255 이하의 값만 받을 변수는 char로 선언했습니다. 그런데 scanf로 char 자료형에 정수를 받으려고 scanf_s("%u", &index)와 같이 입력했더니 버퍼 언더런 또는 오버런이 발생할 수 있다며 경고를 띄워주었습니다. 이런 위험성을 없애려면 어떻게 수정해야 할까요?

A. %%hhu를 사용하면 unsigned char을 표현할 수 있습니다.

<br/>

## **유의사항**

**1) bit 연산 문제**

만약 회사에서 시험을 보는데 2진수를 출력해야 하는 문제가 나온다면 대부분의 경우 bit 연산을 기본적으로 할 수 있는지 확인하기 위해 내는 것이니 소인수분해나 포인터로 해결하기보다는 bit 연산으로 해결하는 것이 좋다. 

<br/>

**2) 연산자 우선 순위**

```c++
value = value | input << (index - 1) * 8;
```
위와 같이 사용하면 경고를 띄워준다. Shift 연산자의 경우 덧셈, 뺄셈보다도 우선순위가 낮아서 의도하지 않은 결과가 나올 수 있기 때문이다. 

```c++
value = value | input << ((index - 1) * 8);
```

연산자 우선순위가 매우 명확한 게 아닌 이상 되도록 모든 상황에서 위처럼 우선순위를 명시해주는 것이 좋다. 