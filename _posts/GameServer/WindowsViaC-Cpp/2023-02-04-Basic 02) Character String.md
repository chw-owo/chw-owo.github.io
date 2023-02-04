---
title: Basic 02) Character String
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. ANSI/ 유니코드**

C언어의 char 자료형은 8 bit의 ANSI 문자 표현을 위해 존재하며, wchar_t는 16bit의 유니코드(UTF-16)을 표현하기 위해 존재한다. 모든 문자를 4byte로 인코딩하는 UTF-32 표준안도 있으나, 이는 거의 사용되지 않는다. 최근 컴파일러는 wchar_t를 내장 자료형으로 처리하지만, 옛 컴파일러의 경우 unsigned short로 처리하기도 한다. 되도록 내장 자료형으로 처리하도록 컴파일러 스위치를 사용하는 것이 좋다. WinNT.h 헤더 파일을 보면 아래와 같은 방법으로, 유니코드 설정 시 WCHAR을, ANSI 설정 시 CHAR를 사용하도록 매크로를 이용해 분기하고 있다. 

```c++
#ifdef UNICODE

typedef WCHAR TCHAR, *PTCHAR, PTSTR;
typedef CONST WCHAR * PCTSTR;
#define __TEXT(quote) quote
#define __TEXT(quote) L##quote

#else

typedef CHAR TCHAR, *PTCHAR, PTSTR;
typedef CONST CHAR * PCTSTR;
#define __TEXT(quote) quote

#endif

#define TEXT(quote) __TEXT(quote)

```

개발 시 되도록 유니코드를 사용하길 권장하는데 그 이유는 아래와 같다. 

**-** 어플리케이션을 다른 나라 언어로 지역화 하기 쉽다.

**-** 단일 바이너리 파일로 모든 언어를 지원할 수 있다.

**-** 윈도우는 내부적으로 유니코드를 사용하기 때문에 유니코드를 사용할 떄 코드가 더 빠르게 수행되며 더 작은 메모리를 사용한다.

**-** 몇몇 윈도우 함수는 유니코드만 받아들일 수 있도록 작성되어 있다.

**-** COM, 닷넷 프레임워크의 경우 유니코드를 사용하기에 상호 운용이 쉬워진다. 

**-** 리소스 내 문자열은 모두 유니코드로 유지되므로 리소스를 쉽게 다룰 수 있다. 

<br/>

# **2. 윈도우의 ANSI/ 유니코드 함수**

Window NT 이후의 모든 윈도우 버전은 유니코드를 바탕으로 작성되었다. 하지만 윈도우에서 문자열 인자를 가지는 함수의 경우 일반적으로 _W(Wide의 약자, UTF-16), _A(ANSI의 약자, UTF-8) 두가지 버전을 제공한다. 어떤 버전을 사용할지 역시 내부적으로 매크로를 통해 분기되고 있다. 

이때 버전에 따라서, _A를 호출할 경우 ANSI로 문자열을 만든 뒤 유니코드 문자열로 변경하는 단계를 추가적으로 수행하기도 한다. 또, 최근에는 리소스 컴파일러가 컴파일한 리소스는 항상 유니코드 문자열로 구성되는데 이를 대상으로 ANSI 전용 함수를 호출하면 유니코드 문자열을 ANSI로 변경 후 함수 내용을 실행한다. 이러한 상황은 더 많은 메모리를 소비하며 더 느리게 동작하므로 되도록 처음부터 유니코드를 사용하는 것을 권장한다. 

참고로 WinExec, OpenFile과 같은 몇몇 Window API는 16bit 윈도우 프로그램과의 호환성을 위해 인자로 ANSI 문자열만 지원한다. 따라서 새로 개발하는 프로그램에서는 이 대신 CreateProcess, CreateFile 등을 사용하는 것이 좋다. 이와 반대로 ReadDirectoryChangesW, CreateProcessWithLogonW 등은 유니코드 버전만 제공하기도 한다.

<br/>

# **3. C의 ANSI/ 유니코드 함수**

윈도우 함수와 마찬가지로 C 런타임 라이브러리도 ANSI 함수와 유니코드 함수를 세트로 제공한다. 하지만 C 런타임 라이브러리의 ANSI 함수의 경우 유니코드로 변경하는 과정을 거치지 않기 때문에 윈도우 함수와 달리 성능 저하가 일어나지 않는다. C의 대표적 문자열 함수는 strlen, wcslen이 있다. TChar.h를 참조하면 아래와 같이 분기하고 있음을 알 수 있다. 

```c++
#ifdef _UNICODE
#define _tcslen wcslen
#else
#define _tcslen strlen
#endif
```
이를 사용할 때는 되도록 _tcslen를 사용하는 게 호환성 측면에서 좋다. 이때, 윈도우 함수와 달리 _UNICODE를 정의하고 있는 것을 볼 수 있다. C언어는 모든 구분자에 항상 언더스코어를 붙이나, 이것이 C++의 표준안은 아니기 때문에 윈도우에서는 언더스코어를 붙이지 않았다. 따라서 코드 내부에서 유니코드를 정의할 때는 UNICODE, _UNICODE를 함께 정의하거나 둘 다 정의하지 않아야 한다. 

<br/>

# **4. C의 안전 문자열 함수**

문자열 버퍼가 담으려는 문자열보다 작다면 메모리 관련 문제가 발생할 수 있다. 특히 strcpy, wcscpy의 경우 버퍼의 최대 크기를 인자로 받지 않기 때문에 헷갈릴 위험이 크다. 이러한 위험을 줄이기 위해 최근에는 _s를 붙인 안전 문자열을 사용하길 권장한다. 이들은 사용할 버퍼와 함께 버퍼의 크기를 인자로 전달해야 한다. 버퍼 크기 (문자 개수)는 _countof 매크로를 사용하면 쉽게 계산할 수 있다. 

_s 함수의 경우 내부적으로 인자의 유효성을 먼저 검증한 후 유효하지 않으면 예외를 던진다. 이에는 인자 값이 NULL인지, 정수 값 및 열거형 값이 유효한지, 버퍼의 크기는 충분한 지 테스트 한다. 만약 검증이 실패할 경우 TLS 변수 errno에 에러코드를 설정하고, errno_t 값을 반환한다. 디버그 모드에서는 왜 검증에 실패했는지 메세지를 띄워주고, 릴리즈 모드에서는 메세지 없이 프로그램이 종료된다. 

C 런타임 라이브러리에서는 안전 문자열 함수 외에도 부족한 문자열을 어떤 값으로 채울지, 혹은 문자열 잘림을 어떻게 처리할 지 세부적으로 지정할 수 있는 함수를 지원한다. 해당 함수의 예시는 아래와 같다. 

```c++
HRESULT StringCchCat(PTSTR pszDest, size_t cchDest, PCTSTR pszSrc);
HRESULT StringCchCatEx(PTSTR pszDest, size_t cchDest, PCTSTR pszSrc, PTSTR *ppszDestEnd, size_t *pcchRemaining, DWORD dwFlags);

HRESULT StringCchCopy(PTSTR pszDest, size_t cchDest, PCTSTR pszSrc);
HRESULT StringCchCopyx(PTSTR pszDest, size_t cchDest, PCTSTR pszSrc, PTSTR *ppszDestEnd, size_t *pcchRemaining, DWORD dwFlags);

HRESULT StringCchPrintf(PTSTR pszDest, size_t cchDest, PCTSTR pszFormat, ...);
HRESULT StringCchPrintfEx(PTSTR pszDest, size_t cchDest, PTSTR *ppszDestEnd, size_t *pcchRemaining, DWORD dwFlags, PCTSTR pszFormat, ...);
```
이때 Cch는 Count of characters를 의미하며 _countof 매크로를 이용해 적절한 값을 전달할 수 있다. 간혹 Cch 대신 Cb가 들어가는 함수들의 경우 Count of bytes를 요구하는 것이므로 sizeof 연산자를 이용하여 적절한 값을 전달하면 된다. 위처럼 HRESULT 반환형을 가진 함수는 다음 값을 반환한다. 

|반환 값| 의미 |
|------|------|
|S_OK|성공|
|STRSAFE_E_INVALID_PARAMETER|인자값으로 Null이 전달됨|
|STRSAFE_E_INSUFFICIENT_BUFFER|복사 대상 버퍼가 원본 문자열을 담기에 충분하지 않음|

만약 대상 버퍼의 크기가 부족할 경우 설정한 대로 문자 잘림이 정상적으로 수행되며, STRSAFE_E_INSUFFICIENT_BUFFER 값을 반환한다. Ex는 Extend, 즉 확장 버전의 함수임을 의미하며 추가적으로 3개의 매개변수를 취하는데 각 인자들은 아래와 같은 의미를 갖는다. 

|매개변수|설명|
|------------|---|
|size_t *pcchRemaining|복사 대상 버퍼 내 남아있는 문자의 개수를 가져오며, 이때 종결 문자는 포함하지 않는다. 만약 이 인자에 Null을 전달하면 개수가 반환되지 않는다. |
|PTSTR *ppszDestEnd|ppszDestEnd가 Null이 아닐 경우 복사 대상 버퍼 내 문자열 종결 문자 '\\0'을 가리킨다. |
|DWORD dwFlags| 아래 나열한 값을 \| 를 통해 전달한다. |

dwFlags에 들어갈 수 있는 값은 아래와 같다. 

|값|설명|
|--|---|
|STRSAFE_FILL_BEHIND_NULL|함수가 성공하면 dwFlags의 하위 바이트를 통해 전달한 값을 이용하여 복사 대상 버퍼의 '\\0'이후 나머지 공간을 채운다.|
|STRSAFE_IGNORE_NULLS|Null 값을 가진 문자열 포인터를 비어있는 문자열을 가리키는 포인터처럼 다룬다|
|STRSAFE_FILL_ON_FAILURE|함수가 실패하면 dwFlags의 하위 바이트를 통해 전달한 값을 이용하여 비어있는 문자열을 표시하기 위한 '\\0'을 제외한 모든 공간을 채운다. 만약 STRSAFE_E_INSUFFICIENT_BUFFER가 발생한다면 복사 대상 버퍼에 이미 복사된 잘린 문자열들도 주어진 값으로 대체된다. |
|STRSAFE_NULL_ON_FAILURE|만일 함수가 실패하면 비어있는 문자열을 나타내기 위해 복사 대상 버퍼 최초 문자를 '\\0'으로 설정한다. 만약 STRSAFE_E_INSUFFICIENT_BUFFER가 발생한다면 복사 대상 버퍼에 이미 복사된 잘린 문자열들이 있는 경우에도 덮어씌워진다. |
|STRSAFE_NO_TRUNCATION|STRSAFE_NULL_ON_FAILURE와 동일하게 동작한다.|

<br/>

# **5. 윈도우의 안전 문자열 함수**

윈도우 또한 lstrcat, lstrcpy와 같이 문자열을 다루는 다양한 함수를 제공하나, 이들은 여전히 버퍼 오버런 문제에 노출되어 있으므로 사용하지 말아야 한다. 또, StrFormatKBSize, StrFormatByteSize와 같이 운영체제와 연관되어 숫자 값을 포매팅 하는 함수 역시 동일한 문제를 야기할 수 있으므로 사용하지 않는 것이 좋다. 윈도우 함수를 이용하여 문자열 간의 비교, 정렬을 할 때는 CompareString(Ex), CompareStringOrdinal을 사용하는 것이 권장된다. 

CompareString의 함수 원형은 아래와 같다. 

```c++
int CompareString (

    LCID locale,
    DWORD dwCmdFlags,
    PCTSTR pString1,
    int cch1,
    PCTSTR pString2,
    int cch2
);
```

우선 첫번째 인자 locale을 통해 사용하려는 언어의 지역 ID인 LCID를 전달한다. 이 경우 각 언어별 고유의 의미를 확인해가면서 비교할 수 있다는 장점을 갖는 반면, 순차적 값 비교에 비해 상대적으로 느리게 수행된다. 윈도우의 GetThreadLocale 함수를 이용하면 함수를 호출한 스레드의 LCID 값을 얻을 수 있다. 그 후 dwCmdFlags를 통해 대소문자 구분, 히라가나-가타카나 구분, 특수 기호 사용 등에 대해 설정하고, 마지막으로 복사하려는 문자열과 해당 문자열의 크기 등에 대해 지정한다. 

프로그램 내에서 사용하는 일반적인 문자열을 비교할 때는 CompareStringOrdinal을 많이 사용한다. 이를 통해 경로명, 레지스트리 Key-Value, XML 요소와 특성 등을 비교할 수 있다. 

```c++
int CompareStringOrdinal (

    PCTSTR pString1,
    int cchCount1,
    PCTSTR pString2,
    int cchCount2,
    BOOL bIgnoreCase
);
```
이 함수는 지역 설정을 고려하지 않고 단순 비교를 수행하기 때문에 상대적으로 빠르다. 단, 유니코드 문자열만을 인자로 취한다는 점에 주의해야 한다. 

위의 함수들은 호출 실패할 경우 0을, pString1이 pString2 보다 작을 경우 1(CSTR_LESS_THAN)을, 동일할 경우 2(CSTR_EQUAL), 클 경우 3(CSTR_GREATER_THAN)을 반환한다. 

<br/>

# **6. 문자열 작업 권고 사항**

**-** 문자열을 char 타입 혹은 byte의 배열로 생각하지 말고 문자의 배열로 인식하라.

**-** 문자 혹은 문자열을 나타낼 때는 TCHAR, PTSTR과 같은 중립 자료형을 사용하라.

**-** byte, byte pointer, data buffer 등을 표현할 때는 BYTE, PBYTE와 같이 명시적인 자료형을 사용하라.

**-** 문자 혹은 문자열 상수 값을 표현할 때는 TEXT 혹은 _T 매크로를 사용하되, 가독성을 위해 두 매크로를 혼용해서는 안된다. 

**-** 문자열에 대한 산술적 계산에 유의하라. 보통의 함수는 버퍼 크기를 전달할 때 바이트 대신 문자 단위로 값을 전달하므로, sizeof(szBuffer) 대신 _countof(szBuffer)를 사용해야 한다. 

**-** 문자열 저장을 위한 메모리 블록을 할당할 때, 메모리 할당은 바이트 단위로 수행해야 한다. 즉, malloc(nCharacters) 대신 malloc(nCharacters * sizeof(TCHAR)) 이 되어야 한다. 

**-** prinft 류의 함수 사용을 자제하라.

**-** %s, %S 등을 ANSI, 유니코드 문자열 간의 변환을 통해 사용하는 것은 좋지 않다. 이 대신 MultiByteOToWideChar, WideCharToMultiByte를 사용하라.

**-** UNICODE와 _UNICODE는 항상 동시에 정의, 해제하라.

**-**  _s 로 끝나거나 StringCch로 시작하는 안전 문자열 함수를 사용하고 함수 사용 이후 문자열 잘림에 대비하라.

**-** 버퍼와 버퍼의 크기를 동시에 인자로 받지 않는 함수는 만들지도 사용하지도 않는 것이 좋다. 

**-**  컴파일러가 자동으로 버퍼 오버런을 감지하도록 /GS, /RTCs 컴파일러 플래그를 활용하라.

**-** Kernel32가 제공하는 lstrcat, lstrcpy 등 문자열 관련 함수를 사용하지 마라.

**-** 사용자의 유저 인터페이스를 구성하는 문자열의 경우 사용자의 언어 설정을 고려하는 CompareString(Ex)를 사용하는 것이 좋다. 

<br/>


