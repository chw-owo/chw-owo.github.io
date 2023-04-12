---
title: Procademy, Assignment 09) Profiler for Single Thread
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

# **과제**

**-**  QueryPerformanceCounter를 사용하여 지정한 영역의 성능을 출력해주는 프로파일러를 만든다

**-**  싱글스레드 환경을 가정하고 제작한다. 후에 멀티스레드 환경에서도 동작하도록 수정할 예정

**-**  지정한 영역의 이름, 평균 성능, 최단 성능, 최장 성능, 해당 영역의 호출 횟수를 출력하여 txt 파일로 저장한다. 

<br/>

# **과제 해설**

지금 작성한 코드는 Begin 이후 break, return 등이 나타나서 End 가 나타나지 않았을 때를 대비한 안전장치가 없다. 또, 싱글스레드에서만 동작하도록 구현한 상태이다. 따라서 Begin - End의 짝이 맞지 않거나 멀티스레드일 경우 성능이 엉망진창으로 측정될 것이다. 이 부분은 추후 멀티스레드 용 프로파일러를 만들 때 플래그를 세우거나, 시간이 0이면 그 항목은 측정을 스킵하거나, 혹은 생성자 소멸자를 활용하는 등의 방식으로 보완할 것이다. 

+) 인라인 어셈을 사용할 수 있던 시절에는 예외 처리를 위해 Interrupt 3을 많이 사용했다고 한다. 요즘은 인라인 어셈이 안되므로 그 대신 nullptr을 찌르는 함수를 만들어뒀다가 호출하는 방식을 대신 사용하는 경우가 많다. 이후에 예외처리 핸들러를 두어 메모리 덤프를 저장하는 것도 구현할 예정. 

<br/>

# **과제 코드**

**Profile.h**
```c++
#pragma once
#include <windows.h>

#define PROFILE

#ifdef PROFILE
#define PRO_BEGIN(TagName)	ProfileBegin(TagName)
#define PRO_END(TagName)	ProfileEnd(TagName)
#define PRO_OUT(FileName)	ProfileDataOutText(FileName)
#define PRO_RESET()			ProfileReset()
//#define PRO_DEBUG()			PrintDataForDebug()
#else
#define PRO_BEGIN(TagName)
#define PRO_END(TagName)
#define PRO_OUT(TagName)	
#define PRO_RESET()			
#endif

struct _PROFILE_RESULT;
void ProfileBegin(const WCHAR* szName);
void ProfileEnd(const WCHAR* szName);
void ProfileDataOutText(const WCHAR* szFileName);
void ProfileReset(void);
//void PrintDataForDebug(void);

```

**Profile.cpp**
```c++
#include "Profile.h"
#include <iostream>
#include <float.h>

#define NAME_LEN 64
#define PROFILE_CNT 32
#define MINMAX_CNT 2
#define BUFFER_SIZE 256
#define OUTPUT_SIZE BUFFER_SIZE * PROFILE_CNT
#define NS_PER_SEC 1000

LARGE_INTEGER freq;
struct _PROFILE_RESULT{
	long			lFlag = 0;						// 프로파일의 사용 여부
	WCHAR			szName[NAME_LEN] = { '/0', };	// 프로파일 샘플 이름

	LARGE_INTEGER	lStartTime;						// 프로파일 샘플 실행 시간

	double			dTotalTime = 0;					// 전체 사용시간 카운터 Time
	double			dMin[MINMAX_CNT] = { DBL_MAX, DBL_MAX };	// 최소 사용시간 카운터 Time
	double			dMax[MINMAX_CNT] = { 0, 0 };	// 최대 사용시간 카운터 Time

	__int64			iCall = 0;						// 누적 호출 횟수

} PROFILE_RESULT[PROFILE_CNT];

void ProfileBegin(const WCHAR* szName)
{
	int i = 0;
	for (; i < PROFILE_CNT; i++)
	{		
		if (PROFILE_RESULT[i].lFlag != 0 &&
			wcscmp(PROFILE_RESULT[i].szName, szName) == 0)
		{
			_PROFILE_RESULT& pf = PROFILE_RESULT[i];
			pf.lFlag = 2;
			QueryPerformanceCounter(&pf.lStartTime);
			return;
		}
		
		else if (PROFILE_RESULT[i].lFlag == 0) 
		{
			_PROFILE_RESULT& pf = PROFILE_RESULT[i];
			pf.lFlag = 1;

			wcscpy_s(pf.szName, NAME_LEN, szName);
			QueryPerformanceCounter(&pf.lStartTime);
			QueryPerformanceFrequency(&freq);

			pf.dTotalTime = 0;

			for (int min = 0; min < MINMAX_CNT; min++)
				pf.dMin[min] = DBL_MAX;
			for (int max = 0; max < MINMAX_CNT; max++)
				pf.dMax[max] = 0;

			pf.iCall = 0;
			return;
		}	
	}	

	if (i == PROFILE_CNT)
	{
		wprintf(L"profiler is full!: %s\n", szName);
		return;
	}
}

void ProfileEnd(const WCHAR* szName)
{
	int i = 0;
	for (; i < PROFILE_CNT; i++)
	{
		if (PROFILE_RESULT[i].lFlag != 0 &&
			wcscmp(PROFILE_RESULT[i].szName, szName) == 0)
		{
			_PROFILE_RESULT& pf = PROFILE_RESULT[i];

			LARGE_INTEGER endTime;
			QueryPerformanceCounter(&endTime);

			double result = 
				(endTime.QuadPart - pf.lStartTime.QuadPart)
												/ (double)freq.QuadPart;

			pf.dTotalTime += result;

			for (int min = 0; min < MINMAX_CNT; min++)
			{
				if (pf.dMin[min] > result)
				{
					if (min == MINMAX_CNT - 1)
					{
						pf.dMin[min] = result;
					}
					else
					{
						int tmpMin = min;
						while (tmpMin < MINMAX_CNT - 1)
						{
							pf.dMin[tmpMin + 1] = pf.dMin[tmpMin];
							tmpMin++;
						}
						pf.dMin[min] = result;
					}
				}
			}

			for (int max = 0; max < MINMAX_CNT; max++)
			{
				if (pf.dMax[max] < result)
				{
					if (max == MINMAX_CNT - 1)
					{
						pf.dMax[max] = result;
					}
					else
					{
						int tmpMax = max;
						while (tmpMax < MINMAX_CNT - 1)
						{
							pf.dMax[tmpMax + 1] = pf.dMax[tmpMax];
							tmpMax++;
						}
						pf.dMax[max] = result;
					}
				}
			}
			pf.iCall++;
			return;
		}
	}

	if (i == PROFILE_CNT)
	{
		wprintf(L"Can't find the profiler: %s\n", szName);
		return;
	}
}

void PrintDataForDebug()
{
	printf(
		"\n=====================================\n"
		"| Name | Average | Min | Max | Call |\n"
		"=====================================\n");

	int idx = 0;
	while (PROFILE_RESULT[idx].lFlag != 0)
	{
		_PROFILE_RESULT& pf = PROFILE_RESULT[idx];
		printf(
			"| %ls | %.4lfμs | %.4lfμs | %.4lfμs | %lld |\n",
			pf.szName,
			(pf.dTotalTime / pf.iCall) * NS_PER_SEC,
			pf.dMin[0] * NS_PER_SEC,
			pf.dMax[0] * NS_PER_SEC,
			pf.iCall);
		idx++;
	}

	printf("=====================================\n\n");
}

void ProfileDataOutText(const WCHAR* szFileName)
{
	char data[OUTPUT_SIZE] =
		"\n=====================================\n"
		"| Name | Average | Min | Max | Call |\n"
		"=====================================\n" ;

	int idx = 0;
	char buffer[BUFFER_SIZE];
	while (PROFILE_RESULT[idx].lFlag != 0)
	{
		_PROFILE_RESULT& pf = PROFILE_RESULT[idx];
		memset(buffer, '\0', BUFFER_SIZE);
		sprintf_s(buffer, BUFFER_SIZE,
			"| %ls | %.4lfμs | %.4lfμs | %.4lfμs | %lld |\n",
			pf.szName, 
			(pf.dTotalTime / pf.iCall) * NS_PER_SEC,
			pf.dMin[0] * NS_PER_SEC, 
			pf.dMax[0] * NS_PER_SEC, 
			pf.iCall);
		strcat_s(data, OUTPUT_SIZE, buffer);
		idx++;
	}

	strcat_s(data, OUTPUT_SIZE, 
		"=====================================\n\n");

	FILE* file;
	errno_t ret;
	ret = _wfopen_s(&file, szFileName, L"wb");
	if (ret != 0)
		wprintf(L"Fail to open %s : %d\n", szFileName, ret);
	fwrite(data, strlen(data), 1, file);
	fclose(file);

	printf("Profile Data Out Success!\n");
}

void ProfileReset(void)
{
	for (int i = 0; i < PROFILE_CNT; i++)
	{
		memset(&PROFILE_RESULT[i], 0, sizeof(_PROFILE_RESULT));
	}	
	printf("Profile Reset Success!\n");
}
```

**Test.cpp**

테스트용 코드 (Assignment 6 알파블렌딩 과제 활용)

```c++
#include <windows.h>
#include <stdio.h>
#include "Profile.h"

void AlphaBlending()
{
	int size;
	char fileName1[] = "sample.bmp";
	char fileName2[] = "sample2.bmp";
	char resultFileName[] = "result.bmp";

	errno_t ret;
	FILE* buffer1 = nullptr;
	FILE* buffer2 = nullptr;
	FILE* result = nullptr;
	BITMAPFILEHEADER bf;
	BITMAPINFOHEADER bi;

	// 1. Read Bitmap File

	// 1) Get File Header
	PRO_BEGIN(L"Get File");
	ret = fopen_s(&buffer1, fileName1, "rb");
	if (ret != 0)
		printf("Fail to open %s : %d\n", fileName1, ret);

	fread(&bf, sizeof(bf), 1, buffer1);
	fread(&bi, sizeof(bi), 1, buffer1);
	size = (bi.biWidth * bi.biHeight * bi.biBitCount) / 8;

	char* pImageBuffer1 = static_cast<char*>(malloc(size));
	char* pImageBuffer2 = static_cast<char*>(malloc(size));
	char* pImageBlend = static_cast<char*>(malloc(size));

	// 2) Get File1 Data
	fread(pImageBuffer1, size, 1, buffer1);
	fclose(buffer1);
	PRO_END(L"Get File");

	// 3) Get File2 Data
	ret = fopen_s(&buffer2, fileName2, "rb");
	if (ret != 0)
		printf("Fail to open %s : %d\n", fileName2, ret);

	fread(pImageBuffer2, size, 1, buffer2);
	fclose(buffer2);

	// 2. Write File data

	// 1) Get Blending Data
	PRO_BEGIN(L"Blend Data");
	DWORD* pDest = (DWORD*)pImageBlend;
	DWORD* pSrc1 = (DWORD*)pImageBuffer1;
	DWORD* pSrc2 = (DWORD*)pImageBuffer2;

	for (int h = 0; h < bi.biHeight; h++)
	{
		for (int w = 0; w < bi.biWidth; w++)
		{
			*pDest = ((*pSrc1 / 2) & 0x7f7f7f7f) + ((*pSrc2 / 2) & 0x7f7f7f7f);
			pDest++;
			pSrc1++;
			pSrc2++;
		}
	}
	PRO_END(L"Blend Data");

	// 2) Write Bmp File
	PRO_BEGIN(L"Write File");
	ret = fopen_s(&result, resultFileName, "wb");
	if (ret != 0)
		printf("Fail to open %s : %d\n", resultFileName, ret);

	fwrite(&bf, sizeof(bf), 1, result);
	fwrite(&bi, sizeof(bi), 1, result);
	fwrite(pImageBlend, size, 1, result);
	fclose(result);
	PRO_END(L"Write File");

	free(pImageBlend);
	free(pImageBuffer1);
	free(pImageBuffer2);
}

int main()
{
	for(int i = 0; i < 50; i++)
		AlphaBlending();
	PRO_OUT(L"output.txt");
	return 0;
}
```
 
**출력 결과 txt**
```
=====================================
| Name | Average | Min | Max | Call |
=====================================
| Get File | 7.5521μs | 6.3974μs | 26.5566μs | 50 |
| Open File | 0.7496μs | 0.0876μs | 27.1379μs | 150 |
| Get BMP Header | 0.0188μs | 0.0143μs | 0.0301μs | 50 |
| Get BMP Data | 1.6972μs | 1.4676μs | 3.1787μs | 100 |
| Close File | 0.0845μs | 0.0122μs | 0.4546μs | 150 |
| Blend Data | 6.0610μs | 5.9425μs | 6.5638μs | 50 |
| Write File | 3.6454μs | 2.8572μs | 29.1461μs | 50 |
=====================================
```