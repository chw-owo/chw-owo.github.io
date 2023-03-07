---
title: Memory 02) Virtual Memory
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 시스템 정보**

```c++
VOID GetSystemInfo(LPSYSTEM_INFO psi);
```
이를 호출하면 시스템 구성 정보에 대한 값들을 psi에 반환한다. 이때, SYSTEM_INFO 구조체는 페이지 크기, 가용 최소/최대 주소, 예약 단위 크기, CPU 총 개수, 사용 가능한 CPU, 프로세서 아키텍쳐, 프로세서 레벨 등에 대한 정보를 포함한다. 현재 설치된 프로세스에 대해 더 자세한 내용을 얻고 싶다면 GetLogicalProcessorInformation을 활용할 수 있다. 

특정 프로세스가 어디에서 수행되는지 알고 싶다면 Is~Process 함수를 호출할 수 있으나, 이는 실제 머신의 정보 대신 에뮬레이션 상태의 시스템 정보를 반환한다. GetNativeSystemInfo를 호출하면 에뮬레이션 상태의 시스템 정보가 아니라 실제 머신의 시스템 정보를 얻을 수 있다. 

<br/>

# **2. 가상 메모리 상태**

윈도우는 가상 메모리 현재 상태에 대한 동적 정보를 획득할 수 있도록 GlobalMemoryStatus 함수를 제공한다. 이때 MEMORYSTATUS 구조체의 주소를 인자로 전달하는데, 맨 앞 dwLength 인자에 구조체 크기를 넣어서 초기화하도록 하고 있다. 만약 4GB 이상 램을 갖고 있거나 전체 스왑 파일의 크기가 4GB 이상이라면 GlobalMemoryStatusEx, MEMORYSTATUSEX를 사용해야 한다. 

이때 만약 NUMA 머신에서 GlobalMemoryStatusEx 함수를 사용하면 모든 노드의 가용 메모리 합을 나타내게 된다. 따라서  특정 NUMA 노드 내 가용 메모리만을 얻고 싶다면 GetNumaAvailableMemoryNode를 대신 호출해야 한다. 

**+)** NUMA, Non-Uniform Memory Access 머신

NUMA는 다른 노드의 메모리에도 접근이 가능한 머신으로, 그럼에도 자기 노드 메모리에 접근하는 것이 월등히 빠르다. 따라서 기본적으로는 CPU와 동일 노드 램만을 물리적 저장소로 커밋하려고 하며 공간이 충분하지 않은 경우 다른 노드의 램을 사용한다. 

<br/>

# **3. 주소 공간의 상태 확인**

```c++
DWORD VirtualQuery (
    LPCVOID pvAddress,
    PMEMORY_BASIC_INFORMATION pmbi,
    DWORD dwLength
);
```
```c++
DWORD VirtualQueryEx (
    HANDLE hProcess
    LPCVOID pvAddress,
    PMEMORY_BASIC_INFORMATION pmbi,
    DWORD dwLength
);
```

윈도우는 프로세스 내 특정 메모리에 대해 크기, 형태, 보호 특성 등과 같은 정보들을 얻을 수 있도록 VirtualQuery 함수를 제공한다. MEMORY_BASIC_INFORMATION 구조체는 전달 주소의 페이지 단위 내림 주소, 전달 주소의 포함 영역 시작 주소, 최초 예약시 보호 특성, 동일한 특성과 페이지 형태를 가진 인접 페이지 크기, 인접 페이지 상태, 인접 페이지 보호 특성, 인접 페이지의 물리적 저장소 형태 등에 대한 정보를 포함한다. 

```c++
BOOL VMQuery(
    HANDLE hProcess,
    LPCVOID pvAddress,
    PVMQUERY pVMQ
);
```
간단한 정보만 필요하다면 VirtualQuery로도 충분하지만 예약된 영역의 전체 크기, 영역 내 블록 개수, 스레드 스택 포함 여부 등이 필요할 경우 VMQuery 함수를 쓰는 것이 더 유용하다. 물론 이 함수는 VirtualQuery에 비해 더 느리게 동작하기 때문에 꼭 필요한 상황이 아니라면 권장하지 않는다. VMQuery를 이용해 프로세스 내 주소 공간을 살펴보는 애플리케이션을 만들 수 있다. 그 예제는 아래와 같다. 

```c++
BOOL bOk = TRUE;
PVOID pvAddress = NULL;

...

while(bOk){

    VMQUERY vmq;
    bOk = VMQuery(hProcess, pvAddress, &vmq);

    if(bOk) 
    {
        TCHAR szLine[1024];
        ConstructRgnInfoLine(hProcess, &vmq, szLine, sizeof(szLine));
        ListBox_AddString(hWndLB, szLine);

        if(bExpandRegions)
        {
            for(DWORD dwBlock = 0; bOk && (dwBlock < vmq.dwRgnBlocks); dwBlock++)
            {
                ConstructBlkInfoLine(&vmq, szLine, sizeof(szLine));
                ListBox_AddString(hWndLB, szLine);

                pvAddress = ((PBYTE)pvAddress + vmq.BlkSize);
                if(dwBlock < vmq.dwRgnBlocks - 1)
                    bOk = VMQuery(hProcess, pvAddress, &vmq);
            }
        }
        pvAddress = ((PBYTE)vmq.pvRgnBaseAddress + vmq.RgnSize);
    }
}
```

이 루프는 NULL 가상 주소로부터 시작하여 프로세스의 주소 공간에 더 이상 살펴볼 영역이 없을 때까지 VMQuery를 반복한다. 매 루프마다 영역에 대한 세부정보를 이용해 문자 버퍼를 구성하는 ConstructRgnInfoLine, ConstructBlkInfoLine 함수를 호출하며 이렇게 구성한 문자 버퍼를 리스트박스에 추가한다. 