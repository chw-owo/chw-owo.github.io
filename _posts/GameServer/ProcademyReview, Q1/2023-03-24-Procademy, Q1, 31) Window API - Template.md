---
title: Procademy, Q1, 31) Window API - Template
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 윈도우 프로그래밍의 구조**

윈도우 애플리케이션은 WinMain에서 시작하여 RegisterClass, CreateWindow, ShowWindow, MessageLoop 순으로 실행된다. Visual Studio에서 Windows 데스크톱 애플리케이션을 선택하면 이 순서로 동작하는 템플릿이 제공된다. WinMain 없이 콘솔앱으로 만들고 CreateWindow를 해도 콘솔을 통해 윈도우가 뜨기는 한다.

윈도우는 메시지 드리븐 방식을 사용하는데, 모든 내/외부의 통신과 처리를 메시지를 통해 진행한다. 응답 없음은 메시지를 제때 처리하지 않았을 때 뜨는데, 이건 윈도우 애플리케이션 한정이고 콘솔 애플리케이션에서는 이런 일이 일어나지 않는다. 메시지 큐에 이벤트가 전송되면 그걸 디큐잉 해서 사용하며, 이것을 실질적으로 처리하는 영역을 WinProc, 윈도우 프로시져라고 부른다. 

할 일이 없을 때의 윈도우 애플리케이션은 늘 Waiting 상태이다. 절차지향 방식의 애플리케이션은 늘 loop를 도는 반면 메시지 방식의 애플리케이션은 메시지가 없으면 Waiting으로 대기하게 된다. 여기서 쉰다는 건 아예 block 시켜서 CPU를 잡아먹지 않는 상태로 둔다는 것을 의미한다. 이런 구조이므로 로직의 전체적인 순서도를 파악하기는 어렵지만, 효율적이고 간결한 구조로 만들 수 있다. 

기본적으로 게임, mp3 같은 프로그램은 메시지가 없어도 멈추지 않고 동작해야 한다. 따라서 윈도우 애플리케이션 기반으로 게임을 만들 땐 일반적으로 블락 함수를 다른 걸로 대체한다. 아주 초창기에는 메시지를 무시하는 방식을 사용했기에 게임을 켜면 게임 외의 모든 동작이 안먹혔다. 당연히 요즘은 이러면 안되고 메시지를 받으면서 block을 막아야 한다.

<br/>

# **2. template의 구조**

## **0) template 전체**

```c++
#include "framework.h"
#include "WindowsProject1.h"

#define MAX_LOADSTRING 100

// 전역 변수:
HINSTANCE hInst;                                // 현재 인스턴스입니다.
WCHAR szTitle[MAX_LOADSTRING];                  // 제목 표시줄 텍스트입니다.
WCHAR szWindowClass[MAX_LOADSTRING];            // 기본 창 클래스 이름입니다.

// 이 코드 모듈에 포함된 함수의 선언을 전달합니다:
ATOM                MyRegisterClass(HINSTANCE hInstance);
BOOL                InitInstance(HINSTANCE, int);
LRESULT CALLBACK    WndProc(HWND, UINT, WPARAM, LPARAM);
INT_PTR CALLBACK    About(HWND, UINT, WPARAM, LPARAM);

int APIENTRY wWinMain(_In_ HINSTANCE hInstance,
                     _In_opt_ HINSTANCE hPrevInstance,
                     _In_ LPWSTR    lpCmdLine,
                     _In_ int       nCmdShow)
{
    UNREFERENCED_PARAMETER(hPrevInstance);
    UNREFERENCED_PARAMETER(lpCmdLine);

    // TODO: 여기에 코드를 입력합니다.

    // 전역 문자열을 초기화합니다.
    LoadStringW(hInstance, IDS_APP_TITLE, szTitle, MAX_LOADSTRING);
    LoadStringW(hInstance, IDC_WINDOWSPROJECT1, szWindowClass, MAX_LOADSTRING);
    MyRegisterClass(hInstance);

    // 애플리케이션 초기화를 수행합니다:
    if (!InitInstance (hInstance, nCmdShow))
    {
        return FALSE;
    }

    HACCEL hAccelTable = LoadAccelerators(hInstance, MAKEINTRESOURCE(IDC_WINDOWSPROJECT1));

    MSG msg;

    // 기본 메시지 루프입니다:
    while (GetMessage(&msg, nullptr, 0, 0))
    {
        if (!TranslateAccelerator(msg.hwnd, hAccelTable, &msg))
        {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        }
    }

    return (int) msg.wParam;
}
```
이 중 필요없는 것은 지운 뒤에 작업을 할 것이다. 

## **1) WinMain**

```c++
// 전역 변수:
HINSTANCE hInst;                                // 현재 인스턴스입니다.
WCHAR szTitle[MAX_LOADSTRING];                  // 제목 표시줄 텍스트입니다.
WCHAR szWindowClass[MAX_LOADSTRING];            // 기본 창 클래스 이름입니다.
```
이 변수들은 전역으로 관리할 이유가 없으니 지운다.

```c++
int APIENTRY wWinMain(_In_ HINSTANCE hInstance,
                     _In_opt_ HINSTANCE hPrevInstance,
                     _In_ LPWSTR    lpCmdLine,
                     _In_ int       nCmdShow)
```

lpCmdLine가 문자열 인자이기 때문에 WinMain도 ANSI 버전과 Wide 버전이 나뉜다. 2013에서는 중립자료형을 권장하던 시기라 _tWinMain이 기본이었으나 현재는 wWinMain이 기본 템플릿으로 제공된다. hInstance는 실행 파일 이미지 메모리의 핸들을 전달하는데, 이를 연동하는 형태의 프로그램에서는 사용하겠지만 지금은 사용할 일 없다. hPrevInstance도 마찬가지로 쓸 일 없다. nCmdShow는 Cmd를 어떻게 보여줄지 옵션을 지정한다. 

```c++
int main(int argc, char** argv)
```
lpCmdLine의 경우 위 함수에서 문자 넣듯이 인자로 값을 넣는다. 당장은 쓸 일이 없다. 

```c++
UNREFERENCED_PARAMETER(hPrevInstance);
UNREFERENCED_PARAMETER(lpCmdLine);
```
매크로를 확인해보면 아무것도 없는 것을 확인할 수 있다. 이것들은 윈도우 애플리케이션에서 전반적으로 잘 쓰이지 않는 변수인데, 안쓰는 변수라고 계속 경고 띄우니까 그것을 없애기 위해 넣은 아무 의미없는 코드이다. 

```c++
// 전역 문자열을 초기화합니다.
LoadStringW(hInstance, IDS_APP_TITLE, szTitle, MAX_LOADSTRING);
LoadStringW(hInstance, IDC_WINDOWSPROJECT1, szWindowClass, MAX_LOADSTRING);
MyRegisterClass(hInstance);
```
Visual Studio - 보기 - 리소스 뷰를 보면 편리한 기능이 많다. ACCELERATOR로 단축키 구현도 할 수 있고, 다이얼로그도 쉽게 구현할 수 있다. 이 중 String Table이라는 게 있는데 이걸 Load 할 때 저 LoadStringW를 사용한다. 참고로 이건 실행파일에 들어가는 값이라 수정하면 빌드를 새로 해야 한다. 사용하지 않을 것이라면 지워도 된다. 

```c++
if (!InitInstance (hInstance, nCmdShow))
    {
        return FALSE;
    }
```
```c++
//   함수: InitInstance(HINSTANCE, int)
//   용도: 인스턴스 핸들을 저장하고 주 창을 만듭니다.
//   주석: 이 함수를 통해 인스턴스 핸들을 전역 변수에 저장하고
//        주 프로그램 창을 만든 다음 표시합니다.
BOOL InitInstance(HINSTANCE hInstance, int nCmdShow)
{
   hInst = hInstance; // 인스턴스 핸들을 전역 변수에 저장합니다.

   HWND hWnd = CreateWindowW(szWindowClass, szTitle, WS_OVERLAPPEDWINDOW,
      CW_USEDEFAULT, 0, CW_USEDEFAULT, 0, nullptr, nullptr, hInstance, nullptr);

   if (!hWnd)
   {
      return FALSE;
   }
   ShowWindow(hWnd, nCmdShow);
   UpdateWindow(hWnd);

   return TRUE;
}
```
이 함수도 WinMain 안에 하나로 합쳐준다.

```c++
HWND hWnd = CreateWindowW(szWindowClass, szTitle, WS_OVERLAPPEDWINDOW,
    CW_USEDEFAULT, 0, CW_USEDEFAULT, 0, nullptr, nullptr, hInstance, nullptr);
```
CreateWindow의 경우 과거에는 함수였지만 지금은 매크로로 매핑되어서 문자열 별로 다른 버전을 갖는다. 따라서 아래와 같은 인자가 들어가야 한다. 

```c++
HWND hWnd = CreateWindowW(L”Class”, L”WindowTitle”, WS_OVERLAPPEDWINDOW,
    CW_USEDEFAULT, 0, CW_USEDEFAULT, 0, nullptr, nullptr, hInstance, nullptr);
```
참고로 윈도우는 창을 껐다 키면 원래 있던 자리에 창이 뜨는데, 이 값은 윈도우 레지스트리에 저장되었다가 불려진다. CW_USEDEFAULT가 이 설정에 쓰이는데 자세한 건 문서를 참조하면 된다.

```c++
HWND hWnd = CreateWindowW(L”Class”, L”WindowTitle”, WS_OVERLAPPEDWINDOW,
    CW_USEDEFAULT, 0, CW_USEDEFAULT, 0, nullptr, nullptr, hInstance, nullptr);

if (!hWnd)
{
    DWORD dwError = GetLastError();
        return FALSE;
}
ShowWindow(hWnd, nCmdShow);
UpdateWindow(hWnd);

return TRUE;
```
에러가 나면 GetLastError()로 무조건 에러를 확인해야한다. 모든 윈도우 API는 에러 코드를 명확하게 주게 되어있다.  

```c++
HACCEL hAccelTable = LoadAccelerators(hInstance, MAKEINTRESOURCE(IDC_WINDOWSPROJECT1));
```
```c++
…
if (!TranslateAccelerator(msg.hwnd, hAccelTable, &msg))
```
액셀레이트 쓰지 않을 것이라면 이것도 지운다

```c++
    MSG msg;
    // 기본 메시지 루프입니다:
    while (GetMessage(&msg, nullptr, 0, 0))
    {
TranslateMessage(&msg);
          DispatchMessage(&msg);
    }
    return (int) msg.wParam;
```
이 과정을 마치면 아래쪽은 이렇게 남는다. 이 영역이 Waiting을 처리해준다.

이때 DispatchMessage(&msg)에서 윈도우 프로시저를 호출하는데, 이 윈도우 프로시저는 WinMain에서 호출된다. 나중에 Overlap IO, 콜백 등을 많이 쓸텐데 그것들도 실제로 호출하는 주체는 요청한 스레드이다. 콜백은 "작업이 끝나면 알려주세요" 라는 뜻이지 "이걸 다른 스레드에서 비동기적으로 처리해주세요"가 아니다. 요청 받은 스레드는 작업이 끝났을 때 직접 그걸 호출하는 대신 작업 끝났으니 너가 호출해라 라고 메시지만 준다. 윈도우는 이 원칙을 따르며, 앞으로 나오는 모든 콜백 개념도 이를 따를 것이다. 

## **2) RegisterClass & CreateWindow**

```c++
//  함수: MyRegisterClass()
//  용도: 창 클래스를 등록합니다.
ATOM MyRegisterClass(HINSTANCE hInstance)
{
    WNDCLASSEXW wcex;

    wcex.cbSize = sizeof(WNDCLASSEX);

    wcex.style          = CS_HREDRAW | CS_VREDRAW;
    wcex.lpfnWndProc    = WndProc;
    wcex.cbClsExtra     = 0;
    wcex.cbWndExtra     = 0;
    wcex.hInstance      = hInstance;
    wcex.hIcon          = LoadIcon(hInstance, MAKEINTRESOURCE(IDI_WINDOWSPROJECT1));
    wcex.hCursor        = LoadCursor(nullptr, IDC_ARROW);
    wcex.hbrBackground  = (HBRUSH)(COLOR_WINDOW+1);
    wcex.lpszMenuName   = MAKEINTRESOURCEW(IDC_WINDOWSPROJECT1);
    wcex.lpszClassName  = szWindowClass;
    wcex.hIconSm        = LoadIcon(wcex.hInstance, MAKEINTRESOURCE(IDI_SMALL));
    return RegisterClassExW(&wcex);
}
```

RegisterClass는 함수 포인터, 윈도우 형태, 색 등을 지정하는 것에 쓰이며, 똑같은 형태의 윈도우를 여러개 만들 경우를 대비해 CreateWindow와 분리하여 지정한다. 제일 중요한 건 wcex.lpfnWndProc = WndProc; 이 부분으로, 이는 윈도우 프로시저를 지정하는 것이다. wcex.lpszClassName = szWindowClass; 이건 아무거나 지정해도 괜찮다. 이 내용도 WinMain 안에 합친다.
