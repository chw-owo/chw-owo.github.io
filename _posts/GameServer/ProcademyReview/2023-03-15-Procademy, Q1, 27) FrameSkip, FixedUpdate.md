---
title: Procademy, Q1, 27) FrameSkip, FixedUpdate
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. FrameSkip**

컴퓨터가 권장 프레임을 따라가지 못하면 화면이 끊기더라도 플레이는 동일하게 할 수 있도록 맞춰줘야 한다. 만약 권장 50fps, 최소 20fps인 게임에서 컴퓨터 사양이 20fps이라면 렌더는 20번만 돌리더라도 로직은 50번을 돌리기 위해 렌더를 스킵할 것이다. 이를 두고 프레임 스킵이라고 부른다. 일반적으로는 렌더가 cpu를 제일 오래 쓰기 때문에, 렌더 직전에 fps를 체크하여 이를 바탕으로 FrameSkip 여부를 결정한다.

FrameSkip을 구현할 때, 만약 권장 시간보다 일찍 끝난 경우 그 격차를 구해 Sleep으로 맞춰준다. 오래 걸린 경우 일반적으로는 느려지는대로 그냥 누적하다가, 누적된 것이 1프레임을 넘어가게 되면 그때 스킵하는 방식을 사용한다. 만약 한번에 여러 프레임이 스킵될 만큼 시간이 오래 걸렸다면, 걸린 시간에서 최소 fps만큼 차감하다가 그 결과가 음수가 되는 시점에서 다시 이전처럼 동작하게 한다. 

최근의 3D 게임들은 프레임 개념을 사용하지 않는다. 관절에 큰 변화가 있는 지점만 키프레임으로 잡은 뒤 중간 과정은 시간차 보간을 통해 구현한다. 컴퓨터의 성능이 좋으면 자연스러운 모션이 나오고 안좋으면 움직임이 다소 단절되어서 보일 것이다. 이동을 구현하는 것도 마찬가지로 속도값을 바탕으로 좌표 보간을 이용한다. 따라서 이런 경우에는 프레임 단위로 굳이 계산할 필요가 없다.

유니티는 프레임을 직접 사용하는 대신, Time.deltaTime으로 구한 전 프레임과 현 프레임의 시간차를 사용한다. 따라서 서버에서 클라의 프레임을 고려할 필요는 없으며, 서버에서도 좌표 계산 등을 구현할 때 이와 같은 방식을 사용할 것이다. MO 게임 정도는 락-스택을 이용해 사용자들의 로직 프레임을 맞추는 것이 가능하지만 더 규모가 커지면 이는 불가능해진다. 

<br/>

# **2. FixedUpdate**

Time.deltaTime이 매우 큰 값이 나오면 벽을 뚫는 문제가 생길 수도 있는데, 이를 막기 위해 FixedUpdate 방식을 사용한다. FixedUpdate도 최소 프레임의 로직을 보장하기 위한 것으로, 일반적으로 프레임 방식에서는 FrameSkip을, 시간 방식에서는 FixedUpdate를 사용한다. Unity도 내부적으로 FixedUpdate 방식을 사용하며, 공식 문서에서 물리 처리는 FixedUpdate로 처리하기를 권장하고 있다.

FrameSkip은 렌더를 생략하여 로직의 최소 프레임을 보장한다면, FixedUpdate는 fps를 초과한 경우 그만큼 Update를 연달아 호출하여 이를 보장한다. 이때, FixedUpdate에서 Time.deltaTime을 호출하면 실제 소요 시간과 상관 없이 늘 지정된 최소 시간 이상의 값이 나온다. 따라서 연속적으로 FixedUpdate가 호출되어도 Time.deltaTime으로 0이 나오지 않고 정상적인 이동이 보장된다. 

<br/>

# **3. FPS 측정 구현**

```c++
#include <iostream>
#include <Windows.h>

// loop를 돌때마다 FPS를 구한다.
void FPS()
{
	static int Frame;
	static DWORD oldTick = timeGetTime();
	Frame++;

	if (timeGetTime() - oldTick > 1000)
	{
		printf("FPS: %d\n", Frame);
		oldTick += 1000; 
		Frame = 0;
	}
}

void Update()
{
	int delay = rand() % 10;
	if (GetAsyncKeyState(VK_RETURN))
		delay += 5;
	Sleep(delay);
}

void Render()
{
	int delay = rand() % 10;
	if (GetAsyncKeyState(VK_RETURN))
		delay += 15;
	Sleep(delay);
}

int main()
{
	timeBeginPeriod(1);
	DWORD curTick = 0;
	DWORD oldTick = timeGetTime();
	DWORD deltaTick = 0;

	while (1)
	{
		Update();
		
		FPS();
		curTick = timeGetTime();
		deltaTick = curTick - oldTick;

		if(deltaTick < 20)
			Sleep(20 - deltaTick);

		oldTick += 20;

		Render();
	}
}
```

의도대로면 빠르게 동작할 시 일정 시간만큼 Sleep을 걸기 때문에 FPS가 일관적으로 나와야 한다. 그러나 실제로는 불규칙하게 값이 떨어지는 것을 볼 수 있다. 여기에는 여러가지 원인이 있는데 우선 Sleep 자체가 정교하게 동작하지 않기 때문이다. Sleep은 스레드 함수라 호출만으로도 15-16을 잡아먹는다. 

또, oldTick에 값을 직접 더하는 대신 oldTick = timeGetTime(); 를 할 수도 있는데, 이 경우 값이 밀리면서 점점 오차가 커진다. 실제 얼마나 걸렸는지에 상관 없이 1000, 20이 정확하게 흘렀다고 가정해야 이후 계산이 밀리지 않기 때문에 위처럼 정해진 값을 직접 더하는 것이 좋다. 

정교한 시간 값을 측정할 경우 분기 하나, 더하기 하나도 시간에 영향을 많이 미친다. 데이터를 시간에 맞춰 전송하는 등의 프로그램을 만들 때 이를 고려하지 않으면 데이터가 비거나 밀릴 수 있다. 따라서 오차 발생 요인들에 대해 잘 고려하여 설계해야 한다.

<br/>

# **4. FrameSkip 구현**

```c++
#include <iostream>
#include <Windows.h>

int g_logic;
int g_render;

// loop를 돌때마다 FPS를 구한다.
void FPS()
{
	static DWORD oldTick = timeGetTime();

	if (timeGetTime() - oldTick > 1000)
	{
		printf("FPS: logic %d / render %d\n", g_logic, g_render);
		oldTick = timeGetTime();
		g_logic = 0;
		g_render = 0;
	}
}

void Update()
{
	g_logic++;
	int delay = rand() % 10;
	if (GetAsyncKeyState(VK_RETURN))
		delay += 5;
	Sleep(delay);
}

void Render()
{
	g_render++;
	int delay = rand() % 20;
	if (GetAsyncKeyState(VK_RETURN))
		delay += 15;
	Sleep(delay);
}

bool Skip()
{
	static DWORD oldTick = timeGetTime();

	DWORD curTick = 0;
	DWORD deltaTick;

	curTick = timeGetTime();
	deltaTick = curTick - oldTick;
	oldTick += 20;

	if (deltaTick < 20)
	{
		Sleep(20 - deltaTick);
		return true;
	}

	return false;
}

int main()
{
	timeBeginPeriod(1);

	while (1)
	{
		Update();
		
		FPS();

		if(Skip())
			Render();
	}
}
```

```c++
curTick = timeGetTime();
deltaTick = curTick - oldTick;
oldTick += 20;
```
이 부분을 통해 얼마나 시간이 초과했는지 자동으로 구해지기 때문에 별도의 변수에 밀린 시간을 누적하지 않아도 된다. 또, 로직은 계속 돌아가되 렌더는 돌아가지 않아야 하므로 이 둘을 분리하여 따로 계산하였다. 이렇게 되면 블락이 걸려도 풀린 직후 멈췄던 만큼 연속으로 돌아가면서 FPS를 일정하게 맞추게 된다. FixedUpdate도 내부적으로 유사하게 구성되어있다.

<br/>

# **5. 서버에서의 FrameSkip**

클라는 로직과 렌더가 나누어지고, 렌더가 cpu를 많이 차지하기 때문에 이런 작업이 필요하다. 반면 서버는 스킵해도 되는 후순위 작업이랄 게 거의 없다. 그래서 서버에서는 프레임이 떨어지면 그냥 떨어지도록 냅두게 된다. 이처럼 프레임을 맞춰줄 수가 없으므로 애초에 프레임 단위의 로직이 아니라 시간 단위의 로직을 짜야한다.

물론 CPU 낭비를 줄이기 위해 최대 프레임은 제한하겠지만 최소 프레임은 제한할 수 없다. 즉, 빠를 때 느리게 하는 로직은 들어가지만, 느려졌을 때 빠르게 돌아가는 것은 서버의 특성상 할 수 없는 것이다. 실제로 최대 동접자를 잡을 때도 일반적으로 약간 느려져도 정상적인 플레이는 가능한 수준으로만 잡게 된다.