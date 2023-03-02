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
