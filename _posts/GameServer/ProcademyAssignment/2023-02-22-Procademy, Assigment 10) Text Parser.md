---
title: Procademy, Assignment 09) Profiler for Single Thread
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

## **과제**

**-**  QueryPerformanceCounter를 사용하여 지정한 영역의 성능을 출력해주는 프로파일러를 만든다

**-**  싱글스레드 환경을 가정하고 제작한다. 후에 멀티스레드 환경에서도 동작하도록 수정할 예정

**-**  지정한 영역의 이름, 평균 성능, 최단 성능, 최장 성능, 해당 영역의 호출 횟수를 출력하여 txt 파일로 저장한다. 

<br/>

## **과제 코드**

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
#include "FileIO.h"
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

	FILE* pFile;
	OpenFile(szFileName, &pFile, L"wb");
	WriteFile(&pFile, strlen(data), data);
	CloseFile(&pFile);

	wprintf(L"Profile Data Out Success!\n");
}

void ProfileReset(void)
{
	for (int i = 0; i < PROFILE_CNT; i++)
	{
		memset(&PROFILE_RESULT[i], 0, sizeof(_PROFILE_RESULT));
	}	
	wprintf(L"Profile Reset Success!\n");
}
```

**FileIO.h**
```c++
#pragma once
#include <windows.h>
#include <stdio.h>

void OpenFile(const WCHAR* filename, FILE** pFile, const WCHAR* mode);
void ReadFile(FILE** pFile, const int& size, char* data);
void WriteFile(FILE** pFile, const int& size, char* data);
void CloseFile(FILE** pFile);
int GetFileSize(FILE** pFile);
```

**FileIO.cpp**
```c++
#include "FileIO.h"
#include "Profile.h"

// File IO
void OpenFile(const WCHAR* filename, FILE** pFile, const WCHAR* mode)
{
	PRO_BEGIN(L"Open File");
	errno_t ret = _wfopen_s(pFile, filename, mode);
	if (ret != 0)
	{
		wprintf(L"Fail to open %s: %d\n", filename, ret);
	}
	PRO_END(L"Open File");
}

void CloseFile(FILE** pFile)
{
	PRO_BEGIN(L"Close File");
	errno_t ret = fclose(*pFile);
	if (ret != 0)
	{
		wprintf(L"Fail to close: %d\n", ret);
	}
	PRO_END(L"Close File");
}

void ReadFile(FILE** pFile, const int& size, char* data)
{
	fread(data, 1, size, *pFile);
	errno_t ret = ferror(*pFile);
	if (ret != 0)
	{
		wprintf(L"Fail to read data : %d\n", ret);
	}

}

void WriteFile(FILE** pFile, const int& size, char* data)
{
	fwrite(data, size, 1, *pFile);
	errno_t ret = ferror(*pFile);

	if (ret != 0)
	{
		wprintf(L"Fail to write data : %d\n", ret);
	}

}

int GetFileSize(FILE** pFile)
{
	fseek(*pFile, 0, SEEK_END);
	int size = ftell(*pFile);
	rewind(*pFile);

	if (size == -1)
	{
		wprintf(L"Fail to get file size\n");
	}

	return size;
}
```

**Test.cpp**
테스트용 코드 (Assignment 6 알파블렌딩 과제 활용)
```c++
#include <windows.h>
#include <stdio.h>
#include "Profile.h"
#include "FileIO.h"

void AlphaBlending();
void ReadBmpHeader(FILE** file, BITMAPFILEHEADER* bf, BITMAPINFOHEADER* bi);
void ReadBmpData(FILE** file, int size, char* output);
void WriteBmpFile(FILE** output, BITMAPFILEHEADER* bf, BITMAPINFOHEADER* bi, int size, char* data);

int main()
{
	for(int i = 0; i < 50; i++)
		AlphaBlending();
	PRO_OUT(L"output.txt");
	return 0;
}

void AlphaBlending()
{
	int size;
	WCHAR fileName1[] = L"sample.bmp";
	WCHAR fileName2[] = L"sample2.bmp";
	WCHAR resultFileName[] = L"result.bmp";

	FILE* buffer1 = nullptr;
	FILE* buffer2 = nullptr;
	FILE* result = nullptr;
	BITMAPFILEHEADER bf;
	BITMAPINFOHEADER bi;

	// 1. Read Bitmap File

	// 1) Get File Header
	PRO_BEGIN(L"Get File");
	OpenFile(fileName1, &buffer1, L"rb");
	ReadBmpHeader(&buffer1, &bf, &bi);
	size = (bi.biWidth * bi.biHeight * bi.biBitCount) / 8;

	char* pImageBuffer1 = static_cast<char*>(malloc(size));
	char* pImageBuffer2 = static_cast<char*>(malloc(size));
	char* pImageBlend = static_cast<char*>(malloc(size));

	// 2) Get File1 Data
	ReadBmpData(&buffer1, size, pImageBuffer1);
	CloseFile(&buffer1);

	// 3) Get File2 Data
	OpenFile(fileName2, &buffer2, L"rb");
	ReadBmpData(&buffer2, size, pImageBuffer2);
	CloseFile(&buffer2);
	PRO_END(L"Get File");

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
	OpenFile(resultFileName, &result, L"wb");
	WriteBmpFile(&result, &bf, &bi, size, pImageBlend);
	CloseFile(&result);
	PRO_END(L"Write File");

	free(pImageBlend);
	free(pImageBuffer1);
	free(pImageBuffer2);

}

void ReadBmpHeader(FILE** file, BITMAPFILEHEADER* bf, BITMAPINFOHEADER* bi)
{
	PRO_BEGIN(L"Get BMP Header");
	errno_t ret;

	fread(bf, sizeof(*bf), 1, *file);
	ret = ferror(*file);

	if (ret != 0)
		printf("Fail to read file header : %d\n", ret);

	fread(bi, sizeof(*bi), 1, *file);
	ret = ferror(*file);

	if (ret != 0)
		printf("Fail to read info header : %d\n", ret);
	PRO_END(L"Get BMP Header");
}

void ReadBmpData(FILE** file, int size, char* data)
{
	PRO_BEGIN(L"Get BMP Data");
	errno_t ret;
	fread(data, size, 1, *file);
	ret = ferror(*file);

	if (ret != 0)
		printf("Fail to read data : %d\n", ret);

	PRO_END(L"Get BMP Data");
}

void WriteBmpFile(FILE** output, BITMAPFILEHEADER* bf,
	BITMAPINFOHEADER* bi, int size, char* data)
{
	errno_t ret;

	fwrite(bf, sizeof(*bf), 1, *output);
	ret = ferror(*output);

	if (ret != 0)
		printf("Fail to write file header : %d\n", ret);

	fwrite(bi, sizeof(*bi), 1, *output);
	ret = ferror(*output);

	if (ret != 0)
		printf("Fail to write info header : %d\n", ret);

	fwrite(data, size, 1, *output);
	ret = ferror(*output);

	if (ret != 0)
		printf("Fail to write data : %d\n", ret);
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