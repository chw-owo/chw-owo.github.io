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
이 함수도 WinMain 안에 하나로 합쳐준다. CreateWindow는 이름 그대로 윈도우를 생성하는 함수로, 세번째 인자를 통해 윈도우 창의 외형을 결정한다. WS_OVERLAPPEDWINDOW는 겹쳐질 수 있는 일반 윈도우 창을 말한다. 창 혹은 프레임을 생성하지 않을 수도 있는데 이 옵션을 OR로 설정할 경우 윈도우 창이 보이지 않는 프로그램이 된다. 

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

이 과정을 마치면 아래쪽은 이렇게 남는데, 이 영역이 Waiting 및 메시지 수신을 처리한다. GetMessage는 호출 스레드 메시지 큐에서 메시지를 검색하며, WM_QUIT (=0) 메시지를 받으면 while을 빠져나와서 함수를 종료시킨다. 그 외의 메시지는 DispatchMessage(&msg)에서 윈도우 프로시저를 호출하여 각 번호에 맞는 기능을 수행한다. 오류가 있는 경우 1을 반환한다. 

이때 윈도우 프로시저는 WinMain에서 호출된다. Overlap IO, 콜백 등과 마찬가지로 요청한 것을 실제로 수행하는 주체는 "요청한" 스레드이다. 콜백은 "작업이 끝나면 알려주세요" 라는 의미지 "이걸 다른 스레드에서 비동기적으로 처리해주세요"가 아니다. 요청 받은 스레드는 작업이 끝났을 때 그걸 호출하는 대신 작업 끝났으니 네가 호출하라고 메시지만 준다. 윈도우는 이 원칙을 따르며, 앞으로 나오는 모든 콜백 개념도 이를 따를 것이다. 

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

이 과정을 마치고 나면 아래와 같은 형태가 된다. 물론 윈도우에서 기본으로 제공하는 템플릿을 그대로 사용하여도 무관하다. 

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
    HWND hWnd;
    MSG msg;
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
    RegisterClassExW(&wcex);
    hWnd = CreateWindowW(szWindowClass, szTitle, WS_OVERLAPPEDWINDOW,
      CW_USEDEFAULT, 0, CW_USEDEFAULT, 0, nullptr, nullptr, hInstance, nullptr);

    if (!hWnd)
        return FALSE;
    ShowWindow(hWnd, nCmdShow);
    UpdateWindow(hWnd);
   
    while (GetMessage(&msg, nullptr, 0, 0))
    {
        
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    return (int) msg.wParam;
}
```

# **3. message**

## **1) 기본 구조**

```c++
typedef struct tagMSG 
{
	HWND hwnd;
    UINT message;
    WPARAM wParam;
    LPARAM lParam;
    DWORD time;
    POINT pt;
    DWORD lPrivate;
} MSG;
```
메시지 구조체는 이와 같이 생겼다. wParam, lParam은 은 상황에 맞게 선택할 수 있게끔 제공되는 것으로, 64bit에서는 64bit(8byte), 32bit에서는 32bit(4byte)의 크기를 갖는다. W/L은 별 의미 없으며 그때그때 필요한 걸 문서에서 찾아 사용하면 된다. 보통 W에는 Key 값이, L에는 플래그 값이나 좌표값이 들어간다.

```c++
switch(message)
{
case WM_DESTROY:
    // …
    break;

default:
    return DefWindowProc(...);
}
```
WndProc에 여러 case가 있는데 위 두개를 제외하고는 지워도 된다. 중요한 시스템 메시지에 대한 대비를 우리가 모두 마련할 수는 없기 때문에 윈도우에서 이를 처리하기 위해 DefWindowProc를 제공한다. 프로시저가 호출이 됐을 때 우리가 나눈 분기로 간다면 우리가 만든 코드가 실행되고 아닌 경우엔 DefWindowProc가 실행이 된다. default: DefWindowProc이 없으면 창 생성도 되지 않으므로 반드시 필요하다. 

또 WM_DESTROY는 프로세스 종료시 필요한 분기이다. 창을 닫는 것은 WM_SYSCOMMAND에서 처리하므로 이게 없어도 창을 닫을 수는 있으나 프로세스가 종료되지 않는다. 창이 없어서 작업 관리자 - 앱에도 뜨지 않고 프로세스 항목에 들어가서 강제 종료 해야한다. WM_DESTROY는 어플리케이션 파괴가 아닌 윈도우 파괴 시에 실행되며, PostQuitMessage가 WM_QUIT를 발생시켜서 GetMessage가 0을 반환하고 while 문을 나오도록 하는 역할을 한다. 윈도우 종료와 프로세스의 종료가 다른 것임을 유의해야 한다. 

## **2) 메시지 종류**

```c++
switch(message)
{
case WM_SYSCOMMAND:
    break;
case WM_DESTROY:
    // …
    break;
default:
    return DefWindowProc(...);
}
```

WM_SYSCOMMAND는 커맨드 입력과 관련된 메시지를 받는다. 따라서 위 예시처럼 만들면 커맨드 입력을 할 수 없으며, 창을 닫을 수도 없다. DefWindowProc();에서 WM_SYSCOMMAND를 받아 커맨드 입력을 처리하는데 그것을 가로채고 아무 동작도 하지 않기 때문이다. 이 경우 창을 끄려면 작업 관리자에서 프로세스를 강제 종료해야 한다. 

```c++
switch(message)
{
case WM_SYSCOMMAND:
	if(wParam == SC_CLOSE)
		if( IDNO == MessageBox(hWnd, L”종료하시겠습니까?”,  L”종료”, MB_YESNO);)
            break;
    DefWindowProc(...);

break;
    case WM_DESTROY:
    // …
    break;
default:
    return DefWindowProc(...);
}
```

해당 메시지를 위처럼 수정하면 메시지 박스를 띄운 뒤, YES 고른 경우에 한하여  DefWindowProc();를 호출한다. 이 처리를 while(GetMessage)안에서 한 뒤 continue로 이어서 실행하도록 만들어도 동일하게 동작하긴 하지만, 위 코드처럼 switch(message) 안에서 처리하는 것이 보다 정석적인 방법이다. 메시지 박스는 블락 함수이므로 서버 자체에서 쓸 일은 없지만, 애플리케이션 제작에서는 유용하게 사용된다. 

```c++
wsprintf(szMessage, L”msg: %d\n”, msg.message); 
OutputDebugString(szMessage);
```

while(GetMessage) 안에서 msg.message를 통해받은 메시지의 번호를 알 수 있는데 이걸 이용해 애플리케이션의 모든 메시지를 출력할 수 있다. 그러나 이 경우 이름 볼 수 없기 때문에 대부분 그냥 Spy++를 쓴다. Visual Studio -  도구 - Spy++ - 메시지 옵션에서 과녁을 어플리케이션 위로 옮겨보면 해당 위치에 발생하는 메시지를 모두 로그로 남겨준다. 게임 제작할 땐 쓸 일이 없지만 어플리케이션 제작시에는 유용하게 쓸 수 있다. 

그 외에, WM_CREATE의 경우 생성될 때 호출되며 생성자와 같은 용도로 사용할 수 있다. WM_KEYDOWN도 있는데 잘 사용되지 않는다. 한번 눌렀을 때 한번 전송되는 게 아니라 눌린 시간만큼 연속적으로 전송되기 때문이다. 게임을 만들 때는 GetAsyncKeyState 나 Direct의 Input을 사용한다. Unity는 키보드 입력을 영어로만 받아서 한글 누른 상태에서는 Key가 안먹는데 이것 때문에 Unity에서도 GetAsyncKeyState을 사용한다고 한다. 

## **3) 게임에서의 메시지 처리**

MS에서는 게임을 위한 WinMain 예제를 제공하는데 여기서는 GetMessage 대신 PeekMessage를 사용한다. Peek은 데이터를 pop 하는 대신 Queue의 데이터를 유지한 채로 복사를 통해 데이터의 값만 얻어낸다는 점에서 Deque와 다르다. 이때 Deque는 메시지가 없을 경우 Block을 걸지만 Peek은 false를 반환할 뿐 Block이 걸리지는 않는다. 이를 통해 메시지가 있을 경우 메시지를 처리하고 없을 경우 게임 처리를 하는 로직을 만들 수 있다. 

이때 if에서는 윈도우 메시지만, else에서는 게임 로직만 처리하도록 분리하는 게 좋다. 그 이유로는 첫째로, 윈도우 메시지 처리가 우선순위에 있기 때문이다. 둘째로, 게임 로직은 윈도우 메시지에 비해 처리가 오래 걸리기 때문이다. 이런 이유로 만약 if 안에 윈도우 메시지와 게임 로직을 함께 처리한다면 창 드래그가 버벅이는 등 윈도우 메시지가 밀릴 수 있다. 실제로 유니티, 언리얼 엔진의 경우 내부적으로 이런 방법을 통해 구현했다고 한다. 
