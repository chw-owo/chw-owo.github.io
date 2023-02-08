---
title: Procademy, Assignment 04) 난수 생성기 02
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

## **과제**

랜덤값을 생성할 때 rand를 그대로 사용할 경우, 확률적으로는 맞지만 유저 입장에서 불만이 생길 수 있다. 10%의 확률로 추첨을 했을 때, 100번을 뽑는다고 해서 10번이 당첨되는 것은 아니기 때문이다. 랜덤을 사용하되 확률을 정확하게 맞추기ㅡ, ,546  위해서, 즉 100번 뽑았을 때 정확히 10번 나오는 프로그램을 설계하기 위해서는 조금의 장치가 필요하다. 이 포스트에서는 당첨될 때마다 당첨 항목의 확률을 낮추고, 그 총합이 0이 될 때 원본 확률로 다시 초기화 하는 방법을 사용하였다. 

엄밀히 말하면 과제는 아니지만 연습 삼아 한번씩 구현해보기를 권장하셨던 주제라 작성해보았다. 

<br/>

## **과제 코드**

```c++
#include <iostream>
#include <conio.h>
#include <windows.h>
#include <tchar.h>

#define CNT 7

struct st_ITEM
{
	char	Name[30];
	int		Rate;	// 아이템이 뽑힐 비율
};

st_ITEM g_Gatcha[CNT] = {
			{"칼",		20},
			{"방패",		20},
			{"신발",		20},
			{"물약",		20},
			{"초강력무기",	5},
			{"초강력방패",		5},
			{"초강력레어레어레어신발", 1}
};

void Gatcha()
{
    // 바뀐 확률 값을 기록하기 위한 배열
	int rate[CNT] = { };
	int rateAccum[CNT] = {};

	int total = 0;
	int iRand;
	int idx;
	int i;

	while (1)
	{
		if (_kbhit())
		{
			//logic section
			_getch();

            // total이 0이 되었을 경우 
            // rate, idx를 원본 값으로 초기화
			if (total <= 0)
			{
				for (i = 0; i < CNT; i++)
				{
					rate[i] = g_Gatcha[i].Rate;
				}
				idx = 0;
			}

            // 당첨으로 인해 변한 확률값을 반영한다
			total = 0;
			for (i = 0; i < CNT; i++)
			{
				total += rate[i];
				rateAccum[i] = total;
			}

            // 새롭게 추첨
			iRand = rand() % (total + 1);

            // 방금전 추첨으로 
            // 당첨된 항목의 확률값을 낮춘다
			for (i = 0; i < CNT; i++)
			{
				if (iRand <= rateAccum[i])
				{
					rate[i]--;
					break;
				}
			}

			//render section       
			printf("%d : %s (origin: %d / left: %d)\n", idx, g_Gatcha[i].Name, g_Gatcha[i].Rate, rate[i]);
			idx++;
		}
	}
}

int _tmain(int argc, _TCHAR* argv[])
{
	Gatcha();
	return 0;
}

```

결과는 아래와 같이 나온다. 난수를 이용하면서도 정해진 횟수(확률)만큼 정확하게 나오도록 설계된 것을 확인할 수 있다. 

![cmd](https://user-images.githubusercontent.com/96677719/217446069-0564cac5-9116-4201-8f92-9be455a0fbca.png)

