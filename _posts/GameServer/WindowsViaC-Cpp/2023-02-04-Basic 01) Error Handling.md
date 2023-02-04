---
title: Basic 01) Error Handling
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Window Function Return**

윈도우 함수의 반환 자료형과 반환 자료형 별 실패시 반환하는 값은 아래 표와 같다. 더 자세한 것은 플랫폼 SDK 문서를 참조하는 것이 좋다. 

|자료형|실패 시 반환 값|
|-----|--------------|
|VOID| 절대 실패하지 않는 함수|
|BOOL| 실패 시 0(FALSE)을, 성공 시 0이 아닌 값을 반환. 반환값을 TRUE와 비교하면 적합하지 않은 결과가 나올 수 있으므로 FALSE와 비교하는 것이 좋다. |
|HANDLE| 실패 시 NULL 혹은 -1(INVALID_HANDLE_VALUE)을, 성공시 유효한 핸들을 반환. |
|PVOID| 실패 시 NULL을, 성공시 PVOID가 데이터를 저장한 메모리 주소를 반환|
|LONG, DWORD| 실패 시 0 혹은 -1을 반환. |

윈도우 함수가 실패하면 TLS(Therad Local Storage)에 에러코드를 저장하기 때문에 각 스레드 별 에러코드를 유지할 수 있다. 함수 호출이 성공하면 ERROR_SUCCESS를 에러코드로 기록한다. 

```c++
DWORD GetLastError()
```

위 함수는 가장 최근 호출된 함수의 에러 코드를 TLS로부터 가져온다. 

# **2. Define Custom Error Code**

```c++
VOID SetLastError(DWORD dwErrCode);
```

위 함수를 통해 에러코드를 설정할 수 있다. 함수의 인자로 전달하는 값은 아래와 같은 틀에 맞게 정하면 된다. 

|비트|31-30|29|28|27-16|15-0|
|---|------|--|--|-----|---|
|내용|심각도|지정 주체|예약 여부|식별 코드|예외 코드|
|의미|00 = 성공 / 01 = 정보 / 10 = 주의 / 11 = 에러 | 0 = 마이크로소프트/ 1 = 사용자 정의 | 항상 0 |256까지 마이크로소프트에 의해 예약됨|예외를 구분하는 코드|

# **출처**

출처: 제프리 리처의 Windows via C/C++ (한빛미디어)