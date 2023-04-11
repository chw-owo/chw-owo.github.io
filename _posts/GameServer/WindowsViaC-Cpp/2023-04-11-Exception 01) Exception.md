---
title: Exception 01) Exception
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. SEH, Structed Exception Handling**

SEH를 사용하면 예외 처리를 주 작업에서 분리하여, 수행하고자 하는 작업에 집중할 수 있다. 수행 중 무언가 비정상적으로 동작하면 시스템이 이를 확인하여 문제를 알려준다. 그러나 컴파일러가 SEH를 위한 코드와 테이블, 콜백 함수, 스택 프레임 등을 구성해야 한다는 단점이 있다. SEH의 세부 구현 방식은 컴파일러마다 조금씩 다르며, 이 포스트는 MS Visual C++ 컴파일러 문법을 따른다. 

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

## **2) EXCEPTION_EXECUTE_HANDLER**

## **3) EXCEPTION_CONTINUE_EXECUTION**

## **4) EXCEPTION_CONTINUE_SEARCH**

## **5) GetExceptionCode**

## **6) GetExceptionInformation**

## **7) UnhandledExceptionFilter 내부**

## **8) 소프트웨어 예외**

<br/>

# **4. Etc Exception**

## **1) JIT 디버깅**

## **2) 벡터화된 예외와 컨티뉴 처리기**

## **3) C++ 예외와 구조적 예외**

## **4) 예외와 디버거**

