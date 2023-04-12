---
title: Exception 01) Exception
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. SEH, Structed Exception Handling**

SEH를 쓰면 예외 처리를 주 작업에서 분리하여 주 작업에 집중할 수 있다. 비정상적인 동작이 있을 때 시스템이 이를 확인하여 문제를 알려준다. 그러나 컴파일러가 SEH를 위한 코드와 테이블, 콜백 함수, 스택 프레임 등을 구성하기 때문에 성능 저하가 생길 수 있고, 다른 OS에 SEH를 이식할 수 없다는 한계가 있다. SEH의 세부 구현 방식은 컴파일러마다 조금씩 다르며, 이 포스트는 MS Visual C++ 컴파일러 문법을 따른다. 

<br/>

# **2. Termination Handling**

## **1) 종료 처리기**

종료 처리기는 try로 본문을, finally로 종료 코드를 묶어서, 본문 내에서 제어가 어떤 식으로 빠져나오든 종료 처리기 내부 코드 블록이 반드시 수행될 것임을 보장하는 것이다. 이는 return, goto, longjump 등으로 빠져나와도 동일하지만 이런 문장은 성능 상의 이유로 되도록 쓰지 않는 게 좋다. 단, Exit-, Terminate- 함수로 종료되는 경우는 보장하지 않는다. 

## **2) 사용 예제**

```c++
DWORD Funcenstein()
{
    DWORD dwTemp;
    __try 
    {     
        WaitForSingleObject(g_hSem, INFINITE);
        g_dwProtectedData = 5;
        dwTemp = g_dwProtectedData;
        return(dwTemp);
    }
    __finally 
    { 
        ReleaseSemaphore(g_hSem, 1, NULL); 
    }

    dwTemp = 7;
    return(dwTemp);
}
```

이 예제에서 컴파일러는 우선 try 내의 return을 확인한다. 이를 처리하기 위해 반환값(5)를 임시 변수에 저장하고, finally 코드를 return 앞에 생성한 뒤, 임시 변수 값으로 return을 한다. 따라서 함수를 나가기 직전에 세마포어가 해제되며, try의 return 이하 코드는 수행되지 않는다. 

위와 같은 과정을 로컬 언와인드라고 한다. 그러나 이는 추가적인 코드를 생성해야 하므로 성능에 안좋은 영향을 줄 수 있다. 따라서 종료 처리기의 try 블록 내에 함수를 나가는 코드를 쓰는 등 부자연스러운 흐름을 만드는 건 지양하는 게 좋다. 

```c++
DWORD Funcenstein()
{
    DWORD dwTemp;
    __try 
    {     
        WaitForSingleObject(g_hSem, INFINITE);
        g_dwProtectedData = 5;
        dwTemp = g_dwProtectedData;
        return(dwTemp);
    }
    __finally 
    { 
        ReleaseSemaphore(g_hSem, 1, NULL); 
        return(9)
    }

    dwTemp = 7;
    return(dwTemp);
}
```

이 경우 함수는 9를 반환한다. finally의 return이 5가 저장되었던 임시 변수에 9를 덮어쓰기 때문이다. 

```c++
DWORD Funcenstein()
{
    DWORD dwTemp;
    __try 
    {     
        WaitForSingleObject(g_hSem, INFINITE);
        g_dwProtectedData = 5;
        dwTemp = g_dwProtectedData;
        goto ReturnValue;
    }
    __finally 
    { 
        ReleaseSemaphore(g_hSem, 1, NULL); 
    }
    
    dwTemp = 7;
    ReturnValue;
    return(dwTemp);
}
```

이 경우에도 goto에 앞서 finally를 먼저 수행하도록 로컬 언와인드 작업이 일어난다. 꼭 return이 아니더라도 이렇게 try 블록의 자연스러운 제어 흐름을 가로채어 finally 블록을 수행하는 것은 CPU 성능에 불이익을 초래할 수 있다. 

```c++
DWORD Funcfurter()
{
    DWORD dwTemp;
    __try 
    {     
        WaitForSingleObject(g_hSem, INFINITE);
        dwTemp = Funcinator(g_dwProtectedData);

    }
    __finally 
    { 
        ReleaseSemaphore(g_hSem, 1, NULL); 
    }
    
    return(dwTemp);
}
```

유효하지 않은 메모리에 접근하는 Funcinator 함수를 가정해보자. SEH를 쓰지 않으면 다이얼로그 박스가 나타나며, 프로세스가 종료된다. 그러나 세마포어는 여전히 해제되지 않은 채 종료된 프로세스 내 스레드가 소유하게 될 것이며, 다른 스레드는 CPU 시간을 할당받지 못한다. 그러나 위처럼 종료 처리기를 사용하면 메모리 접근 위반 상황에서도 세마포어를 사용할 수 있다.  

그러나 이전의 윈도우는 예외 발생 시 finally의 호출을 항상 보장해주지는 않는다. 예를 들어 윈도우 XP에서 스택 고갈 예외가 나는 경우, WER 코드가 예외 유발 프로세스 내에서 수행되기 때문에 에러 보고를 위한 스택을 확보하지 못해 finally가 수행되지 않고 종료된다. 또, SEH 체인 내 손상으로 예외가 발생하거나 예외 필터에서 또 다른 예외가 생긴 경우에도 수행되지 않는다. 

윈도우 비스타 이후로는 에러 보고 기능을 독립된 프로세스가 수행하도록 변경되었다. 따라서 이전에는 종료 전에 finally가 실행될 경우 특별한 경우를 빼면 글로벌 언와인드가 발생했는데 비스티 이후로는 글로벌 언와인드가 발생하지 않으며 finally도 수행되지 않는다. 따라서 예외 발생 시 finally가 수행되게 하려면 이를 명시해야 한다.

이런 이유로 가능한 catch, finally 블록에서는 최소한의 동작만 수행하는 것이 좋다. 

```c++
DWORD Funcarama()
{
    HANDLE hFile = INVALID_HANDLE_VALUE;
    PVOID pvBuf = NULL;
    BOOL bFunctionOk = FALSE;

    ___try
    {
        DWORD dwNumBytesRead;
        BOOL bOk;
        hFile = CreateFile(TEXT("SOMEDATA.DAT"), GENERIC_READ,
                    FILE_SHARE_READ, NULL, OPEN_EXISTING, 0, NULL);
        if(hFile == INVALID_HANDLE_VALUE) 
            __leave;

        bOk = ReadFile(hFile, pvBuf, 1024, &dwNumBytesRead, NULL);
        if(!bOk || (dwNymBytesRead == 0))
            __leave;
        
        // ...
        bFunctionOk = TRUE;
    }
    __finally
    {
        if(pvBuf != NULL)
            VirtualFree(pvBuf, MEM_RELEASE | MEM_DECOMMIT);

        if(hFile != INVALID_HANDLE_VALUE)
            CloseHandle(hFile);    
    }

    return(bFunctionOk);
}
```
__leave 키워드를 사용하면 로컬 언와인드가 수행되지 않도록 코드를 작성할 수 있다. try 내에서 __leave 키워드를 사용하면 try 블록의 가장 마지막 부분으로 이동하면서 자연스럽게 제어 흐름이 finally로 이동한다. 따라서 어떠한 추가 비용도 발생하지 않게 된다. 이를 로컬 언와인드 없이도 예외처리를 깔끔하게 처리할 수 있다.

## **3) 주의 사항**

위와 같이 종료 처리기를 리소스 해제에 활용할 때는 try 진입 전 리소스에 대한 핸들 값을 모두 유효하지 않은 값으로 초기화 해야 한다. 또, finally에서는 이 핸들 값을 확인하여 성공적으로 리소스 할당이 이루어졌는지 판단하고 리소스를 해제해야 한다. 혹은 플래그를 이용해서 리소스가 할당되었을 때만 삭제되도록 할 수도 있다. 

finally가 수행되는 상황은 제어 흐름이 자연스럽게 이동한 상황, 로컬 언와인드, 글로벌 언와인드로 분류된다. finally 내에서만 쓸 수 있는 내장 함수, AbnormalTermination를 호출할 경우 자연스러운 상황에선 FALSE를, 언와인드 상황에선 TRUE를 반환한다. 이 함수는 CRT에 포함되지 않으며, 컴파일러에 의해서만 인식되는 특수한 함수이다. 

참고로 내장 함수는 함수 호출 코드를 만드는 대신 인라인 코드를 삽입하므로 코드 영역 크기가 증가하며 수행 속도가 빨라진다. memcpy의 경우 내장 함수와 일반 함수 두가지 버전이 존재하는데, 컴파일 스위치로 /Oi를 지정하면 내장 함수가 되고 지정하지 않을 경우 일반 함수로 사용된다. 

<br/>

# **3. Exception Handling**

## **1) 예외 필터와 예외 처리기**

```c++
__try
{

}
___except(exception filter)
{

}
```

try 블록은 finally 블록 혹은 except 블록을 가져야 하나 둘을 동시에 가질 수는 없다. 종료 처리기와 달리 예외 처리기는 OS에 의해 직접 수행되며, 컴파일러는 예외 필터 평가와 같은 최소한의 작업만 수행한다. 예외 처리기는 종료 처리기와 달리 try 블록 내에 return, goto 등을 사용해도 성능 저하가 생기지 않는다. 

예외가 발생하면 except 블록으로 제어를 이동하여 예외 필터 식을 평가하며, 이 값은 EXCEPTION_EXECUTE_HANDLER(1), EXCEPTION_CONTINUE_EXECUTION(0), EXCEPTION_CONTINUE_SEARCH(-1) 중 하나이다. 예외 필터 내부는 가능한 간단하게 구성하는 것이 좋다. 예를 들어 힙 손상이 일어났는데 필터 내부에서 많은 코드를 수행하려고 하면 프로세스가 멈추거나 통지 없이 종료될 수 있다. 

## **2) EXCEPTION_EXECUTE_HANDLER**

EXCEPTION_EXECUTE_HANDLER는 이 예외 처리를 위한 코드가 준비되어 있고 이를 바로 수행하려는 상황에서 사용된다. 이때 시스템은 글로벌 언와인드를 수행한 후 except 블록 내 코드를 수행한다. 이후 시스템은 예외가 처리되었다고 생각하여 사용자에게 예외 상황을 보여주지 않고 except 블록의 바로 다음 명령부터 수행을 재개한다. 이를 효율적으로 사용한 예제는 아래와 같다. 

```c++
int RobustHowManyToken(const char* str)
{
    int nHowManyTokens = -1;
    char* strTemp = NULL;

    __try
    {
        strTemp = (char*)malloc(strlen(str) + 1);
        strcpy(strTemp, str);
        char* pszToken = strtok(strTemp, " ");
        for(; pszToken != NULL; pszToken = strtok(NULL, " "))
            nHowManyTokens++;
        nHowManyTokens++;
    }
    __except(EXCEPTION_EXECUTE_HANDLER) { }

    free(strTemp);
    return(nHowManyTokens);
}
```

잘못된 접근 때문에 데이터 오염, 접근 예외로 인한 프로세스 종료 등의 가능성이 있을 때, 위와 같이 처리하면 잘못된 접근이 있어도 데이터 오염, 프로세스 종료를 일으키는 대신 except로 넘어간 뒤 그 아래에서 free로 임시블록을 삭제, NULL을 반환할 것이다. strTemp에 NULL이 들어가도 free(NULL)은 아무런 작업도 수행하지 않으므로 문제가 생기지 않는다. 이처럼 이 예외 필터에서는 시스템이 글로벌 언와인드를 수행한다. 

```c++
void FuncOStimpy()
{
    __try
    {
        FuncORen();
    }
    __except(EXCEPTION_EXECUTE_HANDLER)
    {
        MessageBox(...);
    }
}

void FuncORen()
{
    DWORD dwTemp = 0;
    __try
    {
        WaitForSigleObject(g_hSem, INFINITE);
        g_dwProtectedData = 5 / dwTemp;
    }
    __finally
    {
        ReleaseSemaphore(g_hSem, 1, NULL);
    }
}
```

위와 같이 try-except, try-finally가 중첩된 경우 글로벌 언와인드는 수행을 재개하기 위해 try-except 블록 이하의 모든 try-finally 블록을 수행한다. 위 상황에서는 세마포어를 얻은 직후 0으로 나누는 연산으로 인해 예외가 발생한다. 그럼 시스템은 제어권을 가져와서 except 블록을 가진 try 블록을 찾는데, 여기서 FuncORen은 finally 블록을 가지고 있으므로 FuncOStimpy의 try에 연결된 except 블록을 찾을 것이다. 

예외 필터를 평가한 후에는 값이 반환될 때까지 대기한다. EXCEPTION_EXECUTE_HANDLER가 반환되면 시스템은 FuncORen의 finally로부터 글로벌 언와인드를 시작한다. 이는 except 블록 내 코드가 실행되기 전에 수행되며, 가장 안쪽의 try 블록부터 시작하여 역방향을 finally를 찾으며 진행된다. 계속 역방향으로 이동하여 수행해야 할 finally를 찾고, 더 이상 없을 경우 except로 예외를 처리한다. 


```c++
void FuncMonkey()
{
    __try
    {
        FuncFish();
    }
    __except(EXCEPTION_EXECUTE_HANDLER)
    {
        MessageBeep(0);
    }
    MessageBox(...);
}

void FuncFish()
{
    FuncPheasant();
    MessageBox(...);

}

void FuncPheasant()
{
    __try
    {
        strcpy(NULL, NULL);
    }
    __finally
    {
        return;
    }
}
```
위와 같이 finally에 return이 있으면 글로벌 언와인드가 중단된다. 따라서 FuncPheasant는 FuncFish로 반환되며, FuncFish는 수행을 재개하여 메시지 박스를 띄울 것이다. 또 예외가 전달되지 않았으므로 FuncMonkey의 MessageBeep은 호출되지 않는다. 일반적인 경우에는 이처럼 finally 내에서의 return 사용은 지양하는 것이 좋다. 


## **3) EXCEPTION_CONTINUE_EXECUTION**

```c++
TCHAR g_szBuffer[100];

void FuncA()
{
    TCHAR *pchBuffer = NULL;
    int x = 0;

    __try 
    {
        *pchBuffer = TEXT('A');
        x = 5 / x;
    }
    __except(OilFilter(&pchBuffer))
    {
        MessageBox(..., "Exception!", ...);
    }
}

LONG OilFilter(TCHAR **ppchBuffer)
{
    if(*ppchBuffer == NULL)
    {
        *ppchBuffer = g_szBuffer;
        return(EXCEPTION_CONTINUE_EXECUTION);
    }
    return(EXCEPTION_EXECUTE_HANDLER);
}
```

위 예제는 *pchBuffer = TEXT('A'); 예외의 경우, *ppchBuffer == NULL 이므로 전역 버퍼를 연결한 후 EXCEPTION_CONTINUE_EXECUTION을 반환한다. 이 필터가 except에 들어오면 예외를 유발했던 명령을 다시 한번 수행한다. 반면 x = 5 / x; 예외의 경우 EXCEPTION_EXECUTE_HANDLER를 반환하여 except 안의 구문을 수행하도록 한다. 

이 필터는 주의하여 사용해야 한다. *pchBuffer = TEXT('A');는 일반적으로 mov eax, dword ptr [pchBuffer] 와 mov eax, 'A' 두 줄로 구성되며 mov eax, 'A'에서 예외가 발생하므로 앞 명령부터 수행하는 게 아니라 이 명령만 다시 수행하게 된다. 이 경우 레지스터에는 새로운 주소 값이 할당되지 않았으므로 다시 예외가 발생하며 무한 루프에 빠지게 된다. 

컴파일러가 위 코드를 최적화 하는 경우 정상적으로 동작할 수도 있지만 이 역시 완전히 보장되는 것은 아니다. 따라서 이 필터를 쓸 땐 어셈블리 단위까지 고려하여 정상적으로 동작할 것인지 고민하고 사용하는 게 좋다. 참고로 메모리 상 예약된 영역에 대해 부분적으로 커밋하려는 예외가 발생한 경우 이 필터를 통한 처리가 문제 없이 정상적으로 처리된다. 

시스템은 스레드 스택으로 활용할 주소 공간을 예약, 커밋할 때 내부적으로 SEH를 이용한다. 스레드가 커밋하지 않은 저장소에 접근하면 예외가 발생하는데, 이때 시스템은 VirtualAlloc 호출로 추가적인 저장소를 커밋한 뒤 EXCEPTION_CONTINUE_EXECUTION을 반환한다. 그리고 다시 한번 CPU 명령을 수행하면 이번엔 정상적으로 메모리 공간에 접근할 수 있게 되며, 프로세스 종료 없이 스레드가 계속해서 수행된다. 

## **4) EXCEPTION_CONTINUE_SEARCH**


```c++
TCHAR g_szBuffer[100];

void FuncA()
{
    TCHAR *pchBuffer = NULL;

    __try 
    {
        FuncB(pchBuffer);
    }
    __except(OilFilter(&pchBuffer))
    {
        MessageBox(...);
    }
}

void FuncB(TCHAR *sz)
{
    __try 
    {
        *sz = TEXT('\0');
    }
    __except(EXCEPTION_CONTINUE_SEARCH)
    { }
}

LONG OilFilter(TCHAR **ppchBuffer)
{
    if(*ppchBuffer == NULL)
    {
        *ppchBuffer = g_szBuffer;
        return(EXCEPTION_CONTINUE_EXECUTION);
    }
    return(EXCEPTION_EXECUTE_HANDLER);
}
```
FuncB가 NULL에 '\0' 할당을 시도하면 이번에는 FuncB의 예외 필터가 수행된다. 이 예외 필터는 이전에 진입했던 try 블록 중 except를 가진 블록으로 이동하여 해당 try 블록의 예외 필터를 호출하며,  try - finally는 검색 대상에서 제외된다. 즉, FuncA의 예외 필터인 OilFilter를 이용해 필터 값을 평가하게 되며, ppchBuffer가 NULL이므로 고 EXCEPTION_CONTINUE_EXECUTION를 반환할 것이다. 

달라지는 것은, 이 경우 Continue 하는 지점이 FuncB의 *sz = TEXT('\0')이 된다. 그러나 FunB의 지역 변수인 sz는 예외 필터의 동작에도 올바른 값으로 변경되지 않기 때문에 다시 예외를 발생시킨다. 이렇게 동일한 절차로 OilFilter가 다시 호출되고, 이때는 ppchBuffer가 NULL이 아니기 때문에 EXCEPTION_EXECUTE_HANDLER를 반환하여 FuncA except 안의 MessageBox가 호출된다.

## **5) GetExceptionCode**

```c++
__except (
   ((GetExceptionCode() == EXCEPTION_ACCESS_VIOLATION) ||
    (GetExceptionCode() == EXCEPTION_INT_DIVIDE_BY_ZERO)) ? 
    EXCEPTION_EXECUTE_HANDLER:EXCEPTION_CONTINUE_SEARCH)
{
    switch(GetExceptionCode())
    {
    case EXCEPTION_ACCESS_VIOLATION:
        // ...
        break;
    case EXCEPTION_INT_DIVIDE_BY_ZERO:
        // ...
        break;
    }
}
```

예제처럼 GetExceptionCode 내장 함수로 예외를 식별하고 except에서 활용할 수 있다. 이 내장 함수는 예외 필터 내부 및 예외 처리기 내부에서만 호출할 수 있으며, 예외 필터 함수 내부에서 호출할 경우 컴파일 에러가 발생한다. 반환 값은 DWORD로 31-30bit는 심각도 표현용으로, 29bit는 정의자 구분용으로, 15-0bit 예외 코드로 쓰인이며, 크게 메모리, 예외 필터, 디버깅, 정수, 부동소수점에 대한 예외로 분류된다. 

## **6) GetExceptionInformation**

예외가 발생하면 OS는 EXCEPTION_RECORD, CONTEXT, EXCEPTION_POINTERS 세 구조체를 예외 유발 스레드 스택에 삽입한다. EXCEPTION_RECORD는 예외와 관련된 CPU 독립적인 정보를, CONTEXT는 CPU 의존적인 정보를 갖고 있다. EXEPTION_POINTERS 구조체는 스택에 삽입된 저 두 구조체를 가리키는 데이터 멤버를 갖는다. GetExceptionInformation 함수를 통해 EXEPTION_POINTERS 구조체 포인터를 얻을 수 있다. 

```c++
__except(
    exceptRec = *(GetExceptionInformation())->ExceptionRecord,
    context = *(GetExceptionInformation())->ContextRecord,
    EXCEPTION_EXECUTE_HANDLER
)
{
    switch(exceptRec.ExceptionCode)
    {
        //...
    }
}
```
위의 데이터 구조체들은 예외 필터를 처리하는 동안에만 유효하고 예외 처리기로 제어가 넘어가면 파괴된다. 따라서 이 함수는 예외 필터 내에서만 사용할 수 있으며, 예외 처리기 블록에서 이 정보들이 필요하다면 해당 임시 변수에 저장하여 사용해야 한다. 

ExceptionRecord는 예외 코드, 예외 플래그, 처리되지 않은 또 다른 EXCEPTION_RECORD 구조체, 예외 유발 명령의 주소, 예외와 관련된 매개변수 개수 등이 들어있다. 간혹 예외 처리 과정이 또 다른 예외를 유발하기도 하는데, 또 다른 EXCEPTION_RECORD 구조체는 이전에 처리되지 않은 예외에 대한 추가 정보를 제공한다. 없는 경우 이 값은 NULL이 된다. 

ContextRecord는 플랫폼 의존적인 정보를 담고 있어서 CPU에 따라 내용이 달라진다. 기본적으로는 CPU 레지스터 각각을 저장하는 멤버들이 있어서 예외 발생 시 이 구조체를 이용해 자세한 정보를 확인할 수 있다. 그러나 해당 구조체의 특징 및 머신에 적합한 코드에 대한 지식이 있어야 구조체 내용을 이해할 수 있다. 이 구조체를 활용하는 편리한 방법은 코드 내 여러개의 #ifdef를 추가하는 것이라고 한다.  

<br/>

# **4. 그 외**

## **1) 소프트웨어 예외**

CPU가 발생시킨 예외를 하드웨어 예외, OS나 애플리케이션이 자체적으로 발생시킨 예외를 소프트웨어 예외라고 한다. 코드 내에서 예외를 강제적으로 발생시킴으로써 특정 함수가 실패했음을 호출자에게 알려줄 수 있다. 소프트웨어 예외는 RaiseException 함수 호출로 발생시킬 수 있으며, 잡는 방법은 하드웨어 예외를 잡는 방법과 동일하다. 

```c++
VOID RaiseException(
    DWORD dwExceptionCode,
    DWORD dwExceptionFlags,
    DWORD nNUmberOfArguments,
    CONST ULONG_PTR *pArguments
);
```
dwExceptionCode에는 표준 윈도우 에러 코드를 따르는 예외 값을 전달한다. dwExceptionFlags에는 0 혹은 EXCEPTION_NONCONTINUABLE로 EXCEPTION_CONTINUE_EXECUTION를 반환 가능 여부를 전달한다. 0을 전달한 경우 CONTINUE라면 RaiseException 바로 다음부터 재개한다. pArguments에 주소를 (불필요하면 NULL을), nNUmberOfArguments에 주소가 가리키는 ULONG 개수를 지정하면 예외에 대한 추가 정보를 전달할 수 있다.  

## **2) UnhandledExceptionFilter**

모든 예외 필터가 CONTINUE_SEARCH를 반환하면 처리되지 않은 예외가 된다. 이때 CRT를 쓴다면 진입점 함수 호출 전에 CxxUnhandledExceptionFIlter라는 전역 예외 필터를 설치하는데, 이는 C++ 예외인지 여부만 확인하여 맞을 경우 UnhandledExceptionFilter 함수를 호출하는 abort 함수를 수행한다. C++ 예외가 아닐 경우에는 CONTINUE_SEARCH를 반환하며 윈도우가 처리되지 않은 예외로 규정한다. 

```c++
PTOP_LEVEL_EXCEPTION_FILTER SetUnhandledExceptionFilter(
    PTOP_LEVEL_EXCEPTION_FILTER pTopLevelExceptionFilter
);
```

이 윈도우 함수를 사용하면 처리되지 않은 예외로 규정하기 전에 예외를 처리할 수 있다. 이 함수는 프로세스 초기화 시 호출되며, 이후 모든 스레드의 처리되지 않은 예외들을 매개변수의 최상위 필터함수로 전달한다. 이는 EXCEPTION~ 구분자를 반환하는데, 새 예외 필터를 설치하면 이전 예외 필터 주소를 반환한다. 반환된 필터 함수를 포함한 DLL이 언로드 된 상태라면 호출시 문제가 되므로 반환 함수 호출은 지양해야 한다. 

스택 오버플로, 해제되지 않은 동기화 오브젝트, 해제되지 않은 힙 데이터 등의 예외는 프로세스가 손상된 상태일 수 있는데, 이 경우에는 저 함수 안에서도 제대로 된 예외 처리를 할 수 없다. 또, 힙이 손상되었을 수 있으므로 필터 함수 내에서의 동적 메모리 할당은 피하는 것이 좋다. 따라서 필터 함수 내에서는 가능한 최소한의 작업만 수행해야 한다.

```c++
VOID BaseThreadStart(PTHREAD_START_ROUTINE pfnStartAddr, PVOID pvParam)
{
    __try
    {
        ExitThread((pfnStartAddr)(pvParam));
    }
    __except (UnhandledExceptionFilter(GetExceptionInformation()))
    {
        ExitProcess(GetExceptionCode());
    }
}
```
모든 스레드는 NTDLL.dll의 BaseThreadStart를 호출하며, 이는 모든 필터가 CONTINUE_SEARCH를 반환하는 경우 EXCEPTION~ 구분자를 반환하는 UnhandledExceptionFilter 필터 함수를 호출한다. 이 함수는 다섯가지 과정을 거치는데 첫째로, 스레드 쓰기 작업 접근 위반으로 이 함수가 실행된 경우, VirtualProtect를 호출하여 해당 리소스 페이지를 PAGE_READWRITE로 변경하고 CONTINUE_EXECUTION을 반환한다. 

둘째로, 애플리케이션이 디버거의 제어 하에 수행되고 있는 경우 CONTINUE_SEARCH를 반환하여 디버거에게 이 시점까지 예외가 처리되지 않았음을 통지한다. 디버거는 EXCEPTION_RECORD 구조체의 ExceptionInformation 멤버를 전달 받아 어떤 명령이 예외를 유발했는지 사용자에게 전달한다. 디버거 제어 여부는 IsDebuggerPresent 함수를 사용하여 확인할 수 있다. 

셋째로, 전역 예외 필터 함수를 호출해서 그 함수가 EXECUTE_HANDLER, CONTINUE_EXECUTION을 반환하면 그 값을 시스템에게 그대로 반환, CONTINUE_SEARCH를 반환하면 네번째 작업을 수행한다. 이때 CxxUnhandled 필터는 명시적으로 Unhandled~ 함수를 호출하기 때문에, 무한 재귀 호출을 막기 위해 Unhandled~ 함수 호출에 앞서 SetUnhandledExceptionFilter(NULL)을 호출한다. 

CRT를 사용하는 경우 런타임은 스레드 진입점 함수를 try로 감싸고 except에서 CRT의 _XcpFilter를 호출하게 만든다. _XcpFilter는 Unhandled~ 를 호출하며, Unhandled~ 는 전역 필터가 설치된 경우 이를 호출한다. 따라서 전역 필터가 설치 되었는데 _Xcp~가 처리되지 않은 예외를 발견하고 전역 필터가 CONTINUE_SEARCH를 반환하면, 처리되지 않은 예외가 BaseThreadStart의 예외 필터에 도달하여 Unhandled~ 가 두번 호출 된다.

넷째로, 처리되지 않은 예외를 디버거에 다시 통지한다. 앞선 작업에서 전역적으로 설치된 예외 필터 함수는 디버거를 수행하고, 이후 처리되지 않은 예외를 유발한 스레드의 프로세스에 디버거를 붙인다. 이제 처리되지 않은 예외 필터가 COUNTINUE_SEARCH를 반환하면 디버거에게 예외를 통지하게 된다. 

마지막으로, 프로세스 내 특정 스레드가 SEM_NOGPFAULTERRORBOX 플래그를 인자로 SetErrorMode를 호출한 경우 Unhandled~ 는 EXECUTE_HANDLER를 반환한다. 그럼 글로벌 언와인드가 진행되어 대기 중이던 finally 블록이 모두 수행되며 프로세스가 조용히 종료된다. 잡의 제한 사항으로 JOB _OBJECT _LIMIT _DIE _ON _UNHANDLED _EXCEPTION 플래그가 설정된 경우에도 동일하게 진행된다. 

이 모든 과정에서도 처리되지 못한 경우, Window Error Reporting으로 제어를 전달한다. 비스타 이전에는 Unhandled~ 함수가 직접 예외를 MS 서버에 전송했지만, 그 이후로 단순히 CONTINUE_SEARCH를 반환한다. 그럼 이후에 커널이 제어를 넘겨받아 유저 모드 스레드에 의해 처리되지 않은 예외가 있음을 감지해 WerSvc라는 서비스로 예외 통지를 전달한다. 

WerSvc 통지는 문서화되지 않은 메커니즘을 사용한다. 우선 전달된 내용이 처리될 때까지 문제의 스레드가 수행되지 않도록 하고, WerSvc가 CreateProcess를 호출하여 WerFault.exe를 수행, 종료시까지 대기한다. 이것은 문제 보고서를 구성하여 서버로 전송, 다이얼로그 박스를 출력하여 사용자가 애플리케이션을 닫을지 혹은 디버거를 붙일지 결정하도록 한다. 닫을 경우 TerminateProcess를 사용한다. 

## **3) JIT, Just In Time 디버깅**

대부분의 OS는 디버거를 이용해 수행한 프로세스만 디버깅 할 수 있지만 윈도우는 수행 중인 프로세스에 디버거를 붙이는 JIT 디버깅 기능을 제공한다. vsjitdebugger.exe - p PID 혹은 윈도우 작업 관리자 - 프로세스 - 디버그를 통해 이를 수행할 수 있다. 디버거로 실행한 것과 동일하게 스레드 개수, 로드된 DLL 정보까지 디버거에게 알려주며, 이걸로 위와 같은 상황에서도 실패 시점의 문제를 이어서 디버깅할 수 있다. 

Unhandled~ 예외 마지막 상황에서 디버깅을 택하면 WerFault.exe는 bInheritHandles를 TRUE로 논시그널 수동 리셋 이벤트를 생성한다. 이를 통해 WerFault.exe의 자식 프로세스는 동일 이벤트를 상속받을 수 있게 되며, 이후 WER은 기본 디버거를 찾아 수행해줌으로써 문제의 프로세스에 디버거를 붙인다. 이후엔 전역변수, 지역변수, 정적변수 내용을 확인할 수 있고, 중단점, 호출 트리 확인, 프로세스 재시작 등을 수행할 수 있다. 

위 상황에서 예외를 유발한 코드는 WerSvc로부터 ALPC 호출 반환을 기다리는데, 이때 ALPC는 WaitForSingle ObjectEx를 통해 스레드를 alertable 상태로 대기시킨다. 그래야만 스레드가 자신의 APC 큐에 삽입된 요청을 수행할 수 있다. 비스타 이전에는 예외 유발 스레드 외의 스레드들은 정지되지 않았으나, 이는 손상된 컨텍스트 내에서 스레드가 수행되게 함으로써 더 많은 예외를 유발했기에 이와 같이 수정되었다. 

이런 이유로 윈도우는 일단 모든 스레드를 정지시키며, 디버거에게 처리되지 않은 예외를 전달하기 전까지는 CPU 시간을 할당하지 않는다. 디버거가 초기화되면 디버거는 SetEvent를 호출하여 이벤트를 시그널 상태로 변경한다. 차례로 WerFault.exe는 종료되고, WerSvc는 ALPC 호출을 반환하고, 스레드는 깨어난다. 이후 커널은 디버거에게 처리되지 않은 예외를 전달, 디버거는 이 통지를 수신하여 예외를 유발한 명령으로 이동한다.

## **4) VEH 와 Continue Handler**

```c++
PVOID AddVectoredExceptionHandler (
    ULONG bFirstInTheList,
    PVECTORED_EXCEPTION_HANDLER pfnHandler
);

ULONG RemoveVectoredExceptionHandler (PVOID pHandler);
```
윈도우는 SEH와 함께 VEH, 벡터화된 예외 처리 메커니즘을 제공한다. 이는 언어에 종속적인 키워드를 사용하지 않고도 예외 발생 시 호출할 함수를 프로그램에서 등록/삭제할 수 있게 한다. 이때 표준 SEH로 처리되지 않은 예외도 포함할 수 있다. pfnHandler는 벡터화된 예외 처리기를 가리키는 함수 포인터로, 위 함수들을 통해 예외 시 호출할 함수 리스트에 예외 처리기를 등록/삭제할 수 있다.  

```c++
PVOID AddVectoredContinueHandler(
    ULONG bFirstInTheList,
    PVECTORED_EXCEPTION_HANDLER pfnHandler
)

ULONG RemoveVectoredContinueHandler (PVOID pHandler);
```
VEH를 쓰면 Continue Handler를 등록하여 처리되지 않은 예외 통지를 받을 수 있다. 그럼 SetUnhandled~에 의해 처리되지 않은 예외에 대한 전역 필터가 SEARCH를 반환한 이후, 리스트 내의 Continue Handler가 차례로 호출되며 통지를 받는다. Continue Handler가 SEARCH를 반환하면 리스트 내 나머지 함수들이 차례로 수행되며, EXECUTION을 반환하면 나머지 함수들의 수행 없이 예외 유발 명령을 다시 수행하게 된다.

## **5) SEH와 C++ 예외**

SEH는 OS 기능이므로 모든 언어에서 쓸 수 있고, C++ EH는 C++에서만 사용할 수 있다. C++로 개발한다면 C++ EH를 권장하는데, 이는 언어에서 제공하는 기능이라 컴파일러가 C++ 오브젝트의 파괴자 호출 코드 생성 작업 등을 자동으로 처리해주기 때문이다. 그러나 Visual C++ 컴파일러는 try를 __try로, catch를 __except로 변경하며, throw 시 RaiseException을 호출하는 등 C++ EH를 SEH로 구현하고 있으니 참고하면 좋다. 

## **6) 예외와 디버거**

디버거에 대한 예외 통지에는 두가지 형태가 존재한다. 예외가 생기면 OS는 지체 없이 디버거에게 통지하는데 이를 첫번째 통지라고 한다. 첫번째 통지가 발생하면 디버거는 스레드에게 예외 필터를 찾도록 지시하고, 모든 필터가 SEARCH일 경우 OS는 디버거에게 마지막 통지를 전달한다. 이는 예외를 디버깅할 때 개발자에게 보다 많은 제어권을 부여하기 위함이다. 디버거의 예외 다이얼로그 박스를 이용하면 모든 예외를 종류별로 나누어 확인할 수 있으며, 첫번째 예외 통지 발생시 디버거가 어떻게 대응할지 결정할 수 있다. 
