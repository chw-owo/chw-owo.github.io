---
title: Procademy, Q1, 30) Window API - VirtualAlloc
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. VirtualAlloc**

```c++
p = (char*)VirtualAlloc((VOID*)0x00a00000, 4096 * 2, MEM_COMMIT, PAGE_READWRITE);
```
위 코드는 우연히 0x00a00000가 RESERVE 혹은 COMMIT이였던 경우가 아니라면 실패하며, VirtualAlloc은 실패할 경우 null pointer를 반환한다. 왜냐면 RESERVE가 되지 않은 곳을 COMMIT 할 순 없기 때문이다. 우연히 성공하더라도 의도된 공간이 아니기 때문에 위와 같은 사용은 잘못되었다고 볼 수 있다. 

```c++
p = (char*)VirtualAlloc(NULL,, 4096 * 2, MEM_COMMIT, PAGE_READWRITE);
```
포인터로 NULL을 넣으면 크기가 맞는 아무곳이나 할당해서 포인터를 던져준다. 따라서 이렇게 할 경우 RESERVE 되어있던 지점을 알아서 COMMIT 해주기 때문에 올바르게 동작한다. 성공하면 할당 받은 주소를 던져주는데, 이 주소를 바탕으로 직접 접근할 수 있다. 

```c++
SYSTEM_INFO sys;
GetSystemInfo(&sys);
```
GetSystemInfo를 통해 프로세스가 쓸 수 있는 주소 공간의 최소 최대, 한번에 reserve 걸 수 있는 최소 단위(dwAllocationGranularity) 등을 얻을 수 있다. dwAllocationGranularity의 경우 보통 64K가 기본값이며 이 값의 배수가 아닌 주소로 VirtualAlloc을 시도하면 실패한다. 

```c++
p1 = (char*)VirtualAlloc(p + 4096 * 2, 4096, MEM_RESERVE, PAGE_READWRITE);
```
```c++
p2 = (char*)VirtualAlloc(p + 4096 * 2, 4096, MEM_COMMIT, PAGE_READWRITE);
```
위와 같은 이유로 이 코드들은 둘 다 실패한다.

```c++
p = (char*)VirtualAlloc(NULL, 4096 * 2, MEM_COMMIT, PAGE_READWRITE);
```
이런 의미에서는 이 코드도 잘못된 코드라고 볼 수 있다. 64KB - (4096 * 2) 의 공간을 모두 낭비하게 되기 때문이다. 한번 저렇게 잡고 나면 해당 주소로 접근하여 예약 크기를 변경할 수도 없다. 크기를 변경하고 싶다면 RELEASE 한 후 다시 RESERVE - COMMIT을 해야 한다. 

```c++
bResult = VirtualFree(p + 4096, 0, MEM_RELEASE);
```
```c++
bResult = VirtualFree(p, 4096, MEM_RELEASE);
```
이 두가지 경우 모두 실패한다. 예약을 걸었던 시작 주소가 반드시 들어가야 하며, MEM_RELEASE는 size를 지정할 수 없다. 

```c++
bResult = VirtualFree(p, 0, MEM_RELEASE);
```
이렇게 해야만 해제할 수 있다. 참고로 DECOMMIT 플래그도 있는데 이는 COMMIT을 RESERVE로 돌리는거지 FREE가 되는 게 아니다. RELEASE를 해야만 FREE가 된다.

일반적인 사용법은 아니지만 임의로 메모리 속성을 지정할 때 이 함수를 사용할 수 있다. code 영역은 기본적으로 READONLY지만 VirtualAlloc을 통해 WRITE로 수정할 수 있다. 접근하는 순간 크래쉬를 내고 싶은 영역이 있다면 꼭 필요한 순간에만 해제하고 평소엔 PAGE_NOACCESS로 두는 방식으로 관리할 수도 있다. 

VMMap에서 프로세스를 살펴보면 메모리 영역 별 속성을 보여주는데, VirtualAlloc한 메모리는 스택도 힙도 아닌 Private Data로 속성이 정해진다. code, data, stack, heap 으로 나눈 메모리 영역은 우리가 필요로 하는 영역을 분류한 것뿐이지 메모리가 이 4가지로만 나뉘는 것은 아니다. OS가 알아서 저 네개로 분류되지 않는 메모리에 올리는 경우도 있다. 

<br/>

# **2. VirtualAlloc의 활용**

VirtualAlloc을 이용하여 디버깅용 overflow 체크 기능을 구현할 수 있다. 이를 위해서는 페이지 접근 권한을 설정해야 하는데 보통 최소 단위는 페이지 단위, 즉 4KB 단위이다. 따라서 내가 쓰려는 영역이 맨 뒤에 오도록 접근 가능 최소 4KB를 할당 받고, 바로 뒤에 접근 금지로 만들려는 영역의 페이지 시작부터 최소 4KB를 할당 받아서 접근 제한을 걸어야 한다. 

이 방법은 할당 받을 때마다 최소 8KB를 사용해야 되므로 낭비가 많이 생겨서 일반적으로 사용할 수는 없다. 계속 침범이 반복적으로 일어나는데 구체적인 원인을 찾을 수 없는 경우에 디버깅 목적으로 사용하는 것뿐이다. 물론 어떤 변수가 침범하고 있는지까지는 대략적으로 알아야 해당 변수 Alloc 부분을 교체하여 체크할 수 있다. 

VirtualAlloc에 인자로 전할 size는 bit 연산을 이용하여 구할 수 있다. 4096 단위로 내림을 한다면 12bit 마스크를 만들어서 하위 12bit를 날려버리면 되고, 이 경우처럼 올림을 해야한다면 1로 가득찬 12bit를 더한 뒤 not과 and 하면 그 단위로 올림이 될 것이다. 물론 그냥 산수로 계산할 수도 있다. 

```c++
char *p = (char*) Alloc_overflow_check(400);
for(int i = 0; i < 401; i++)
{
	p[i] = '0';
}
```
위와 같이 사용할 수 있도록 만드는 게 목표이다. 이때 경계에서 받은 사이즈만큼 앞 부분의 포인터를 반환해야 하는데, 기존의 HeapAlloc을 보면 64bit에서 malloc은 성능을 위해 16byte 경계에 주소를 잡아준다. 성능 상관 없이 디버깅이 무조건 우선이라면 경계를 무시할테고, 성능을 좀 더 고려한다면 16byte 경계에 맞춰서 건네줄 것이다. Align 인자를 넣어서 선택하도록 만들 수도 있다. 

Free_overflow_check 할 땐 Alloc_overflow_check에서 할당받은 메모리가 맞는지 확인해야 하는데, 이때는 자연적으로 나오기 어려운 값을 맨 앞에 넣어두고 체크하는 방식으로 확인할 수 있다. 성능을 고려하면 이런 단계가 없는 게 맞지만 디버깅 용이라면 안전을 위해 이렇게 사용할 수 있다.

