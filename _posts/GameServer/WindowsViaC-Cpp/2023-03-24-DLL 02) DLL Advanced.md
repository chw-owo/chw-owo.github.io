---
title: DLL 02) DLL Advanced
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. DLL의 명시적 로드**

## **1) 개요**

DLL 파일 이미지를 프로세스 주소 공간에 매핑하는 방법은 두가지가 있다. 하나는 로더가 필요한 DLL을 묵시적으로 로드하여 애플리케이션이 DLL에 포함된 심벌을 단순 참조하는 것이다. 두번째는 애플리케이션이 필요한 DLL을 명시적으로 로드하는 것으로, 수행 중에 해당 함수의 가상 메모리 주소를 획득하여 호출하는 것이다. 

## **2) 명시적 로드 관련 함수**

```c++
HMODULE LoadLibrary(PCTSTR pszDLLPathName);
```

```c++
HMODULE LoadLibraryEx(
    PCTSTR pszDLLPathName,
    HANDLE hFile,
    DWORD dwFlags
);
```
이 함수들을 호출하면 사용자 시스템에서 파일 이미지를 검색, 호출한 주소 공간에 DLL 파일 이미지 매핑을 시도한다. 만일 한 프로세스 내에서 이미 LoadLibrary로 매핑한 DLL 파일 이미지를 또 다시 호출할 경우, 시스템은 이를 두번 매핑하는 대신 DLL의 사용 카운트 값만 증가시킨다. 

이때 시스템은 DLL 사용 카운트를 프로세스별로 유지하기 때문에 서로 다른 프로세스가 이를 사용할 경우 DLL은 각기 다른 주소 공간에 매핑되며 사용 카운트도 각 프로세스에서 1로 유지된다. 이때 GetModuleHandle을 이용하면 해당 프로세스에 DLL이 로드되어 있는지 확인할 수 있다. 

이 함수는 파일 이미지가 매핑된 주소를 HMODULE로 반환한다. HMODULE은 HINSTANCE와 완전히 동일하게 주소를 가리키며, 실제로 DllMain 진입점 함수의 HINSTANCE 인자도 파일 메모리가 매핑된 주소를 가진다. 실패하면 NULL을 반환하며 GetLastError로 원인을 알 수 있다. 

```c++
DWORD GetModuleFileName(
    HMODULE hInstModule,
    PTSTR pszPathName,
    DWORD cchPath
);
```

DLL 파일의 주소 값을 안다면 GetModuleFileName으로 DLL 전체 경로명을 얻을 수 있다. hInstModule에 HINSTANCE 혹은 HMODULE을, pszPathName에 반환 받을 버퍼 주소를 지정한다. 이때 hInstModule에 NULL을 전달하면 현재 실행 중인 애플리케이션 전체 경로명을 반환한다. 

hFile은 예약된 것이므로 NULL을 전달하고 dwFlags에는 플래그를 전달한다. 이때 다른 플래그로 동일 DLL을 호출할 경우 새 위치에 다시 매핑되며 각기 다른 HMODULE을 뱉는데 이에 GetModuleFileName을 사용하면 0을 반환한다. 이는 해당 DLL에 대해서는 동적으로 함수를 사용할 수 없음을 나타낸다. 

## **3) 명시적 로드 플래그**

플래그에 기본값을 설정할 경우 0을 전달하며, 올 수 있는 플래그 예시는 아래와 같다. 

**DONT_RESOLVE_DLL_REFERENCES**

플래그는 함수 호출 프로세스 주소 공간에 DLL을 매핑할 것을 지정한다. 보통 DLL이 매핑되면 시스템은 일반적으로 DllMain을 호출하여 DLL을 초기화하는데, 이 플래그를 사용할 경우 매핑까지만 수행하고 DllMain은 호출하지 않는다. 

또, DLL이 다른 DLL을 import할 경우 원래는 이것도 시스템이 함께 로드해주는데, 이 경우엔 다른 DLL을 자동으로 로드하지 않는다. DLL이 export 하는 함수들은 내부 자료구조가 완전히 초기화되고 추가 DLL 파일이 로드되기 전엔 호출할 수 없으므로 이 플래그는 사용하지 않는 게 좋다. 

**LOAD_LIBRARY_AS_DATAFILE**

이 플래그는 DONT_RESOLVE_DLL_REFERENCES와 마찬가지로 데이터 파일처럼 DLL을 매핑까지만 수행하고 초기화 작업을 하지 않는다. 더불어 DLL 정보를 이용한 페이지 보호 특성 설정 작업도 수행하지 않는다. 이걸로 로드된 DLL에 대해 GetProcAddress 함수를 호출하면 NULL이 반환된다. 

이는 DLL이 리소스만 갖고 함수를 갖지 않은 경우에 유용하게 쓸 수 있다. 또, exe 파일 내 리소스를 사용하고 싶을 때도 이 플래그를 사용하면 exe 파일 이미지를 매핑만 하므로 리소스에 접근할 수 있게 된다. exe 파일은 DllMain 함수가 없으므로 매핑하려면 반드시 이 플래그를 써야 한다. 

**LOAD_LIBRARY_AS_DATAFILE_EXCLUSIVE**

이 플래그는 LOAD_LIBRARY_AS_DATAFILE와 유사하지만 바이너리 파일을 사용하는 동안 다른 애플리케이션이 해당 파일을 수정하지 못하도록 배타적으로 파일에 접근한다는 점에서 차이가 있다. 이를 사용하면 더 안전하게 바이너리 파일을 사용할 수 있다. 

**LOAD_LIBRARY_AS_IMAGE_RESOURCE**

이 플래그는 LOAD_LIBRARY_AS_DATAFILE와 유사하지만 DLL을 로드할 때 RVA에 접근하는 방식에 차이가 있다. 이를 사용하면 메모리 영역에 DLL을 로드한 후 심벌의 시작 주소를 변경하지 않은 상태에서 RVA 값에 직접 접근할 수 있다. 이는 PE 내 여러 섹션을 분석할 때 사용된다. 

**LOAD_WITH_ALTERED_SEARCH_PATH**

이 플래그를 사용하면 LoadLibraryEx 함수의 DLL 검색 알고리즘을 변경할 수 있다. 

**-** pszDLLPathName에 \ 문자가 포함되어 있지 않다면 기존 검색 순서를 따른다. 

**-** pszDLLPathName가 \ 문자를 포함한 전체 경로명 혹은 네트워크 공유 위치를 나타내는 값이라면 지정 위치에 대해서만 DLL을 로드하고 다른 경로에 검색을 시도하지 않는다. 

**-** pszDLLPathName가 확장자를 갖지 않은 경우 DLL 확장자를 가진 것처럼 프로세스 현재 디렉토리, 윈도우 시스템 디렉토리, 16bit 시스템 디렉토리, 윈도우 디렉토리, PATH 환경변수 디렉토리를 순차로 검색한다. 

**-** pszDLLPathName가 ., .. 문자를 포함하면 앞서 말한 검색 단계 별 상위 폴더 위치에서 해당 문자 의미를 반영한 상대 경로를 검색한다. 

그러나 이 플래그를 사용하거나 애플리케이션의 위치를 변경하기보다는 SetDllDirectory 함수로 DLL 파일 로드 위치를 지정하는 것이 좋다. 이 함수를 사용하면 애플리케이션을 포함한 디렉토리 다음 순서로 SetDllDirectory에서 지정한 디렉토리를 검색하게 된다. 

SetDllDirectory를 사용하면 동일 파일 명의 다른 DLL을 로드할 위험 없이 체계적으로 관리할 수 있다. 비어있는 문자열을 전달하면 검색 단계로부터 현재 디렉토리를 제거하고, NULL을 전달하면 기본 검색 알고리즘을 사용한다. GetDllDirectory로 지정된 디렉토리명을 가져올 수 있다. 

**LOAD_IGNORE_CODE_AUTHZ_LEVEL**

이 플래그를 사용하면 실행 중 코드 권한 제어를 위해 윈도우 XP에서부터 소개된 WinSafer을 사용하지 않는다. 참고로 이 기능은 윈도우 비스타 UAC에 의해 흡수되었다. 

## **4) 명시적 DLL 모듈 언로드**

```c++
BOOL FreeLibrary(HMODULE hInstDll);
```
```c++
BOOL FreeLibraryAndExitThread(
    HMODULE hInstDll,
    DWORD dwExitCode    
);
```
스레드에서 더 이상 DLL 파일 심벌을 사용하지 않는다면 위 함수들로 언로드한다. FreeLibrary를 호출하면 DLL 사용 카운트를 1 감소한다. 

아래 함수는 말 그대로 FreeLibrary 직후에 ExitThread를 수행하는 함수이다. 이를 각각 호출하면 FreeLibrary가 반환과 동시에 ExitThread를 하려던 코드가 메모리 상에 남아있지 않아 접근 위반이 된다. 그러나 이 함수를 쓰면 다음에 수행할 코드가 Kernel32.dll 파일 내에 존재하므로 ExitThread를 호출할 수 있게 된다. 

## **5) export 심벌의 명시적 링크**

```c++
FARPROC GetProcAddress(
    HMODULE hInstDll,
    PCSTR pszSymbolName
);
```
DLL을 로드 후엔 이 함수로 심벌 주소를 얻어야 한다. 인자가 PCSTR인 것은 export section이 ANSI 문자열로 기록되기 때문이다. 이 값은 두가지 형태로 지정할 수 있는데, 하나는 '\0'으로 끝나는 문자열을 지정하는 것이고 하나는 심벌의 숫자를 전달하는 것이다. 숫자를 전달하는 것이 더 빠르긴 하지만 유지 관리 측면에서 첫번째 방법을 권장한다. 

```c++
PFN_DUMPMODULE pfnDumpModule = (PFN_DUMPMODULE)GetProcAddress(hDll, "DumpModule");

if(pfnDumpModule != NULL)
    pfnDumpModule(hDll);
```
이 함수는 심벌 주소 값을 반환하며 존재하지 않는 심벌일 경우 NULL을 반환한다. 위와 같은 형변환을 거치면 함수를 호출할 수 있게 된다. 

<br/>

# **2. DLL 진입점 함수**

## **1) DLLMain**

DLL은 하나의 진입점 함수를 가질 수 있다. 시스템은 정보 제공, 스레드 별 초기화 수행 및 정리 등을 목적으로 여러번에 걸쳐 진입점 함수를 호출한다. 리소스만 가진 DLL 파일은 진입점 함수를 구현할 필요가 없지만, DLL 파일이 추가적인 정보를 통지 받아야 하는 경우 다음과 같이 구현하면 된다. DllMain은 대소문자를 구분하므로 DLLMain으로 쓰지않게 유의해야 한다. 

```c++
BOOL WINAPI DllMain(HTINSTANCE hInstDll, DWORD fdwReason, PVOID fImpLoad){

    switch(fdwReason)
    {
    case DLL_PROCESS_ATTACH:
        // DLL이 프로세스 주소 공간에 매핑되고 있다. 
        break;

    case DLL_THREAD_ATTACH:
        // 스레드가 생성되고 있다.
        break;

    case DLL_THREAD_DETACH:
        // 스레드가 종료되고 있다. 
        break;

    case DLL_PROCESS_DETACH:
        // DLL이 프로세스 주소 공간에서 해제되고 있다.
        break;
    }

    return(TRUE); //DLL_PROCESS_ATTACH 경우에만 사용된다.
}
```
DllMain을 만들고 hInstDll에 핸들값을 전달하면 DLL이 자신을 초기화 하기 위해 이를 사용한다. DLL이 암시적으로 로드된 경우 fImpLoad에 0이 아닌 값이, 명시적으로 로드된 경우 0이 전달된다. fdwReason로는 이 함수 호출 이유를 전달하는데 DLL_PROCESS_ATTACH, DLL_THREAD_ATTACH, DLL_THREAD_DETACH, DLL_PROCESS_DETACH 중 하나를 전달한다. 

문서에서는 DllMain을 TLS 설정에만 사용하기를 명시하고 있는데, 이는 다른 DLL이 아직 DllMain을 호출하지 않았을 수 있기 때문이다. 따라서 이 안에 Load/FreeLibrary 등과 같은 타 DLL 함수를 호출하면 의존 관계 루프가 생길 수 있다. User, Shell, ODBC, COM, RPC, 소켓 그리고 생성자를 호출하는 전역, 정적 c++ 오브젝트도 호출하지 말아야 한다.

## **2) DLLMain의 fdwReason**

**- DLL_PROCESS_ATTACH**

DLL이 최초 매핑되면 이 값을 전달하여 DllMain을 호출한다. 이는 첫 매핑 때만 발생하며, 이후 LoadLibrary를 할 경우 사용 카운트 값만 증가시키고 DllMain을 재호출하진 않는다. 이 안에서는 힙 생성과 같은 프로세스와 관련된 초기화 작업만 수행해야 하며, 이 과정에서 생긴 핸들은 DLL 포함 함수들이 접근할 수 있도록 전역 변수에 저장하는 것이 좋다. 

이 경우 초기화 성공 여부에 따라 반환값이 결정된다. 힙을 생성한다고 하면 HeapCreate가 성공한 경우 TRUE, 실패할 경우 FALSE를 반환하도록 하는 식이다. fdwReason이 다른 값일 땐 함수 반환 값이 무시된다. DllMain 함수들 중 하나라도 초기화에 실패하면 프로세스는 종료될 것이며, 이 과정에서 로드된 파일 이미지들은 주소 공간에서 제거된다. 

DLL을 명시적으로 로드한 경우 시스템은 DLL 파일을 찾아서 매핑하고, LoadLibrary를 호출한 스레드로 하여금 이 값을 전달하여 DllMain을 호출하도록 한다. 이 과정 이후에야 LoadLibrary가 반환된다. 만일 초기화에 실패하면 시스템은 매핑을 해제하고, LoadLibrary는 NULL을 반환하여 이후 그 값에 접근했을 때 프로세스가 종료되도록 한다. 

**- DLL_PROCESS_DETACH**

DLL 사용 카운트가 0이 되어 매핑을 해제하면 이 값으로 DllMain이 호출된다. ExitProcess를 호출한 경우 그 스레드가 대신 수행한다. 이 처리가 완료되어야 스레드가 반환되기 때문에 여기서 무한 루프가 발생한다면 프로세스가 종료되지 못할 수 있다. 타 스레드가 TerminateProcess를 호출한 경우엔 이 함수가 호출되지 않는다. 

DLL_PROCESS_DETACH를 인자로 전달하는 경우에는 DllMain은 힙 해제와 같이 프로세스와 관련이 있는 정리 작업만 수행해야 한다. 만약 DLL_PROCESS_ATTACH로 DllMain을 호출했을 때 FALSE가 돌아왔다면 이 부분은 실행되지 않는다. 

**- DLL_THREAD_ATTACH**

프로세스내 새 스레드가 생성되면 시스템은 이 값을 전달하여 현재 매핑된 모든 DLL의 DllMain 함수를 호출한다. 이 과정을 통해 모든 DLL은 스레드별로 초기화 과정을 수행할 수 있게 된다. 새롭게 생성된 스레드는 모든 DllMain을 호출할 책임이 있으며, 이러한 통지를 전달 받은 이후에야 자신의 스레드 함수를 수행할 수 있게 된다. 

여러 스레드들이 수행 중인 상태에서 새 DLL이 주소 공간 내 매핑되는 경우, 기존 스레드에 대해 이 값으로 DllMain을 호출하지 않는다. 또, 주 스레드에 대해서도 이 값으로 DllMain을 호출하지 않는다. 프로세스가 처음 시작되어 매핑될 땐 이것 대신 DLL_PROCESS_ATTACH 통지가 전달되는 것에 유의하여 함수를 구성해야 한다. 

**- DLL_THREAD_DETACH**

ExitThread가 호출되면 시스템은 이 값을 인자로 매핑된 모든 DllMain을 실행한 후 스레드를 종료한다. 이를 통해 DLL은 스레드 단위의 정리 작업을 수행할 수 있다. 멀티스레드 애플리케이션 관리를 위해 사용한 데이터 블록은 이 시점에서 Free 하면 된다. 만약 이 안에서 무한 루프에 걸리면 스레드는 종료되지 못하니 주의해야 한다. TerminateThread가 호출된 경우에는 호출되지 않는다.

DLL 분리 이후 수행 중인 스레드가 있다면 이 스레드들에 대해서는 이 값을 인자로 DllMain이 호출되지 않는다. 이 경우 이 값을 인자로 DllMain이 호출되는 시점에 추가적으로 수행해야 하는 정리 작업이 있는지 확인해야 한다. 또, DLL 결합 시점에 DLL_THREAD_ATTACH로 통지 받지 못한 경우에도 DLL_THREAD_DETACH가 수신될 수 있다는 점에 주의해야 한다. 따라서 LoadLibrary를 호출한 스레드를 이용해 FreeLibrary를 호출해야 한다. 

## **3) DLLMain의 순차적 호출**

시스템은 순차적으로 DllMain을 호출한다. 스레드 A, B가 각각 스레드 C, D를 생성한다 했을 때, 시스템은 DLL_THREAD_ATTACH로 DllMain을 호출할 것이다. 만약 A가 B보다 먼저 CreateThread를 했다면 C가 DllMain 을 완전히 수행할 때까지 D는 일시 정지 될 것이다. 보통은 이 특성이 문제가 되지 않지만, 아래와 같은 코드에서는 데드락을 발생시킨다. 

```c++
BOOL WINAPI DllMain(HTINSTANCE hInstDll, DWORD fdwReason, PVOID fImpLoad){

    HANDLE hThread;
    DWORD dwThreadId;

    switch(fdwReason)
    {
    case DLL_PROCESS_ATTACH:
        hThread = CreateThread(NULL, 0, SomeFunction, NULL, 0, &dwThreadId);
        WaitForSingleObject(hThread, INFINITE);
        CloseHandle(hThread);
        break;

    case DLL_THREAD_ATTACH:
        break;

    case DLL_THREAD_DETACH:
        break;

    case DLL_PROCESS_DETACH:
        break;
    }

    return(TRUE);
}
```

위 코드에서 DllMain은 새 스레드를 생성하므로, DLL_THREAD_ATTACH로 DllMain을 다시 호출한다. 그러나 기존 스레드가 DLL_PROCESS_ATTACH에서 빠져나오지 않았으므로 새 스레드는 일시 정지가 된다. 동시에 WaitForSingleObject 때문에 현재 스레드는 새 스레드가 종료될 때까지 대기해야 되고, 새 스레드는 DllMain을 호출하지 못해 자신의 스레드 함수를 수행할 수 없으므로 데드락이 걸리게 된다. 

이를 해결하기 위해 첫번째로 고안할 수 있는 방법은 DLL_THREAD_ATTACH, DLL_THREAD_DETACH 통지를  DllMain로 전달되지 않게 하는 함수, DisableThreadLibraryCalls을 사용하는 것이다.

```c++
DisableThreadLibraryCalls(hInstDll);
hThread = CreateThread(NULL, 0, SomeFunction, NULL, 0, &dwThreadId);
WaitForSingleObject(hThread, INFINITE);
CloseHandle(hThread);
break;
```

그러나 이 방법도 데드락을 막지 못한다. 프로세스가 생성되면 시스템은 스레드 간 동기화에 사용하기 위해 프로세스끼리 공유되지 않는 그 프로세스만의 락을 생성한다. 그리고 이 락은 DllMain을 호출하는 여러 스레드에 대해 동기화를 수행하는데 이 과정에서 내부적으로  DLL_THREAD_ATTACH을 인자로 dllMain이 호출된다. 

상세하게 보자면, CreateThread를 호출했을 때 시스템은 내부적으로 뮤텍스로 WaitForSingleObject를 호출하고, 새 스레드가 뮤텍스를 얻으면 DLL_THREAD_ATTACH로 DllMain을 호출한 후에 ReleaseMutex를 한다. 따라서 데드락을 막기 위해서는 DllMain 내에서 WaitForSingleObject를 호출하지 않도록 소스 코드를 재설계해야만 한다. 

## **4) DLLMain과 CRT Library**

DLL을 작상히랴먄 CRT Library의 시작 코드 지원이 필요하다. 예를 들어 C++ 인스턴스를 전역변수로 가진 DLL을 개발한다면 해당 클래스 생성자가 DllMain보다 먼저 호출되어야 한다. 이러한 작업은 CRT의 DLL 시작 코드에서 수행된다. 

링커는 DLL 링크 시 파일 이미지에 진입점 함수 주소를 포함시키는데, /ENTRY를 쓰면 이 주소를 사용자가 지정할 수 있다. 기본은 CRT에 포함된 _DllMainCRTStartup을 진입점 함수로 지정하며, 이는 DLL 버전의 CRT를 사용할 때도 링크 시에 항상 정적으로 파일 이미지 내에 링크된다. 

시스템은 DllMain 대신 _DllMainCRTStartup를 호출하고 이는 __DllMainCRTStartup로 통지 내용을 전달한다. DLL_PROCESS_ATTACH 통지가 전달된 경우 보안 작업을 수행한 후 CRT를 초기화 하고 전역/정적 C++ 객체를 생성한다. 이것이 완료된 이후 DllMain을 호출하게 된다. 

DLL_PROCESS_DETACH를 전달하는 경우에도 __DllMainCRTStartup를 호출하는데, 이때 __DllMainCRTStartup은 사용자 정의 DllMain을 먼저 호출한다. 이것이 반환된 이후 DLL내 전역/정적 C++ 오브젝트 파괴자를 호출한다. DLL_THREAD를 통지 받는 경우에는 수행하지 않는다. 

```c++
BOOL WINAPI DllMain(HTINSTANCE hInstDll, DWORD fdwReason, PVOID fImpLoad){
    if(fdwReason == DLL_PROCESS_ATTACH)
        DisableThreadLibraryCalls(hInstDll);
    return(TRUE);
}
```
만일 DllMain을 구현하지 않은 경우 위와 같이 구현된 CRT 자체 DllMain을 사용한다. DisableThreadLibraryCalls는 스레드 생성 파괴 시 성능 개선을 위해 호출된다. 

<br/>

# **3. 지연 로드 DLL**

## **1) 필요성과 제한 사항**

지연 로드는 심벌 참조 전까지 로드를 하지 않는 암시적 링크이다. 프로세스 초기화 시간을 줄이고 싶을 때, 버전 별로 다른 함수를 호출할 때 사용할 수 있다. 이전 윈도우에서 지원하지 않는 함수를 호출하면 로더가 에러를 보고하는데, 이때 지연 로드를 이용하면 초기화 시 GetVersionEx로 버전을 확인한 뒤 각 버전에 맞는 함수만 로드하도록 할 수 있다. 
 
지연 로드는 상당히 유용하지만 동시에 몇가지 제한사항을 갖는다. 우선 지연 로드 DLL은 필드를 export할 수 없다. 또, 진입점 함수 내에 지연 로드될 함수를 호출하면 프로세스가 손상된다. 마지막으로 LoadLibrary와 GetProcAddress를 호출해야 하기 때문에 Kernel32.dll 모듈은 지연 로드할 수 없다. 

지연 로드 DLL은 함수들이 시스템 판단 하에 주소 공간 어디에든 바인딩 될 수 있다. 물론 바인딩 가능한 delay-load import section이 실행 파일에 포함되어야 하므로 파일 크기는 조금 더 커진다. 이때 /Delay:nobind로 크기를 줄일 수도 있지만, 동적 바인딩을 사용하는 것이 더 효율적이므로 권장되지 않는다. 

## **2) 지연 로드 DLL 작성 방법**

```
/Lib:DelayImp.lib
/DelayLoad:MyDll.lib
```

우선 일반적인 방식으로 DLL을 작성한 뒤, 일부 링커 스위치를 수정하여 실행 파일을 다시 링크한다. 이때 위의 링커 스위치들은 반드시 추가되어야 한다. 이때 /DELAYLOAD, /DELAY는 #pragma comment 형식으로 설정할 수 없고 프로젝트 속성-구성 속성-링커-입력-Delay Loaded DLLs에서 설정해야 하며 옵션은 구성 속성-링커-고급에서 설정할 수 있다. 

/Lib를 사용하면 링커는 __delayLoadHelper2 함수를 실행 파일에 포함시킨다. /DelayLoad는 import 섹션으로부터 MyDll.dll를 제거하여 이를 암시적으로 로드하지 못하게 하며, 대신 실행 파일 내에 delay-load import section을 포함시켜 지연 로드 DLL 정보를 추가한다. 그리고 지연 로드 함수 호출 코드가 __delayLoadHelper2를 호출하도록 변경한다. 

이후 지연 로드 함수를 호출하면 __delayLoadHelper2가 호출되는데, 이는 delay-load import section를 참조하여 LoadLibrary, GetProcAddress를 호출한다. 이렇게 주소를 한번 획득하면, 이후 동일 함수를 호출할 때 그 함수를 직접 호출할 수 있도록 호출부를 수정한다. 이때 해당 함수 혹은 DLL이 없으면 예외를 발생시키고 DelayLoadInfo에 예외 정보를 담는다. 

## **3) 지연 로드 DLL과 Hook**

사용자는 __delayLoadHelper2 호출 시 수행할 훅 함수를 구성하여 함수 진행 상황, 에러 상황을 받을 수 있다. DelayLoadApp.cpp의 DilHook은 이때 필요한 훅 함수 구조를 정의하고 있으며, 이는 __delayLoadHelper2 동작 방식에 영향을 미치지 않는다. 동작 방식을 변경하려면 DilHook에서 작업을 수행하고 그 주소를 전달하면 된다. 

DelayImp.lib 정적 링크 라이브러리 내부에는 pfnDilHook 타입의 전역 변수__pfnDilNofityHook2, __pfnDilFailureHook2가 정의되어 있다. 이들은 DilHook과 같은 원형의 함수 포인터형 데이터 타입으로, 기본 NULL로 초기화 된다. 여기에 사용자가 구현한 Hook 주소를 전달하면 __delayLoadHelper2가 이 콜백 함수들을 이용해 작업을 수행하게 된다.  

## **4) 지연 로드 DLL의 언로드**

지연 로드 DLL을 언로드하려면 첫째로 실행 /Delay:unload 스위치를 지정해야 하며 둘째로 언로드 시 __FUnloadDelayLoadedDLL2를 호출해야 한다. 스위치를 쓰지 않으면 해당 함수는 FALSE를 반환한다. 저 스위치는 파일에 unload section을 추가하는 역할을, 함수는 section에서 해당 DLL을 찾아 주소값을 모두 삭제한 후 FreeLibrary를 호출하는 역할을 한다.

이때 FreeLibrary를 직접 호출하면 함수 주소가 삭제되지 않으므로 이후 DLL 함수들을 호출할 때 접근 위반이 발생한다. 또, __FUnloadDelayLoadedDLL2를 호출 시 경로명을 포함해서는 안된다. 마지막으로 지연 로드 DLL을 언로드 하는 게 아니라면 /Delay:unload를 사용하지 않는 게 실행 파일 크기를 줄일 수 있다.

<br/>

# **4. 그 외**

## **1) 함수 전달자**

함수 전달자는 DLL export section 내 항목을 일컫는 말로, 이 항목은 함수 호출을 다른 DLL에 있는 함수로 전달하는 역할을 한다. 

```
CloseThreadpoolIo (forwarded to NTDLL.TpReleaseIoCompletion)
```
DumpBin 툴로 Kernel32.dll를 조사하면 위와 같은 부분이 나온다. 이때 CloseThreadpoolIo는 Kernel32.dll를 동적으로 링크하며 실제로는 없는 함수이다. 이를 호출하면 시스템은 GetProcAddress를 호출하여 NTDLL.dll의 export section으로 부터 TpReleaseIoCompletion를 찾아 호출한다.

```c++
#pragma comment(linker, "/export:SomeFunc=DllWork.SomeOtherFunc")
```

자체 개발하는 DLL내에서도 pragma 지시자로 함수 전달자를 적용할 수 있다. 이때 pragma 행은 함수 각각에 대해 작성해야 한다. 

## **2) 알려진 DLL**

운영체제가 제공하는 DLL 중 다소 특별하게 취급되는 것들을 Known DLL이라 한다. 이들은 구조적으로는 일반 DLL과 동일하지만, 해당 DLL을 로드할 때 항상 지정된 레지스트리 키 이하의 DllDirectory 값의 데이터로 주어진 디렉토리에서 찾는다. 그 레지스트리 키 값은 아래와 같다. 

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDlls
```

키 이하에는 .dll을 제외하고 DLL의 이름들로만 설정된 값이 있다. LoadLibrary는 이름에 .dll이 포함되는지 확인한 뒤, 포함되어 있으면 일반적인 검색 규칙으로 탐색을, 포함되지 않으면 확장자 제거 후 KnownDlls 레지스트리 키에서 탐색을 하게 된다. 

## **3) DLL 리다이렉션**

과거에 램과 디스크 공간이 부족하던 때에는 가능한 많은 자원을 공유하도록 하기 위해 DLL을 시스템 디렉토리에 둘 것을 장려했다. 그러나 최근에는 되도록 자신의 디렉터리에 두어서 애플리케이션끼리 영향을 미치는 것을 막기를 권고한다. 이것을 쉽게 지키도록 하기 위해 윈도우 2000부터 리다이렉션 기능이 운영체제에 포함되었다. 

AppName.local 파일 혹은 폴더(ChwApp.exe이라면 ChwApp.exe.local으로) 를 만들면 로더가 해당 디렉토리를 먼저 탐색한다. 폴더를 만들 경우 해당 폴더 내에 필요한 DLL을 모두 저장해두면 깔끔하게 관리할 수 있다. LoadLibrary는 리다이렉션 파일이 있는지 먼저 확인한 뒤, 있다면 이 디렉토리로부터 로드하고 없다면 기존 방식대로 동작한다. 

이 기능은 등록이 필요한 COM 오브젝트를 사용할 때 유용하다. 애플리케이션은 COM 오브젝트 DLL 파일을 디렉터리에 둠으로써 동일한 COM 오브젝트를 등록하는 다른 애플리케이션으로부터의 영향을 막을 수 있다.  그러나 시스템 파일처럼 가장한 파일이 윈도우 시스템 디렉토리 대신 현재 디렉토리로부터 로드될 수도 있으므로 안정성에 주의를 기울여야 한다. 

<br/>

## **4) 모듈 시작 위치 변경**

DLL 모듈은 선호하는 시작 위치를 가지며, 이를 바탕으로 전역 변수 주소를 디스크 상 DLL 이미지에 하드코딩한다. 이때 DLL 두개를 로드하면 첫번째 DLL은 선호 위치에, 두번째 DLL은 다른 위치에 로드될 것이다. 그러나 DLL 이미지의 코드는 하드코딩 된 주소를 건드리므로 문제가 생긴다. 

이를 위해 링커가 모듈을 빌드하면 파일에 relocation section을 만드는데, 이는 기계어 명령어가 사용할 주소를 byte offset로 포함한다. 로더가 선호하는 주소에 로드할 수 없는 경우 이 section을 바탕으로 코드를 수정한다. 이를 통해 시작 위치가 바뀌어도 제대로 동작할 수 있다. 

그러나 코드 수정이 초기화 시간을 길게 만들며, 무엇보다도 copy on write 매커니즘에 의해 시스템 페이징 파일을 사용하게 된다. 모듈 코드 페이지를 폐기 후 재로드 하는 대신, 페이징 파일로 스와핑해야 하는 것이다. 이는 저장소 측면에서도 수행 속도 측면에서도 악영향을 미친다. 

/FIXED 스위치를 사용할 경우 relocation section을 제외할 수 있다. 이 경우 선호 주소에 로드할 수 없게 되면 프로그램이 종료된다. 만약 리소스만 포함하고 있어서 재배치가 필요 없는 모듈이라면 /SUBSYSTEM:WINDOWS, 5.0 혹은 /SUBSYSTEM:CONSOLE, 5.0로 링크를 수행하면 정상적으로 수행된다. 

이러한 이유로 하나의 주소 공간에 여러개의 모듈이 로드 되는 경우 각 시작 주소를 다르게 하는 것이 좋다. 프로젝트 속성 - 링커 - 고급 - 기준 주소 필드에서 이를 설정할 수 있으며, 높은 주소에서 낮은 주소 쪽으로 로드해가는 것이 단편화를 줄일 수 있다. 

모든 모듈의 시작 주소를 재설정하고 싶다면 Visual Studio의 Rebase.exe 도구를 권장한다. 시작 위치 변경을 직접 구현하고 싶다면 ImageHlp API 내의 ReBaseImage 함수를 이용하면 된다. 운영체제에 기본으로 포함된 모듈은 이미 시작 주소 조정 작업이 되어있으므로 절대 변경하면 안된다. 

## **5) 모듈 바인딩**

로더가 import section에 import된 가상 주소를 기록할 경우 copy on write 매커니즘에 의해 이 내용은 페이징 파일을 이용하게 된다. 이때 모듈 바인딩 작업을 미리 수행해두면 초기화 시간을 개선하고 적은 저장소를 사용할 수 있다. 모듈 바인딩은 모듈의 import section을 import된 가상 주소로 미리 준비해두는 것을 말한다. 

모듈 바인딩은 Visual Studio의 Bind.exe으로 하기를 권장하며, ImageHlp API의 BindImageEx 함수로 직접 구현할 수도 있다. 이때, DLL이 선호 주소에 로드되어 있고 심벌 위치가 바인딩 수행 이후 변경되지 않아야 한다. 이 조건이 맞지 않을 경우 일반적인 경우처럼 import section을 수정하고 페이징 파일을 이용한다. 

바인딩은 운영체제 버전을 알아야 하기 때문에 애플리케이션 설치 작업의 일부로서 수행되도록 하는 것이 좋다. 그러나 운영체제를 업그레이드 한 이후에 모듈을 다시 바인딩할 수 있는 도구는 아직 MS에서 제공되지 않고 있다. 