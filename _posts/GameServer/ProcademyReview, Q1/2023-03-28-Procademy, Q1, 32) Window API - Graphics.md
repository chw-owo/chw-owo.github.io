---
title: Procademy, Q1, 32) Window API - Graphics
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 윈도우에서의 그래픽 출력**

## **1) WM_PAINT**

```c++
case WM_PAINT:
	// 렌더링
	break;

case WM_MOUSEMOVE:
	// 데이터 변화
	// WM_PAINT 메시지 발생
	break;
```

일반적으로 렌더 관련 코드는 모두 WM_PAINT에서 처리한다. 마우스가 움직이는 대로 그림이 그려지도록 하고 싶다면, WM_PAINT에서 렌더링 코드를 작성한 뒤, WM_MOUSEMOVE에서 WM_PAINT 메시지를 발생시키면 된다. WM_MOUSEMOVE에서 렌더링 코드를 작성해도 비슷하게 동작하지만 이런 방법을 써야 윈도우가 다른 윈도우에 의해 가려졌다가 돌아왔을 때, 자동으로 화면 갱신을 유도할 수 있다. 


```c++
BOOL InvalidateRect(
	//…
)
```

이 함수로 WM_PAINT를 의도적으로 발생시켜 어디서든 갱신을 유도할 수 있다. 이때 인자로 일부 영역만 갱신되도록 설정할 수도 있으나 이는 잘 사용되지는다. Window 10에서는 반투명 기능이 나오며 자체적인 복원 기능이 생겨서 외부적 요인으로 인한 WM_PAINT의 호출 자체가 사라졌다. 따라서 윈도우가 가려졌다가 돌아왔을 때 WM_PAINT가 자동으로 호출되지 않는다. 그러나 최소화 후 다시 켤 경우에는 호출이 된다. 

## **2) DC**

윈도우에서 그래픽을 출력할 경우 보통 멀티미디어 어플엔 DirectX를, 그 외엔 GDI를 쓴다. GDI를 쓸 경우 GDI 출력을 위한 Device Context, DC 핸들이 있어야 한다. 이때 모니터 외에 메모리, 프린터 등도 그래픽 출력의 대상이 될 수 있다. WM_PAINT 안에는 beginPaint, endPaint가 반드시 포함되어야 하는데, 이것이 렌더링 작업 수행 여부를 전달하여 DC를 얻는 역할을 한다. 이게 없을 경우 작업이 되지 않았다고 판단하여 WM_PAINT 메시지를 계속 보낸다. MS는 이것을 WM_PAINT 외부에서 쓸 경우 예상치 못한 예외가 생길 수 있다고 경고한다. 

윈도우 핸들만 있다면 어디서든 GetDC로 DC를 얻을 수 있다. 이때 ReleaseDC를 안하고 GetDC만 1만번 하면 이후 GetDC에서 INVALID_HANDLE이 반환된다. 과거에는 GDI 개체가 누수나면 윈도우 전체가 이상해졌는데 지금은 안전 장치를 넣어준 셈이다. GDI 개체를 쓰는 함수는 모두 커널 모드에서 동작한다. 과거에는 프로세스간 공유도 안되고 보안 인자도 필요 없다보니 GDI 개체를 유저 오브젝트로 두었으나, 이걸 사용하는 과정에서 반드시 커널 모드 전환이 필요하다보니 최근에는 MS에서 커널이 관리하도록 바꾸고 커널 오브젝트로 새로 분류하였다. 

<br/>

# **2. 그래픽 출력 예제**

## **1) WM_MOUSEMOVE, WM_PAINT 기본 예제**

```c++
case WM_MOUSEMOVE:

    int xPos = GET_X_LPARAM(lParam);
    int yPos = GET_Y_LPARAM(lParam);

    HPEN hPen = CreatePen(PS_SOLID, 5, RGB(255,0,0));
    HDC hdc = GetDC(hWnd);

    HPEN hOldPen = SelectObject(hdc, hPen);
    MoveToEx(hdc, 0, 0, nullptr);
    LineTo(hdc, xPos, yPos);
    SelectObject(hdc, hOldPen);
    DeleteObject(hPen);
    ReleaseDC(hWnd, hdc);
    break;

```

DC는 기본 값으로 윈도우의 하얀 창을 주는데, 버튼이나 상단 창을 전달하여 다른 이미지로 덮어쓰기 할 수도 있다. 이 경우 선 하나 그을 때마다 GDI 개체가 생성과 삭제를 반복한다. 작업 관리자에서 GDI 개체 개수도 확인할 수 있는데, 일정한 크기로 유지되는 것을 볼 수 있다. 반면 ReleaseDC를 지우고 실행한다면 마우스가 움직일 때마다 이 개체가 늘어나며, 이게 1만개를 넘기면 더 이상 선이 그어지지 않는다. 

pen과 같은 개체는 사용 후 Delete 해야 하는데 이때 DC에 연결되어 있으면 지울 수 없다. 따라서 다른 pen에 대한 값을 갖고 있다가 Select를 통해 이전 pen과 연결하고 삭제해야 한다. 최근에는 Select를 생략해도 검정색 default pen을 자동으로 연결 후 해제하기 때문에 에러가 나지는 않지만, 원칙적으로는 해주는 게 맞다. 

이처럼 매번 생성-삭제를 하는 것이 일반적이지만, 전역 객체를 이용하여 더 효율적으로 동작하도록 수정할 수도 있다. 

```c++
int g_OldX;
int g_OldY;
bool g_Click = false;

case WM_LBUTTONDOWN:
	g_Click = true;
    g_OldX = GET_X_LPARAM(lParam);
    g_OldY = GET_Y_LPARAM(lParam);
	break;

case WM_LBUTTONUP:
	g_Click = false;
	break;

case WM_MOUSEMOVE:
	if(g_Click)
    {
        HDC hdc = GetDC(hWnd);
        MoveToEx(hdc, g_OldX, g_OldY, , nullptr);
        LineTo(hdc,  GET_X_LPARAM(lParam),  GET_Y_LPARAM(lParam));
        ReleaseDC(hWnd, hdc);

        g_OldX = GET_X_LPARAM(lParam);
        g_OldY = GET_Y_LPARAM(lParam);
    }
    break;

```

위 예시는 마우스 움직임대로 선이 생기도록 하는 코드이다. 선처럼 보이려면 연속된 선들을 출력해야 하므로 과거의 좌표를 저장해두어야 한다. MoveToEx에 과거의 마지막 좌표를 ref를 통해 돌려주는 기능이 있지만, 위 예제에서는 그걸 사용하는 대신 과거 좌표를 전역으로 관리하고 있다.  

```c++
int g_OldX;
int g_OldY;
HPEN g_hPen;

case WM_CREATE:
	g_hPen = CreatePen(PS_SOLID, 10, RGB(rand()% 256…));
	break;

case WM_RBUTTONDOWN:
	DeleteObject(g_hPen);
	g_hPen = CreatePen(PS_SOLID, 10, RGB(rand()% 256…));
	break;

case WM_MOUSEMOVE:
	if(wParam & MK_LBOTTUN)
    {
        HDC hdc = GetDC(hWnd);
        MoveToEx(hdc, g_OldX, g_OldY, , nullptr);
        LineTo(hdc,  GET_X_LPARAM(lParam),  GET_Y_LPARAM(lParam));
        ReleaseDC(hWnd, hdc);

        g_OldX = GET_X_LPARAM(lParam);
        g_OldY = GET_Y_LPARAM(lParam);
    }
    break;

case WM_DESTROY:
	DeleteObject(g_hPen);
	break;
```

g_Click를 직접 만드는 대신 if(wParam & MK_LBOTTUN)처럼 MS가 제공하는 플래그를 사용해도 된다. 대신 처음 눌리는 시점을 특정하기 어렵다보니 다른 장치가 더 필요한데, 그럼에도 일반적으로 MS에서 제공하는 플래그를 더 많이 사용한다. 

## **2) 타이머를 활용한 예제**

윈도우에서는 기본 타이머 기능을 제공하는데, 타이머 ID에 임의 번호를 지정하고 필요할 때 번호를 바탕으로 호출하면 된다. TIMERPROC에 함수 포인터를 넣어서 콜백처럼 사용할 수 있으며, 지정하지 않을 시 WM_TIMER 메시지가 발생한다.

```c++
int g_OldX;
int g_OldY;
HPEN g_hPen;

case WM_CREATE:
	SetTimer(hWnd, 1, 100, nullptr);
	g_hPen = CreatePen(PS_SOLID, 10, RGB(rand()% 256…));
	break;

case WM_TIMER:
	DeleteObject(g_hPen);
	g_hPen = CreatePen(PS_SOLID, 10, RGB(rand()% 256…));
	break;

case WM_MOUSEMOVE:
	if(wParam & MK_LBOTTUN)
    {
        HDC hdc = GetDC(hWnd);
        MoveToEx(hdc, g_OldX, g_OldY, , nullptr);
        LineTo(hdc,  GET_X_LPARAM(lParam),  GET_Y_LPARAM(lParam));
        ReleaseDC(hWnd, hdc);

        g_OldX = GET_X_LPARAM(lParam);
        g_OldY = GET_Y_LPARAM(lParam);
    }
    break;

case WM_DESTROY:
	KillTimer(~~);
	DeleteObject(g_hPen);
	break;
```
이는 100ms 마다 펜의 색이 바뀌는 예제이다. 이때 타이머 메시지는 우선순위가 낮은 메시지이므로 타이머를 바탕으로 중요한 메시지를 발생시킬 경우 메시지가 밀리거나 느려질 수 있다. 따라서 중요한 작업 혹은 정교한 작업에서는 사용하면 안된다. 


## **3) 맵 그리기 툴 제작 가이드**

길은 직접 제작할 수도 있고, 랜덤으로도 만들 수 있게끔 하는 게 좋다. 그리드 펜을 전역에 두고, DC를 인자로 받아서 연결시킨다. for문 안에 LineTo.. 가로 세로 격자를 치고 있는 것. 그 후 장애물 펜을 전역에 두고 RenderObstacle에서 DC와 연결시킨다. 이때 NULL_PEN을 지정할 수 있는데, 이는 투명한 펜으로 시스템이 제공하는 오브젝트이므로 지우지 않아도 된다. 이게 연결된 DC로 Rectangle을 호출하면 투명한 네모를 그려준다. 속성을 바꿀 때마다 InvalidateRect 해야 반영이 되는데, 이때 false를 하면 잔상이 남고 true를 하면 깜빡깜빡 거리기 때문에 더블 버퍼링을 사용해야 한다. 

더블 버퍼링을 할 때는 윈도우창과 동일한 메모리 DC를 버퍼로 쓴다. 프레임, 메뉴, 캡션바를 제거한 출력 영역을 Client 영역이라 부르는데, GetClientRect를 통해 현재 윈도우의 Client 영역 크기를 구할 수 있다. 그 후 CreateCompatibleBitmap에 베끼고 싶은 DC를 전달하면 동일하게 생긴 bitmap을 만들어준다. DC를 잘못 넣으면 색을 표현할 수 없으므로 검정 화면이 나온다. 이 후 CreateCompatibleDC를 통해 현 윈도우 DC와 호환되는 DC를 만든 뒤 연결시키면 메모리 DC 생성은 끝난다. 

새 렌더링을 위해서는 메모리 DC를 매번 초기화해주어야 한다. 특정한 패턴을 원한다면 FillRect를, 단색으로 밀고 싶다면 PatBlt을 권장한다. 메모리 DC Clear - Render - Flip (BitBlt으로 메모리 DC를 화면에 주사) 과정을 WM_PAINT에서 실행하면 깜빡임 없이 렌더링을 반영할 수 있다. InvalidateRect에서는 화면을 더 이상 지울 필요 없기 때문에 false를 지정해야 한다. 마지막으로 WM_SIZE에서 삭제 생성 코드를 추가해주어야 윈도우 크기가 변경될 때마다 DC를 새로 만들어주게 된다. 이 작업을 하지 않으면 윈도우 크기가 변경되었을 때 변경된 영역에서는 렌더 작업이 안 돌아간다.

