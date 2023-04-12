---
title: Procademy, Assignment 12) 다형성 실습 - 움직이는 별
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

# **과제**

순수 가상 함수와 다형성 실습을 위한 과제로, 3종의 클래스가 있지만 main 함수 쪽 에서는 어떤 클래스인지 알 필요 없이 오직 BaseObject 클래스만의 포인터를 다루며, 가상함수인 Update, Render 만을 호출하면 각 클래스가 알아서 자신이 해야 할 일을 하는 코드를 설계한다.


**-** 각각의 객체들은 X 축 0 좌표부터 시작하여 우측으로 이동을 한다.

**-** 우측의 끝은 X 축 75 위치로 하며, 끝에 닿는 객체는 파괴된다.

**-** 출력은 cout,printf 를 사용하여 X 좌표에 맞도록 스페이스를 넣어서 출력한다

**-** OneStar 는 * / TwoStar 는 ** / ThreeStar 는 *** 를 출력한다

**-** OneStar 는 1칸 / TwoStar 는 2칸 / ThreeStar 는 3칸을 이동한다

**-** Y 축은 없으며, 각 배열 위치에 따라서 출력한다 (배열 마다 줄바꿈)

<br/>

# **과제 기본 양식**

```c++
main()
{
	while ( 1 )
	{
		KeyProcess();

		Update();

		system("cls");	
		Render();	

		Sleep(30);
	}
}

Update()
{
	BaseObject 객체 배열 전체를 돌면서
	for ( 0 ~ Max )
	{
		Array[N]->Update();
	}

	// 우측에 닿으면 삭제를 해야하는데 이건 알아서 시도 해봄.
}

render()
{
	BaseObject 객체 배열 전체를 돌면서
	Array[N]->Render();
	
	// 각 배열에 객체가 있던 없던 줄 바꿈을 해주자.
	// 새로운 객체를 생성시 빈 배열을 찾아서 넣고 있는데
	// 비어있는 배열에 대해서 줄 바꿈을 안해주면 
	// 자꾸 줄이 바뀌어서 보기가 안좋음 !
}
```

**1)** BaseObject 포인터 배열 20개를 준비한다.

**2)** main에서 무한 루프로 KeyInput, Update, Render 돌아가게 설계한다.

**3)** 입력 멈춤이 없도록 숫자 키보드 입력을 체크한다 (_kbhit(), _getch() 사용 권장) 

**4)** 입력 받은 키를 바탕으로 1번:OneStar / 2번:TwoStar / 3번:ThreeStar 를 생성한다.

**5)** 번호에 맞는 객체를 동적 생성하여 빈 배열에 등록한다.

**6)** 메인 루프에서는 항시 배열의 모든 객체에 대하여 Update(), Render() 를 호출한다.

**7)** 각 클래스는 BaseObject 포인터에서 Update, Render를 호출하는 것만으로 각자의 일을 수행한다.
   
OneStar - 1칸씩 우측 이동 / *  
   
TwoStar - 2칸씩 우측 이동 / **  출력

ThreeStar - 3칸씩 우측 이동 / *** 출력

**8)** 우측 화면에 닿으면 해당 객체는 파괴된다. 이때, 동적할당을 해제할 때는 할당했던 함수와 같은 계층에서 처리하도록 하는 것이 좋다. 

**9)** 원활한 확인을 위해 메인 루프 하단에는 Sleep(10~50) 을 하여 속도를 늦춘다.   

<br/>

# **과제 코드**

**main.cpp**

```c++
#include <Windows.h>
#include <stdio.h>
#include <conio.h>
#include "MovingStars.h"

#define STAR_CNT 20
BaseObject* stars[STAR_CNT];

void KeyProcess();
void Update();
void Render();

int main()
{	
	while (1)
	{
		KeyProcess();
		Update();
		Render();
		Sleep(50);
	}
}

void KeyProcess()
{
	if (_kbhit())
	{
		char key = _getch();
		BaseObject* star = nullptr; 

		switch (key)
		{
		case '1':
			star = new OneStar;
			break;
		case '2':
			star = new TwoStar;
			break;
		case '3':
			star = new ThreeStar;
			break;
		default:
			return;
		}

		for (int i = 0; i < STAR_CNT; i++)
		{
			if (stars[i] == nullptr && star != nullptr)
			{
				star->Initialize();
				stars[i] = star;
				break;
			}
		}
	}
}

void Update()
{
	for (int i = 0; i < STAR_CNT; i++)
	{
		if (stars[i] != nullptr)
		{
			stars[i]->Update();
			if (stars[i]->CheckDead())
			{
				delete(stars[i]);
				stars[i] = nullptr;
			}
		}
	}
}

void Render()
{
	system("cls");
	for (int i = 0; i < STAR_CNT; i++)
	{
		if (stars[i] == nullptr)
			printf("\n");
		else
			stars[i]->Render();
	}
}
```

**MovingStars.h**

```c++
#pragma once
#include <stdio.h>
#include <stdlib.h>

#define ONE 1
#define TWO 2
#define THREE 3
#define MAX 75

class BaseObject
{
public:
	virtual void Initialize() = 0;
	virtual void Update() = 0;
	virtual void Render() = 0;
	virtual bool CheckDead() = 0;

public:
	bool _dead = false;
	int _x = 0;
	int _interval = 0;
	int _starSize = 0;
	char* _star;
	char _line[MAX + 1] = { '\0',};
};

class Star : public BaseObject
{
public:
	void Update();
	void Render();
	bool CheckDead();
};

class OneStar : public Star
{
public:
	void Initialize();
};

class TwoStar : public Star
{
public:
	void Initialize();
};

class ThreeStar : public Star
{
public:
	void Initialize();
};
```

**MovingStars.cpp**

```c++
#include "MovingStars.h"

char oneStar[ONE + 1] = "*";
char twoStar[TWO + 1] = "**";
char threeStar[THREE + 1] = "***";

void Star::Update()
{ 
	_x += _interval;

	if (_x + _starSize >= MAX)
	{
		_dead = true;
		return;
	}
		
	int i = 0;

	for (; i < _x; i++)
		_line[i] = ' ';

	for (int j = 0; j < _starSize; j++)
	{
		_line[i] = _star[j];
		i++;
	}

	for (; i < MAX; i++)
		_line[i] = ' ';

	_line[i] = '\n';
}

void Star::Render()
{
	if (_dead == true)
		return;

	for (int i = 0; i <= MAX; i++)
		printf("%c", _line[i]);
}

bool Star::CheckDead()
{
	return _dead;
}

void OneStar::Initialize()
{
	_interval = ONE;
	_starSize = ONE;
	_star = oneStar;
}

void TwoStar::Initialize()
{
	_interval = TWO;
	_starSize = TWO;
	_star = twoStar;
}

void ThreeStar::Initialize()
{
	_interval = THREE;
	_starSize = THREE;
	_star = threeStar;
}
```