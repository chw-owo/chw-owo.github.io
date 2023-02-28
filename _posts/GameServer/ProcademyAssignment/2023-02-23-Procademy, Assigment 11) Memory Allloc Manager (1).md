---
title: Procademy, Assignment 11) Memory Alloc Manager (1)
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

# **과제**

**-** C++  템플릿 함수를 이용해서 new / delete 메모리 추적기의 간단한 버전을 만들어본다.

**-** 사용 예시

```c++
int *p4 = _MemoryAlloc(int, 1);
int *p400 = _MemoryAlloc(int, 100);
char *pZ1 = _MemoryAlloc(char, 50);
char *pZ2 = _MemoryAlloc(char, 150);
char *pZ3 = _MemoryAlloc(char, 60);
char *pZ4 = _MemoryAlloc(char, 70);

MemoryRelease(p4);
MemoryRelease(p400);
MemoryRelease(pZ1);
MemoryRelease(pZ2);
MemoryRelease(pZ3);
MemoryRelease(pZ4);

PrintAlloc();   //메모리 미해제 내역 출력
```

**-** 출력 결과

```
Total Alloc Size : 734
Total Alloc Count : 6
Not Release Memory : [0x5fe060] 400 Bytes
File : new_log.cpp : 194

Not Release Memory : [0x5fe238] 150 Bytes
File : new_log.cpp : 197
```

<br/>

# **과제 코드**

**MemoryAllocMgr.h**
```c++
#pragma once
#include <vector>
using namespace std;

#define ALLOC_MGR
#define FILE_NAME 256

#ifdef ALLOC_MGR
#define _MemoryAlloc(type, count)	MemoryAlloc<type>(count, __LINE__, __FILE__)
#else
#define _MemoryAlloc			
#endif

// AllocInfo =============================================

class AllocInfo
{
public:
	AllocInfo(void* ptr, int size, int line, const char* filename);
	bool operator==(void* ptr);

public:
	void* _ptr = nullptr;
	int _size = 0;
	int _line = 0;
	char _filename[FILE_NAME] = { '\0', };
}; 

struct TotalAllocInfo
{
	int size = 0;
	int count = 0;
};

extern TotalAllocInfo g_totalAllocInfo;
extern vector<AllocInfo> allocInfos;

// Memory Alloc Function =============================================

template<typename T>
T* MemoryAlloc(int cnt, int line, const char* filename)
{
	T* ptr = new T[cnt];

	AllocInfo* info = new AllocInfo(ptr, sizeof(T) * cnt, line, filename);
	allocInfos.push_back(*info);

	return ptr;
}

template<typename T>
void MemoryRelease(T ptr)
{
	vector<AllocInfo>::iterator it;

	// 없는 주소를 지우려고 했을 때 안내
	it = find(allocInfos.begin(), allocInfos.end(), ptr);
	if (it == allocInfos.end())
	{
		printf("it's not allocated data!\n");
		return;
	}
	
	//AllocInfo* info = &(*it);
	allocInfos.erase(it);
	it = allocInfos.begin();
	//delete info;
	delete ptr;
}

void PrintAlloc(void);

```

**MemoryAllocMgr.cpp**
```c++
#include "AllocMgr.h"
#include <stdio.h>

TotalAllocInfo g_totalAllocInfo;
vector<AllocInfo> allocInfos;

AllocInfo::AllocInfo(void* ptr, int size, int line, const char* filename)
{
	this->_ptr = ptr; 
	this->_size = size;
	this->_line = line;
	strcpy_s(_filename, FILE_NAME, filename);
	g_totalAllocInfo.size += size;
	g_totalAllocInfo.count++;
}

bool AllocInfo::operator==(void* ptr)
{
	return(this->_ptr == ptr);
}

void PrintAlloc(void)
{
	printf("--------------------------------------\n\n");

	printf("Total Alloc Size : %d\n", g_totalAllocInfo.size);
	printf("Total Alloc Count : %d\n\n", g_totalAllocInfo.count);

	vector<AllocInfo>::iterator it;
	
	for (it = allocInfos.begin(); it != allocInfos.end(); it++)
	{
		printf("Not Release Memory : [%p] %d Bytes\n", (*it)._ptr, (*it)._size);
		printf("File : %s : %d\n\n", (*it)._filename, (*it)._line);
	}

	printf("--------------------------------------\n");
}
```

<br/>

# **질문**

template를 사용하면 함수 선언 뿐 아니라 정의도 헤더파일에서 해주어야 하는 것으로 아는데 혹시 template을 쓰면서도 cpp와 header를 깔끔하게 분리할 수 있는 좋은 방법은 없는지...