---
title: Procademy, Assignment 13) Memory Alloc Manager (2) - new 오버로딩
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

# **과제**

## **new, delete 오버로딩 문법**

```c++
void * operator new (size_t size, char *File, int Line)
{
}

void * operator new[] (size_t size, char *File, int Line)
{
	ptr = malloc(size);
	..
	return ptr;
}

void operator delete (void * p, char *File, int Line)
{
}
void operator delete[] (void * p, char *File, int Line)
{
}

// 실제로 사용할 delete	
void operator delete (void * p)
{
}
void operator delete[] (void * p)
{
}
```

```c++
int *p = new("문자열", 숫자) int; 
char *p = new("문자열", 숫자) char[10]; 
char *p = new(__FILE__, __LINE__) char[10]; 
```

위 방식을 메크로를 사용하여 

```c++
int *p = new int;
```
형태로 사용할 수 있게끔 만든다. 

최종적으로 실제 new, delete 를 사용하는 프로젝트에서 "custom_new.h" 를 include 하는 것으로 모든 new 와 delete 가 추적되는 코드로 바뀔 수 있도록 설계해야 한다. 

<br/>

## **필수사항**

**1)** 메모리할당 관리 객체를 전역으로 만든다

**2)** new 를 오버로딩 하여 메모리 생성시 해당 정보를 저장한다. (메모리 할당 소스파일,라인번호 / 할당 포인터 / 배열형태 / 등...)

**3)** delete 를 오버로딩하여 메모리 삭제시 오류가 있는지 확인한다. (실제 할당된적이 있는 포인터인지, 배열형태는 맞는지 등...)

**4)** 3번에서 문제가 없었다면 할당내역을 삭제하고, 3번에서 오류가 났다면 로그를 저장한다.

**5)** 메모리관리 클래스의 파괴자에서 할당 후 파괴가 안된 메모리 로그를 저장한다.

<br/>

## **로그 파일**

NOALLOC - 메모리 해제시 할당되지 않은 포인터 입력.

ARRAY - new[] / delete[] 배열형태 생성과 삭제가 잘못됐을 때

LEAK  - 할당 후 해제되지 않은 메모리

```
NOALLOC [0x00008123] 
NOALLOC [0x00614134] 
NOALLOC [0x00614134] 
ARRAY   [0x00613ca0] [	   40] my_new\my_new.cpp : 350
LEAK    [0x00614130] [     44] my_new\my_new.cpp : 361 
LEAK    [0x00613ca0] [     40] my_new\my_new.cpp : 366 
```

localtime_s 함수를 사용하여 Alloc_YYYYMMDD_HHMMSS.txt 형태의 텍스트 파일로 저장하되 오류상황, [ 할당메모리 번지 ], [ 사이즈 ], 파일이름, 메모리 할당 줄이 줄 별로 출력되도록 한다.


# **과제 코드**

**main.cpp**

```c++
#include "custom_new.h"

int main()
{
	int* p4 = new int;
	int* p400 = new int[100];
	delete[](p400);
	char* pZ1 = new char[50];
	char* pZ2 = new char[150];
	delete[](pZ1);
	char* pZ3 = new char[200];

	//delete(p4);
	delete[](p400);
	delete[](pZ2);
	//delete[](pZ3);
}
```
일부러 no alloc, leak 메세지가 남도록 작성한 메인 코드.

**custom_new.h**

```c++
#pragma once

void* operator new(size_t size, const char* File, int Line);
void* operator new[](size_t size, const char* File, int Line);
#define new new(__FILE__, __LINE__)		

void operator delete (void* p, char* File, int Line);
void operator delete[](void* p, char* File, int Line);
void operator delete (void* p);
void operator delete[](void* p);
```

**custom_new.cpp**

```c++
#include "custom_new.h"
#include "AllocMgr.h"

#include <Windows.h>
#include <stringapiset.h>
#include <stdio.h>
#include <stdlib.h>

#undef new
void* operator new(size_t size, const char* File, int Line)
{
	void* ptr = malloc(size);
	TCHAR filename[FILE_NAME] = { '\0', };
	MultiByteToWideChar(CP_ACP, MB_PRECOMPOSED,
		File, strlen(File), filename, FILE_NAME);
	
	AllocInfo info;
	info._use = true;
	info._ptr = ptr;
	info._size = size;
	_tcscpy_s(info._filename, FILE_NAME, filename);
	info._line = Line;
	info._array = false;
	g_AllocMgr.PushInfo(info);

	printf("new completely!\n");
	return ptr;
}

void* operator new[](size_t size, const char* File, int Line)
{
	void* ptr = malloc(size);
	TCHAR filename[FILE_NAME] = { '\0', };
	MultiByteToWideChar(CP_ACP, MB_PRECOMPOSED,
		File, strlen(File), filename, FILE_NAME);

	AllocInfo info;
	info._use = true;
	info._ptr = ptr;
	info._size = size;
	_tcscpy_s(info._filename, FILE_NAME, filename);
	info._line = Line;
	info._array = true;
	g_AllocMgr.PushInfo(info);

	printf("new[] completely!\n");
	return ptr;
}

void operator delete (void* p, char* File, int Line)
{

}
void operator delete[](void *p, char* File, int Line)
{

}

void operator delete (void* p)
{
	if (g_AllocMgr.FindInfo(p))
	{
		free(p);
		printf("delete completely!\n");
		return;
	}

	AllocInfo info;
	void* arrP = reinterpret_cast<void*>
		(reinterpret_cast<int>(p) - sizeof(void*));

	if (g_AllocMgr.FindInfo(arrP, &info))
	{
		g_AllocMgr.WriteLog(_T("ARRAY"), arrP, info);
		free(arrP);
		return;
	}	
	
	g_AllocMgr.WriteLog(_T("NOALLOC"), p, info);
}

void operator delete[](void* p)
{
	if (g_AllocMgr.FindInfo(p))
	{
		free(p);
		printf("delete[] completely!\n");
		return;
	}
	
	AllocInfo info;
	void* notArrP = reinterpret_cast<void*>
		(reinterpret_cast<int>(p) + sizeof(void*));

	if (g_AllocMgr.FindInfo(notArrP, &info))
	{
		g_AllocMgr.WriteLog(_T("NOARRAY"), notArrP, info);
		free(notArrP);
		return;
	}

	g_AllocMgr.WriteLog(_T("NOALLOC"), p, info);
}
```

**AllocMgr.h**

```c++
#pragma once
#include <tchar.h>
#include <vector>
using namespace std;

#define INFO_SIZE 128

#define FILE_NAME 256
#define FILE_SIZE 1024

struct AllocInfo
{
	bool _use = false;
	void* _ptr = nullptr;
	int _size = 0;
	TCHAR _filename[FILE_NAME] = { _T('\0'), };
	int _line = 0;
	bool _array = false;
};

class AllocMgr
{
public:
	~AllocMgr();
	void PushInfo(AllocInfo info);
	bool FindInfo(void* ptr, AllocInfo* info = nullptr);
	void WriteLog(const TCHAR* msg, 
		void* ptr, const AllocInfo& info);
	void MakeLogFile(TCHAR data[FILE_SIZE]);
	
private:
	int _infoCnt = 0;
	AllocInfo _allocInfos[INFO_SIZE];
};

extern AllocMgr g_AllocMgr;
void SetFileName(TCHAR* filename);
```

**AllocMgr.cpp**

```c++
#include "AllocMgr.h"
#include "FileIO.h"
#include <fileapi.h>
#include <tchar.h>
#include <vector>
#include <time.h>
using namespace std;

#define NUM 8
#define OUTPUT_FILE_NAME 32
#define TIME_BUF 128

AllocMgr g_AllocMgr;
AllocMgr::~AllocMgr()
{
	TCHAR buf[FILE_SIZE] = { '\0', };
	TCHAR data[FILE_SIZE] = { '\0', };
	TCHAR msg[] = _T("LEAK");

	int infoIdx = 0;
	int bufIdx = 0;
	int dataIdx = 0;

	for (int i = 0; i < _infoCnt;)
	{
		if (_allocInfos[infoIdx]._use)
		{
			_stprintf_s(buf, _T("%s [%p] [%d] %s : line is %d\n"),
				msg, _allocInfos[infoIdx]._ptr, _allocInfos[infoIdx]._size,
				_allocInfos[infoIdx]._filename, _allocInfos[infoIdx]._line);
			
			bufIdx = 0;
			while (buf[bufIdx] != _T('\n'))
			{
				data[dataIdx] = buf[bufIdx];
				bufIdx++;
				dataIdx++;
			}
			data[dataIdx] = _T('\n');

			dataIdx++;
			i++;
		}	
		infoIdx++;
	}

	MakeLogFile(data);
}

void AllocMgr::PushInfo(AllocInfo info)
{
	_infoCnt++;
	for (int i = 0; i < _infoCnt; i++)
	{
		if (!_allocInfos[i]._use)
		{
			_allocInfos[i] = info;
			break;
		}
	}
}
bool AllocMgr::FindInfo(void* ptr, AllocInfo* info)
{
	int idx = 0;
	for (int i = 0; i < _infoCnt;)
	{
		if (_allocInfos[idx]._ptr == ptr)
		{
			_allocInfos[idx]._use = false;
			_infoCnt--;
			if(info != nullptr)
				info = &(_allocInfos[idx]);
			return true;
		}	
		
		if (_allocInfos[idx]._use)
			i++;
		idx++;
	}
	return false;
}

void AllocMgr::WriteLog(const TCHAR* msg, void* ptr, const AllocInfo& info)
{
	TCHAR data[FILE_SIZE] = { '\0', };

	if (msg == _T("NOALLOC") || msg == _T("NOALLOC[]") )
		_stprintf_s(data, _T("%s [%p]\n"), msg, ptr);
	else
		_stprintf_s(data, _T("%s [%p] [%d] %s : line is %d\n"),
			msg, ptr, info._size, info._filename, info._line);

	MakeLogFile(data);
}

void AllocMgr::MakeLogFile(TCHAR data[FILE_SIZE])
{
	time_t timer = time(NULL);
	struct tm date;
	TCHAR buf[TIME_BUF] = { _T('\0'), };
	localtime_s(&date, &timer);

	TCHAR filename[OUTPUT_FILE_NAME] = { _T('\0'), };
	_stprintf_s(filename, _T("Alloc_%04d%02d%02d_%02d%02d%02d_*.txt"),
		date.tm_year + 1900, date.tm_mon + 1,
		date.tm_mday, date.tm_hour, date.tm_min, date.tm_sec);

	char chData[FILE_SIZE];
	WideCharToMultiByte(CP_ACP, 0, data, FILE_SIZE,
		chData, FILE_SIZE, NULL, NULL);

	FILE* file;
	SetFileName(filename);
	OpenFile(filename, &file, _T("w"));
	WriteFile(&file, strlen(chData) + 1, chData);
	CloseFile(&file);
}

void SetFileName(TCHAR* filename) 
{
	int nameIdx = 0;
	WIN32_FIND_DATA findData;
	HANDLE hFile;
	hFile = FindFirstFile((LPCWSTR)filename, &findData);
	
	if (hFile != INVALID_HANDLE_VALUE)
		nameIdx++;
		
	while (FindNextFile(hFile, &findData))
		nameIdx++;

	FindClose(hFile);

	TCHAR filenameIdx[NUM] = { _T('\0'), };
	_stprintf_s(filenameIdx, _T("%d.txt"), nameIdx);

	int idx1 = 0;
	int idx2 = 0;
	while (filename[idx1] != '\0')
	{
		if (filename[idx1] == '*')
		{
			while (filenameIdx[idx2] != '\0')
			{
				filename[idx1] = filenameIdx[idx2];
				idx1++;
				idx2++;
			}
			break;
		}		
		idx1++;
	}	
}
```

**FileIO.h**

```c++
#pragma once
#include <iostream>
#include <TCHAR.h>
#include <windows.h>

//File IO
void OpenFile(const TCHAR* filename, FILE** file, const TCHAR* mode);
void ReadFile(FILE** file, const DWORD& size, char* data);
void WriteFile(FILE** file, const DWORD& size, char* data);
void CloseFile(FILE** file);
DWORD GetFileSize(FILE** file);
```

**FileIO.cpp**

```c++
#include "FileIO.h"

// File IO
void OpenFile(const TCHAR* filename, FILE** file, const TCHAR* mode)
{
	errno_t ret = _tfopen_s(&(*file), filename, mode);
	if (ret != 0)
		_tprintf(_T("Fail to open %s : %d\n"), filename, ret);
}

void CloseFile(FILE** file)
{
	errno_t ret = fclose(*file);
	if (ret != 0)
		_tprintf(_T("Fail to close file : %d\n"), ret);
}

void ReadFile(FILE** file, const DWORD& size, char* data)
{
	fread(data, size, 1, *file);
	errno_t ret = ferror(*file);

	if (ret != 0)
		_tprintf(_T("Fail to read data : %d\n"), ret);
}

void WriteFile(FILE** file, const DWORD& size, char* data)
{
	fwrite(data, size, 1, *file);
	errno_t ret = ferror(*file);

	if (ret != 0)
		_tprintf(_T("Fail to write data : %d\n"), ret);
}

DWORD GetFileSize(FILE** file)
{
	fseek(*file, 0, SEEK_END);
	LONG size = ftell(*file);
	fseek(*file, 0, SEEK_SET);

	if (size == -1)
		_tprintf(_T("Fail to get file size\n"));

	return size;
}
```