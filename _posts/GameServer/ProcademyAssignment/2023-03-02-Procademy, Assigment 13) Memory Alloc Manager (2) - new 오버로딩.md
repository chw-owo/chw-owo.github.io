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
#include <stdio.h>
#include <stdlib.h>
#include <stringapiset.h>


#undef new
void* operator new(size_t size, const char* filename, int line)
{
	void* ptr = malloc(size);
	
	AllocInfo info;
	info._use = true;
	info._ptr = ptr;
	info._size = size;
	strcpy_s(info._filename, NAME_SIZE, filename);
	info._line = line;
	info._array = false;
	g_AllocMgr.PushInfo(info);

	printf("new completely!\n");
	return ptr;
}

void* operator new[](size_t size, const char* filename, int line)
{
	void* ptr = malloc(size);

	AllocInfo info;
	info._use = true;
	info._ptr = ptr;
	info._size = size;
	strcpy_s(info._filename, NAME_SIZE, filename);
	info._line = line;
	info._array = true;
	g_AllocMgr.PushInfo(info);

	printf("new[] completely!\n");
	return ptr;
}

void operator delete (void* p, char* file, int line)
{

}
void operator delete[](void *p, char* file, int line)
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
		g_AllocMgr.WriteLog("ARRAY", arrP, info);
		free(arrP);
		return;
	}	
	
	g_AllocMgr.WriteLog("NOALLOC", p, info);
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
		g_AllocMgr.WriteLog("NOARRAY", notArrP, info);
		free(notArrP);
		return;
	}

	g_AllocMgr.WriteLog("NOALLOC", p, info);
}
```

**AllocMgr.h**

```c++
#pragma once
using namespace std;

#define INFO_SIZE 128
#define NAME_SIZE 256
#define file_SIZE 1024

struct AllocInfo
{
	bool _use = false;
	void* _ptr = nullptr;
	int _size = 0;
	char _filename[NAME_SIZE] = { '\0', };
	int _line = 0;
	bool _array = false;
};

class AllocMgr
{
public:
	~AllocMgr();
	void PushInfo(AllocInfo info);
	bool FindInfo(void* ptr, AllocInfo* info = nullptr);
	void WriteLog(const char* msg, 
		void* ptr, const AllocInfo& info);
	void MakeLogfile(char data[file_SIZE]);
	
private:
	int _infoCnt = 0;
	AllocInfo _allocInfos[INFO_SIZE];
};
extern AllocMgr g_AllocMgr;
void SetFileName(char* filename);
```

**AllocMgr.cpp**

```c++
#include "AllocMgr.h"
#include "stdio.h"
#include <windows.h>
#include <fileapi.h>
#include <time.h>

using namespace std;
#define NUM 8
#define OUTPUT_NAME_SIZE 32
#define TIME_BUF 128

AllocMgr g_AllocMgr;
AllocMgr::~AllocMgr()
{
	char buf[file_SIZE] = { '\0', };
	char data[file_SIZE] = { '\0', };
	char msg[] = "LEAK";

	int infoIdx = 0;
	int bufIdx = 0;
	int dataIdx = 0;

	for (int i = 0; i < _infoCnt;)
	{
		if (_allocInfos[infoIdx]._use)
		{
			sprintf_s(buf, "%s [%p] [%d] %s : line is %d\n",
				msg, _allocInfos[infoIdx]._ptr, _allocInfos[infoIdx]._size,
				_allocInfos[infoIdx]._filename, _allocInfos[infoIdx]._line);
			
			bufIdx = 0;
			while (buf[bufIdx] != '\n')
			{
				data[dataIdx] = buf[bufIdx];
				bufIdx++;
				dataIdx++;
			}
			data[dataIdx] = '\n';

			dataIdx++;
			i++;
		}	
		infoIdx++;
	}
	MakeLogfile(data);
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

void AllocMgr::WriteLog(const char* msg, void* ptr, const AllocInfo& info)
{
	char data[file_SIZE] = { '\0', };

	if (msg == "NOALLOC" || msg == "NOALLOC[]" )
		sprintf_s(data, "%s [%p]\n", msg, ptr);
	else
		sprintf_s(data, "%s [%p] [%d] %s : line is %d\n",
			msg, ptr, info._size, info._filename, info._line);

	MakeLogfile(data);
}

void AllocMgr::MakeLogfile(char data[file_SIZE])
{
	time_t timer = time(NULL);
	struct tm date;
	char buf[TIME_BUF] = { '\0', };
	localtime_s(&date, &timer);

	char filename[OUTPUT_NAME_SIZE] = { '\0', };
	sprintf_s(filename, "Alloc_%04d%02d%02d_%02d%02d%02d_*.txt",
		date.tm_year + 1900, date.tm_mon + 1,
		date.tm_mday, date.tm_hour, date.tm_min, date.tm_sec);

	FILE* file;
	SetfileName(filename);

	errno_t ret = fopen_s(&file, filename, "w");
	if (ret != 0)
		printf("Fail to open %s : %d\n", filename, ret);
	fwrite(data, strlen(data) + 1, 1, file);
	fclose(file);
}

void SetfileName(char* filename) 
{
	int nameIdx = 0;
	WIN32_FIND_DATA findData;
	HANDLE hfile;
	hfile = FindFirstFile((LPCWSTR)filename, &findData);
	
	if (hfile != INVALID_HANDLE_VALUE)
		nameIdx++;
		
	while (FindNextFile(hfile, &findData))
		nameIdx++;

	FindClose(hfile);

	char filenameIdx[NUM] = { '\0', };
	sprintf_s(filenameIdx, "%d.txt", nameIdx);

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
