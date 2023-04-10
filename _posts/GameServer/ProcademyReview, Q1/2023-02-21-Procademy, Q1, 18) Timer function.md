---
title: Procademy, Q1, 18) Timer function
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1.  clock**

```c++
clock_t t1 = clock();
DWORD t2 = clock();
```
clock_t는 C에서 제공해주는 자료형으로 typedef unsigned longfh 로 정의되어 밀리세컨드 단위의 시간을 저장할 수 있다. 

# **2.GetTickCount**

```c++
clock_t t1 = GetTickCount();
DWORD t2 = GetTickCount();
```
GetTickCount()은 GetTickCount는 시스템이 부팅된 순간을 기준으로 흐른 시간을 clock_t 타입으로 반환한다. 이는 시스템을 재부팅 하지 않는 이상, 프로그램을 껐다 켠다고 해도 계속 유지된다. 따라서 쿨타임을 구할 때 아래와 같이 활용할 수 있다. 

```c++
DWORD dwSkilllTick = GetTickCount();

if(GetTickCount() - dwSkillTick >= 3000)
{
	// TO-DO
}
```

이때 부팅한 순간으로부터 49.7일이 되면 long의 최대값을 초과하면서 0으로 돌아가버려서 의도하지 않은 결과가 나오게 된다. 따라서 위 함수를 쓸 때는 예외처리를 해주어야 한다. 최근에는 이에 대한 보완책으로는 나온 GetTickCount64() 를 주로 사용한다. 이는 Unsigned Long Long이기 때문에 내가 죽기 전에 값이 초과할 일은 없다. 


# **3. timeGetTime**

시간을 구하는 또 다른 함수로는 timeGetTime이 있다. 이 역시 동일한 long type으로 49.7일에 대한 문제가 존재한다. 그러나 차이점이라면, GetTickCount와 달리 64 버전이 없으며 기본적으로 GetTickCount()가 timeGetTick()보다 훨씬 가볍다. timeGetTick()은 해당 값을 time interrupt에서 직접 읽어와 전역 변수에 계속해서 갱신을 해주는 반면 GetTickCount()은 어딘가에 복사된 time 값을 읽어오기만 하기 때문이다. 이는 디스어셈블리를 보면 확인할 수 있다.

그러나 고해상도의 타이머를 원한다면 timeGetTime을 사용하는 것이 좋다. timeGetTime은 windows 기본 라이브러리가 아니라 멀티미디어 용도로 제공되는 라이브러리로서 더 해상도 높은 타이머로 설정할 수 있기 때문이다. 이를 제어하는 함수가 timeBeginPeriod() - timeEndPeriod()이며 아래와 같이 사용한다. 

```c++
timeBeginPeriod(1); // 시간 해상도를 1ms 단위로 설정

// TO-Do 

timeEndPeriod(1); // 해제. 기본 설정값으로 되돌림
```
위와 같은 방법으로 시간 해상도를 1ms로 높일 수 있다. 이때, 만약 내가 5로 설정했다면 다른 프로세스에서 이보다 작은 값인 1로 바꿀 수는 있으나 큰 값인 10으로 바꿀 수는 없다. 더 높은 해상도를 설정하면 대기 함수에서 보다 정확한 간격을 사용할 수 있지만, 스레드 스케줄러가 작업을 더 자주 전환하기 때문에 전체 시스템 성능에 부담이 갈 수 있다. 하지만 타이머 주기가 짧아진다고 해도 체크하는 시간 단위가 짧아지는 것이지 1ms 단위로 매번 컨텍스트 스위칭을 한다는 의미는 아니다. 따라서 정확한 시간 값이 중요한 작업에서는 이를 1ms로 설정하기를 권장한다. 


# **4. _rdtsc**

우리가 이용할 수 있는 가장 고해상도의 타이머는 cpu의 clock이다. clock을 직접 얻어내서 타이머를 만드는 방법은 다음과 같다. 

```c++
int64 tscS = __rdtsc();
```
이는 인라인 어셈에서 제공되는 것으로, 함수 호출하듯이 쓰지만 실제로는 인라인 처리 된다. 32bit일 때는 레지스터 2개를, 64bit을 땐 1개를 이용하여 실제 현재 cpu clock을 저장한다. 

```c++
int64 tscS = __rdtsc();
for(;;)
{
	// timeGetTime 반복문으로 1초 count
	// sleep 보다 이게 더 정확함
}
int64 elapsed = __rdtsc() - tscS;
printf(“%d”, …);
```

정밀하게 하고 싶다면 이렇게 clock수를 구할 수도 있으나, 그 정도로 정밀한 계산이 필요한 상황이라면 이것도 부정확한 정보가 될 수도 있다. 이는 멀티 프로세스 환경의 한계이므로 어느정도 감안하고 사용해야 한다. 또, _rdtsc는 시간 값을 반환하는 게 아니라 clock 수를 반환하므로 하드웨어마다 값이 달라지는 것을 고려해야 한다. 한 하드웨어 내에서 영역 별 성능 비교에는 적합하나 여러 하드웨어에서 시간을 얻어내야 하는 경우 부적합할 수 있다. 

멀티코어 초창기에는 코어마다 clock이 다르게 나오는 문제 때문에 cpu 선호도를 지정하여 코드가 한 cpu에서만 돌아가도록 지정해야 했다. 현재는 2000년대 초반에 clock이 모든 코어에서 통일되도록 변경되어서 이런 문제가 생기지 않는다. 


# **5. QueryPerformancCount**

```c++
LARGE_INTEGER Start;
LARGE_INTEGER End;
LARGE_INTEGER Freq;
QueryPerformanceFrequency(&Freq);	// 1초의 진동주기

QueryPerformanceCounter(&Start);
Sleep(1000);
QueryPerformanceCounter(&End);

int64 elapsed = (End.QuadPart - Start.QuadPart) / Freq.QuadPart;
```
QueryPerformanceFrequency, QueryPerformanceCounter는 clock을 사용하되, rdtsc 보다 해상도를 낮춘 함수이다. 이들은 union을 이용해 8byte 변수를 구현한 LARGE_INTEGER 구조체를 반환한다. clock 값은 하드웨어마다 달라질 수 있다는 한계가 있기 때문에, 이를 해결하기 위해 QueryPerformanceFrequency를 이용한다. QueryPerformanceFrequency를 통해 내 하드웨어의 1초 진동 주기를 얻어온 후 이를 이용해 QueryPerformanceCounter의 결과 값을 나누면 보다 정확한 시간 간격을 구할 수 있다.

그러나 윈도우8 이후에서는 어떤 하드웨어에서든 QueryPerformanceFrequency가 1000만이, QueryPerformanceCounter가 1/1000만이 나오도록 통일되었다. 저 함수 자체가 rdtsc를 바탕으로 1/1000만으로 통일, 환산하여 반환해주는 것이다. 윈도우 OS는 100ns(1/1000만s)를 최소 단위의 시간으로 사용하는 곳이 많은데, 이와 통일한 것으로 보인다. 시간이 통일 되었다고 하더라도 이전처럼 Frequency와 Counter를 함께 사용하는 것은 정해진 규칙이며, 하위 호환성 및 이후 업데이트를 고려하여 그대로 따르는 것이 좋다. 

# **6. Chrono**

```c++
    chrono::system_clock::time_point start 
                                    = chrono::system_clock::now();

    for(;;)
    {
        // TO-Do
    }

    chrono::duration<double>sec = std::chrono::system_clock::now() - start;
```

chrono는 C++11에서 추가되어 C 언어 차원에서 제공하는 함수로, 내부적으로 보면 OS에서 제공하는 clock 관련 함수를 랩핑한 것이다. 

# **6. 질의 응답**

Q. 왜 GetTickCount은 프로그램 시작 시점이 아니라 시스템 부팅 시점에서부터 측정하나요?

A. 어떻게 시간을 측정하는지 원리를 알면 이해할 수 있다. 

시간을 측정하는 건 소프트웨어가 하는 일이 아니라 하드웨어가 하는 일이며, 하드웨어의 신호를 받아서 소프트웨어가 그에 맞춰 갱신하는 것이다. timeGetTime보다 더 고해상도 용으로 제공되는 타이머의 경우 CPU clock을 사용하며 timeGetTime, GetTickCount의 경우 CPU 안에 내장된 Timer Interrupt를 사용한다. 

CPU가 명령어를 처리하다가 Interrupt가 발생하면 해당 Interrupt를 처리하기 위해 처리하던 것을 중단한다. 이때, Interrupt 사이에도 우선 순위 번호가 매겨지는데 그 중 0번이 Timer Interrupt이다. 이러한 Timer Interrupt가 시간을 체크하고 주기적으로 신호를 보내는 역할을 한다. 옛날에는 Timer가 메인보드에 있었는데 최근에는 CPU 내부로 들어왔다. 이 Timer가 시스템 부팅 시에 시작하기 때문에 기준이 이렇게 잡힌 것이다.

<br/>

Q. QueryPerformance 함수를 두고 GetTickCount를 사용해야 하는 상황이 있나요?

A. 고해상도 타이머가 필요없는 상황이라면 GetTickCount64 쪽이 더 가볍고 성능이 조금 더 좋다. 단순한 컨텐츠 로직을 짠다면 GetTickCount64로도 충분하다. 하지만 요즘은 정확도를 위해 웬만해서는 QueryPerformance 함수를 사용하는 경우가 많다. 

<br/>

Q. QueryPerformance 함수와 TimeGetTime 중에선 어떤 게 더 성능이 좋나요?

A. 그 둘을 비교한 적은 없지만 TimeGetTime도 가벼운 함수는 아니다. 게다가 32bit 라는 한계가 있기 때문에 TimeGetTime을 쓸 거면 QueryPerformance 함수를 쓰는 게 낫다.

<br/>

Q. 전역 변수도 초기화를 꼭 해야되나요?

A. 전역에 두면 초기화를 안해도 0이 들어가니까 영 하기 싫으시면 안하셔도 된다.
