---
title: Memory 03) Virtual Memory On App
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 가상 메모리 함수**

## **1) 가상 메모리**

윈도우는 메모리를 사용하는 방법으로 가상 메모리, 메모리 맵 파일, 힙을 제공한다. 이 중 가상 메모리는 크기가 큰 객체, 구조체의 배열 등을 관리하는 데에 최적인 방법이다. 가상 메모리 관리 함수를 사용하면 주소 공간 내 직접 영역을 예약하고, 물리적 저장소를 커밋하고, 필요한 보호 특성을 설정할 수 있다. 

## **2) 주소 공간에 영역 예약**

```c++
PVOID VirtualAlloc(
    PVOID pvAddress,
    SIZE_T dwSize,
    DWORD fdwAllocationType,
    DWORD fdwProtect
)
```
pvAddress에 예약하려는 주소를 전달하는데 NULL을 전달하면 프리 영역 중 적절한 공간이 할당된다. 이때 할당 단위 경계가 아닌 값을 전달하면 내림 주소로 할당이 되며 성공할 경우 영역의 시작 주소를 반환한다. dwSize에는 예약 크기를 바이트 단위로 전달하는데, CPU 페이지 크기의 배수로 지정해야 한다. 

fdwAllocationType는 예약과 커밋 중 무엇을 수행할지 지정하는 인자로, MEM_RESERVE를 전달하면 예약으로 수행된다. 또, 오랫동안 쓸 데이터는 MEM_TOP_DOWN를 전달하여 예약하면 가장 높은 주소 공간에 예약되므로 메모리 단편화 현상을 피할 수 있다. 

fdwProtect는 보호 특성 지정에 쓰이는데 일반적으로 예약 시와 커밋 시 보호 특성을 동일하게 지정하는 것이 더 효율적으로 동작한다. 이때 PAGE_WRITECOTY, PAGE_EXECUTE_WRITECOPY는 사용할 수 없으며 PAGE_GUARD, PAGE_NOCACHE, PAGE_WRITECOMBITE도 커밋시에만 사용이 가능하다. 

## **3) 예약 영역에 저장소 커밋**

예약된 영역에 접근하기 위해서는 페이징 파일로부터 물리적 저장소를 커밋해주어야 한다. 이때는 VirtaulAlloc을 MEM_COMMIT으로 호출하면 된다. 이때 예약했던 영역 전체에 대해 한번에 커밋할 필요는 없다. 만약 MEM_RESERVE | MEM_COMMIT으로 호출할 경우 한번에 예약과 커밋을 동시 수행할 수 있다.

보호 특성은 커밋을 시도한 전체 페이지에 지정되기 때문에, 여러 페이지에 대해 한번에 커밋을 시도하며 페이지별로 다른 보호 특성을 설정하는 것은 어렵다. 반면 동일한 영역 내라도 여러번에 걸쳐 커밋을 수행하면 페이지 별로 다른 보호 특성을 지정할 수 있다. 

## **4) 큰 페이지 할당**

큰 페이지 할당 기능을 적절하게 사용하면 성능을 개선할 수 있다. 윈도우는 이 기능으로 할당된 공간은 페이징 불가능 영역으로 설정하여 항상 내용이 램에 유지되도록 하기 때문이다. 큰 페이지 할당 기능을 사용하려면 dwSize는 GetLargePageMinimun 결과 값의 배수를, fdwAllocationType는 ( MEM_RESERVE | MEM_COMMIT | MEM_LARGE_PAGE )를, fdwProtect에는 PAGE_READWRITE를 전달해야 한다. 

이때, 이와 같은 페이지 잠금 기능은 일반 사용자 권한으로 할 수 없다. 따라서 관리 도구 - 보안 설정 - 로컬 정책 - 사용자 권한 할당 - 메모리 페이지 잠금 - 액션 - 속성 - 페이지 잠금 속성 - 사용자 또는 그룹 추가를 통해 권한을 할당해야 이 기능을 사용할 수 있다. 또 CPU가 이 기능을 제공하지 않는 경우도 있는데 이럴 땐 GetLargePageMinimun가 0을 반환하니 주의해야 한다. 

## **5) 디커밋과 영역 해제**

VirtualFree를 호출하되, pvAddress에는 영역 예약 시 VirtualAlloc이 반환했던 주소값을, dwSize에는 0을, fdwAllocationType에는 MEM_RELEASE를 전달할 경우 디커밋과 해제가 동시에 일어난다. 시스템은 해당 주소 영역의 크기를 이미 알고 있기 때문에 0이 아닌 다른 값을 전달하면 실패하며 일부 해제는 불가능하다. 

영역 해제는 하지 않고 디커밋만 수행하고 싶은 경우, 디커밋하고자 하는 첫번째 페이지의 주소 값과 해제하려는 메모리 크기, 그리고 MEM_DECOMMIT을 전달하면 된다. 만일 커밋 시작 주소와 0을 전달하면 해당 영역에 매핑된 모든 페이지에 대해 디커밋을 수행하게 된다.

디커밋 역시 페이지 단위로 수행되므로 전달한 주소 + 크기에 걸쳐 있는 페이지 전체에 대해 디커밋이 수행된다. 물리적 저장소 페이지가 디커밋되고 나면 이후 다른 프로세스들에 의해 다시 사용될 수 있으며, 이미 디커밋된 페이지에 커밋 없이 접근할 경우 마찬가지로 접근 위반이 유발된다. 

<br/>

# **2. 가상 메모리 할당-해제 프로그램**

## **1) Setting**
```c++
// 프로그램을 수행하는 머신의 페이지 크기
UINT g_uPageSize = 0;

// 배열을 구성하기 위한 더미 데이터 구조체
typedef struct {
    BOOL bInUse;
    BYTE bOtherData[2048 - sizeof(BOOL)];
} SOMEDATA, *PSOMEDATA;

// 배열 내 구조체 개수
#define MAX_SOMEDATA 50

// 구조체 배열을 가리키는 포인터
PSOMEDATA g_pSomeData = NULL;

// 창 내에서 Memory map에 의해 점유되는 직사각형 영역
RECT g_rcMemMap;

// 다이얼로그 박스 초기화
BOOL Dlg_OnInitDialog(HWND hWnd, HWND hWndFocus, LPARAM lParam);
INT_PTR WINAPI Dlg_Proc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam);

void Dlg_OnPaint(HWND hWnd);

void Dlg_OnDestroy(HWND hWnd)
{
    if(g_pSomeData != NULL)
        VirtualFree(g_pSomeData, 0, MEM_RELEASE);
}

int WINAPI _tWinMain(HINSTANCE hInstExe, HINSTANCE, PTSTR, int)
{
    // CPU 페이지 크기를 가져온다
    SYSTEM_INFO si;
    GetSystemInfo(&si);
    g_uPageSize = si.dwPageSize;
    DialogBox(hInstExe, MAKEINRESOURCE(IDD_VMALLOC), NULL, Dlg_Proc);

    return 0;
}
```

## **2) GarbageCollect**

```c++
VOID GarbageCollect(PVOID pvBase, DWORD dwNum, DWORD dwStructSize)
{
    UINT uMaxPage = dwNum * dwStructSize / g_uPageSize;
    for(UINT uPage = 0; uPage < uMaxPages; uPage++)
    {
        BOOL bAnyAllocsInThisPage = FALSE;
        UINT uIndex     = uPage * g_uPageSize / dwStructSize;
        UINT uIndexLast = uIndex + g_uPageSize / dwStructSize;

        for(; uIndex < uIndexLast; uIndex++)
        {
            MEMORY_BASIC_INFORMATION mbi;
            VirtualQuery(&g_pSomeData[uIndex], &mbi, sizeof(mbi));
            bAnyAllocsInThisPage = ((mbi.State == MEM_COMMIT) &&
                * (PBOOL) ((PBYTE) pvBase + dwStructSize * uIndex));

            // 본 페이지는 디커밋 할 수 없으므로 페이지 검사를 중단한다.
            if(bAnyAllocsInThisPage) break;
        }

        // 본 페이지 내에 할당된 구조체가 없으므로 디커밋을 수행한다. 
        if(!bAnyAllocsInThisPage)
        {
            VirtualFree(&g_pSomeData[uIndexLast - 1], dwStructSize, MEM_COMMIT);
        }
    }
}
```

## **2) Dlg_OnCommand**

```c++
void Dlg_OnCommand(HWND hWnd, int id, HWND hWndCtl, UINT codeNotify)
{
    UINT uIndex = 0;

    switch (id) 
    {
    case IDCANCEL:

        EndDialog(hWnd, id);
        break;

    case IDC_RESERVE:

        // 구조체의 배열을 포함할 수 있을 만큼 충분한 주소 공간을 예약한다.
        g_pSomeData = (PSOMEDATA) VirtualAlloc (NULL,
            MAX_SOMEDATA * sizeof(SOMEDATA), MEM_RESERVE, PAGE_READWRITE);

        // Reserve 버튼을 비활성화 하고 그 외 다른 컨트롤을 활성화 한다. 
        EnableWindow(GetDlgItm(hWnd, IDC_RESERVE), FALSE);

        //EnableWindow ...
        //SetFocus ...
        //InvalidateRect ...
        break;
    
    case IDC_INDEX:

        if(codeNotify != EN_CHANGE)
            break;

        uIndex = GetDlgItemInt (hWnd, id, NULL, FALSE);
        if((g_pSomeData != NULL) && chINRANGE(0, uIndex, MAX_SOMEDATA - 1))
        {
            MEMORY_BASIC_INFORMATION mbi;
            VirtualQuery(&g_pSomeData[uIndex], &mbi, sizeof(mbi));
            BOOL bOk = (mbi.State == MEM_COMMIT);
            if(bOk)
                bOk = g_pSomeData[uIndex].bInUse;

            // EnableWindow ...
        } 
        else
        {
            // EnableWindow ...
        }
        break;

    case IDC_USE:

        uIndex = GetDlgItemInt(hWnd, IDC_INDEX, NULL, FALSE);
        VirtualAlloc(&g_pSomeData[uIndex], sizeof(SOMEDATA),
            MEM_COMMIT, PAGE_READWRITE);

        g_pSomeData[uIndex].bInUse = TRUE;

        // EnableWindow ...
        //SetFocus ...
        //InvalidateRect ...
        break;

    case IDC_CLEAR:
        uIndex = GetDlgItemInt(hWnd, IDC_INDEX, NULL, FALSE);
        g_pSomeData[uIndex].bInUse = FALSE;

        // EnableWindow ...
        //SetFocus ...
        //InvalidateRect ...
        break;

    case IDC_GARBAGECOLLECT:
        
        GarbageCollect(g_pSomeData, MAX_SOMEDATA, sizeof(SOMEDATA));
        //InvalidateRect ...
        break;
    }
}
```

<br/>

# **3. 그 외**

## **1) 보호 특성 변경하기**

실제로는 거의 쓰이지 않는 기능이지만 물리적 저장소가 커밋된 페이지의 보호 특성을 변경할 수 있다. (단, PAGE(_EXCUTE)_WRITECOPY는 사용할 수 없다) 예를 들어 함수 도입부에서는 PAGE_READWRITE로 설정했다가 함수가 종료되기 찍전에 PAGE_NOACCESS로 돌려놓을 경우 잘못된 접근을 막을 수 있을 것이다. 

```c++
BOOL VirtualProtect (
    PVOID pvAddress,
    SIZE_T dwSize,
    DWORD flNewProtect,
    PDWORD pflOldProtect
);
```
단, 서로 다른 영역에 걸쳐있는 여러개의 페이지들에 대해 한번에 보호 특성을 변경할 수는 없다. 서로 이웃한 여러 영역이 있고 이들에 걸쳐있는 다수 페이지에 대해 변경하고자 한다면 이 함수를 여러번 호출해주어야 한다. 

## **2) 물리적 저장소 리셋**

물리적 저장소의 여러 페이지에 걸쳐 내용을 수정하는 경우 시스템은 가능한 변경 사항을 램에 유지하고자 노력한다. 하지만 애플리케이션이 수행 중인 경우 exe, dll, 혹은 페이징 파일로부터 필요한 내용을 수시로 램으로 읽어와야 하며, 이미 페이징 파일의 내용이 로드되어 있는 램의 페이지를 사용할 수도 있다. 이 경우 시스템은 충분한 크기의 페이지를 램으로부터 검색한 후 페이지의 내용이 수정되었을 때 페이징 파일로 그 내용을 저장해야 할 것이다. 

이때 윈도우는 성능 향상을 위해 물리적 저장소 리셋 기능을 제공하는데 이는 저장소에서 페이지 내용이 수정되지 않았다고 알려주는 것을 의미한다. 아주 짧은 시간 동안 저장소를 집중적으로 사용했으나 이후에는 그 내용을 쓸 일이 없다면 성능을 위해 페이징 파일에 내용을 저장하지 않는 게 나을 것이다. 이를 위해 수정되지 않았다고 알려주는 것이다. VirtualAlloc의 세번째 매개변수로 MEM_RESET을 전달할 경우 이를 설정할 수 있다. 

VirtualAlloc을 호출하면 원래 시작 주소는 페이지 경계로 내림이 수행되고, 할당 크기는 페이지 수를 고려하여 올림이 수행된다. 따라서 시작 주소와 할당 크기를 조정하는 것 같은 매커니즘을 저장소 리셋에 사용하면 심각한 문제가 생길 수 있다. 따라서 MEM_RESET을 이용하면 시작 주소는 올림을, 할당 크기는 내림을 수행하여 할당이 된다. 또, 이 플래그는 단독으로만 사용될 수 있으며, 유효한 페이지 보호 특성값을 전달해야 하지만 그 특성이 적용되지는 않는다.


## **3) 주소 윈도우 확장**

윈도우는 AWE, Address Windowing Extension, 윈도우 기법을 이용한 주소 확장 기능을 제공한다. 이를 이용하면 디스크로 스왑되지 않는 램을 할당할 수 있으며, 프로세스 주소 공간보다 더 큰 램에 접근할 수 있게 된다. 애플리케이션이 주어진 메모리보다 더 많은 메모리를 요구하는 경우 이러한 기능을 활용할 수 있다. 

기본적으로 AWE는 애플리케이션이 램에 있는 블록을 할당할 수 있는 방법을 제공한다. 원래 램에 있는 블록을 할당해도 블록은 프로세스 주소 공간에 보이지 않으며, 할당된 블록에 접근하려면 VirtualAlloc으로 영역을 확보하는데 이 공간을 주소 윈도우라고 한다. 이후 특정 함수 호출에 앞서 할당된 램의 블록을 주소 윈도우에 할당하게 된다.

특정 시점의 단일 램 블록은 하나의 주소 윈도우를 통해서만 접근 가능하며, 서로 다른 램 블록에 접근을 시도하기 위해서는 주소 윈도우에 새로운 램 블록을 할당하는 함수가 명시적으로 호출되어야 한다. AWE의 사용 예제는 아래와 같다. 

```c++
// 주소 윈도우로 사용할 1MB를 예약한다.
ULONG_PTR ulRAMBytes = 1024 * 1024;
PVOID pvWindow = VirtualAlloc(NULL, ulRAMBytes,
    MEM_RESERVE | MEM_PHYSICAL, PAGE_READWRITE);

// CPU에서 사용하는 페이지 크기를 구한다. 
SYSTEM_INFO sinf;
GetSystemInfo(&sinf);

// 요청된 메모리를 위해 필요한 램 페이지를 계산한다.
ULONG_PTR ulRAMPages = (ulRAMBytes + sinf.dwPageSize -1 ) /
    sinf.dwPageSize;

// 램 페이지의 페이지 프레임 번호를 저장하기 위해 배열을 할당한다. 
ULONG_PTR* aRAMPages = (ULONG_PTR*) new ULONG_PTR[ulRAMPages];

// 램 페이지를 할당한다. 
// 이때 메모리 페이지 잠금 권한이 필요하다.
AllocateUserPhysicalPages(
    GetCurrentProcess(),    // 어떤 프로세스를 위해 저장소를 할당하는가
    &ulRAMPages,            // 입력: 램 페이지 수 - 출력: 할당된 페이지 수   
    aRAMPages               // 출력: 할당된 램 페이지를 가리키는 배열
);

// 램 페이지를 주소 윈도우에 할당한다.
MapUserPhysicalPages(
    pvWindow,       // 주소 윈도우의 주소
    ulRAMPages,     // 배열 항목 개수
    aRAMPages       // 램 페이지 배열
);

// pvWindow 가상 주소로 램 페이지에 접근한다
FreeUserPhysicalPages(
    GetCurrentProcess(),    // 어떤 프로세스가 사용하던 램을 해제하는가
    &ulRAMPages,            // 입력: 램 페이지 수 - 출력: 삭제된 페이지 수   
    aRAMPages               // 출력: 삭제할 램 페이지를 가리키는 배열
)

// 주소 윈도우 삭제
VirtualFree(pvWindow, 0, MEM_RELEASE);
delete[] aRAMPages;
```

VirtualAlloc에서 MEM_PHYSICAL 플래그를 지정함으로서 램에 할당된 블록을 가리키게 될 것임을 전달한다. 이떄 주소 윈도우를 통해 매핑되는 모든 저장소는 반드시 읽기 쓰기가 가능해야 하기 때문에 PAGE_READWRITE 를 함께 전달해야 한다. 이 경우 VirtualProtect로 보호 특성을 변경할 수 없다. 

AllocateUserPhysicalPages로 물리적 램을 할당한다. aRAMPages을 통해 램 페이지의 페이지 프레임 번호들을 확인할 수 있으나 유용한 값은 아니다. 주소 윈도우로 램 블록에 접근하면 연속된 블록처럼 사용할 수 있어서 램을 더 쉽게 사용하고 삭제할 수 있다. 이 함수가 반환되면 pulRAMPages는 성공적으로 할당된 램 페이지의 개수를 갖는데 이는 대체로 호출 시 전달한 값과 같지만 더 작을 수도 있다.

MapUserPhysicalPages를 호출하면 주소 윈도우에 램 블록을 매핑할 수 있는데 램 블록 인자를 NULL로 지정할 수도 있다. 이 작업은 매우 빨리 이루어지며 이 이후부터 주소 윈도우의 가상 주소를 이용해 간편하게 램 블록에 접근할 수 있다. 더 이상 램 블록이 필요하지 않다면 FreeUserPhysicalPages로 램 블록을 해제하고 VirtualFree로 주소 윈도우도 해제해주어야 한다.

램 페이지를 소유한 프로세스만이 해당 램 페이지를 사용할 수 있으며, 다른 프로세스 주소공간에 할당받은 램 페이지를 매핑하는 것은 불가능하다. AWE를 과도하게 사용하면 사용 가능한 램의 크기가 작아지고, 과도한 디스크 페이징이 생기며, 이는 성능 저하로 이어진다. 따라서 꼭 필요한 경우에 적절히 사용해야 한다. 
