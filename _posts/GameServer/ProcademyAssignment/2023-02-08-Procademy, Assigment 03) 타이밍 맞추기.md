---
title: Procademy, Assignment 03) 타이밍 맞추기
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---
## **과제**

**0.** 초에 해당하는 값을 배열에 넣어두고, 이 시간을 타이밍 데이터로 쓴다. 이때, 뒤로 갈 수록 큰 값을 넣는걸 원칙으로 한다.

**1.** 화면 상단에 시간이 표시된다. (초:밀리세컨드. 00:000  으로 자리수 맞춤)

**2.** 아래에는 각 키 타이밍의 정보와 해당 타이밍에 성공여부 결과를 표시한다.

**3.** 아무런 키를 누르지 않고 지정 시간을 1초 이상 넘으면 자동으로 fail 처리한다.

**4.** 사용자가 키를 누르면 해당 시간을 체크하여 오차 범위에 따라서 지정 타이밍의 결과가 화면에 표시된다.

<br/>

## **과제 코드**

```c++
#include <iostream>
#include <conio.h>
#include <windows.h>

#define CNT 9
int g_Timing[CNT] = { 5, 10, 14, 17, 20, 25, 29, 31, 33 };

void TimingGame()
{
    float fLeftTime;
    DWORD dwStartTime = timeGetTime();
    float error[CNT] = { -1, -1, -1, -1, -1, -1, -1, -1, -1 };
    int idx = 0;

    while (1)
    {
        //logic section 
        fLeftTime = (timeGetTime() - dwStartTime) / (float)CLOCKS_PER_SEC;

        if (fLeftTime >= g_Timing[idx] + 1 && error[idx] == -1)
        {
            error[idx] = 1;
            idx++;
        }           

        if (idx >= CNT)
            break;
           
        if (_kbhit())
        {
            _getch();
            error[idx] = abs(fLeftTime - g_Timing[idx]);
            idx++;        
        }

        //render section
        system("cls");
        printf("%.3f sec\n\n", fLeftTime);
        for (int i = 0; i < CNT; i++)
        {
            if(error[i] == -1)
                printf("%u sec:\n", g_Timing[i]);

            else if(error[i] >= 0 && error[i] < 0.25)
                printf("%u sec: Great!\n", g_Timing[i]);

            else if (error[i] >= 0.25 && error[i] < 0.5)
                printf("%u sec: Good!\n", g_Timing[i]); 

            else if (error[i] >= 0.5 && error[i] < 0.75)
                printf("%u sec: Nogood\n", g_Timing[i]);

            else if (error[i] >= 0.75 && error[i] < 1)
                printf("%u sec: Bad\n", g_Timing[i]);

            else if (error[i] >= 1)
                printf("%u sec: Fail\n", g_Timing[i]);
        } 
    }
}

int main()
{
    timeBeginPeriod(1);
    TimingGame();
    timeEndPeriod(1);
}
```

