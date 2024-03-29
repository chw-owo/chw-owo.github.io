---
title: Memory 01) Windows Memory
categories: WindowsViaC-Cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 <제프리 리처의 Windows via C/C++ (한빛미디어)> 내용을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 프로세스의 가상 주소 공간**

## **1) 가상 주소 공간의 분할**

프로세스는 자신만의 가상 주소 공간을 가지며 프로세스 내에서 스레드가 수행될 때 그 스레드는 프로세스가 소유한 메모리에 대해서만 접근 가능하다. 접근 예외를 유발하지 않고 데이터에 접근하기 위해서는 해당 주소 공간에 물리적 저장소가 할당되거나 매핑되어 있어야 한다. 

이때의 주소 공간은 물리적 저장소가 아니라 단순하게 메모리 위치의 지정 범위를 나타낸 값을 의미한다. 가상 주소 공간은 분할되어 있으며 각각의 분할 공간을 파티션이라고 한다. 분할 방식은 운영체제 구현 방식에 따라 서로 다르지만, 분할된 파티션은 크게 NULL 포인터, 유저 모드, 64KB 접근 금지, 커널 모드 파티션으로 구성된다.

NULL 포인터 파티션은 일반적으로 프로그래머가 NULL 포인터 할당 연산을 수행할 경우를 대비해 준비된 영역이며, 64KB 접근 금지 파티션은 MS에서 커널 모드 파티션을 보호하기 위해 지정해둔 완충 지대 격의 영역이다. 이 영역들의 주소에 접근할 경우 접근 위반이 발생하며 프로세스가 종료된다. 

유저모드 파티션은 프로세스에서 직접 활용할 수 있는 영역으로 타 프로세스가 사용하는 유저 모드 파티션의 주소에는 접근할 수 없다. x86 버전의 윈도우에서는 기본적으로 유저 모드, 커널 모드 공간을 각각 2GB 사용할 수 있는데 BCD를 변경하여 이 영역을 최대 3GB로 확장할 수 있다. 

커널 모드 파티션에는 스레드 스케줄링, 메모리 관리, 파일 시스템 지원, 네트워크 지원 등의 코드 및 디바이스 드라이버들과 같이 운영체제를 구성하는 코드들이 위치한다. 이 영역의 내용은 모든 프로세스에 의해 공유되지만 사용자가 직접 접근할 수는 없는데, 만일 애플리케이션에서 이 파티션에 대해 접근을 시도하면 접근 위반이 발생하며 프로세스가 종료된다. 

32bit 애플리케이션을 64bit 환경에 포팅할 경우, 많은 코드들이 포인터를 32bit 값으로 가정하기 때문에 각종 메모리 접근 에러가 발생할 수 있다. 따라서 0x00000000`7FFFFFFF를 초과하는 메모리를 할당하지 않게끔 하기 위해 주소 공간 샌드박스 내에서 애플리케이션을 수행하거나, /LARGEADDRESSAWARE 링커 스위치를 이용해야 한다. 

## **2) 주소 공간 영역 예약**

프로세스가 생성되고 주소 공간이 주어졌을 때 프로세스 환경 블록, 스레드 환경 블록 등 특별한 영역를 제외하고 대부분의 주소 공간은 할당되지 않은 상태이다. 이 공간을 사용하려면 VirtualAlloc으로 영역을 할당해야 하는데 이를 예약한다고 표현 한다. 더 이상 사용하지 않을 영역은 VirtualFree 함수로 해제해야 한다.

영역을 예약할 때는 항상 시작 주소가 할당 단위 경계 상에 위치해야 하며 영역의 크기는 페이지 크기의 배수로 설정해야 한다. 2023년 기준 일반적으로 64KB의 할당 단위와 4KB의 페이지 크기를 사용하고 있으나 IA-64의 경우 8KB의 페이지 크기를 사용하기도 한다. 

<br/>

# **2. 물리적 저장소**

## **1) 물리적 저장소 커밋**

최근의 운영체제는 디스크 공간을 메모리처럼 활용하는데, 디스크 상에 있는 이러한 파일을 페이징 파일이라고 하며 모든 프로세스가 가상의 메모리처럼 사용할 수 있다. 운영체제는 CPU와 협력하여 페이징 파일에 필요한 데이터가 있을 때 기존 램 내용을 페이징 파일로 내보내고 페이징 파일의 내용을 램으로 읽어들인다. 이를 잘 사용하면 더 많은 메모리 공간이 존재하는 것처럼 사용할 수 있다. 

스레드가 데이터에 접근할 때 데이터가 램에 존재한다면 CPU는 바로 가상 주소를 물리적 주소로 변경하여 접근한다. 만약 램 대신 페이징 파일에 있는 경우, page fault가 발생하면서 램에서 해당 데이터를 페이징 파일로부터 프리페이지로 가져오게 된다. 프리 페이지가 없을 경우 페이지 중 하나를 프리 상태로 변경하여 이를 수행한다. 그 후 CPU가 page fault를 유발했던 명령을 다시 수행하게 되고, 동일하게 가상 주소를 물리적 주소로 변경하여 데이터에 접근하게 된다. 

페이징 파일과 램 사이의 데이터 이동이 많아지면 하드 디스크 트래쉬가 생기며 시스템의 수행 속도가 느려진다. 이러한 스와핑에 시간이 많이 소요되어서 운영체제가 시스템의 수행 속도가 떨어지는 것을 트레슁이라 하는데, 추가적인 램을 설치했을 때 성능이 개선되는 것도 이러한 트래슁이 감소하기 때문이다. 

## **2) 물리적 저장소와 페이지 파일**

애플리케이션을 수행하면 시스템은 실행 파일 (대체로 exe)을 열어서 코드와 데이터의 크기를 얻어낸다. 이후 프로세스 주소 공간에 그만큼의 영역을 예약하고, 커밋된 물리적 저장소를 exe 파일이라 설정한다. 페이징 파일에 공간을 할당하는 대신 프로세스 주소 공간의 영역을 이렇게 활용함으로써 페이징 파일의 크기를 크게 증가시키지 않으면서도 애플리케이션을 빠르게 로딩할 수 있다.

하드디스크 상에 존재하는 exe, dll 등의 프로그램 파일이 주소 공간의 특정 영역에 대한 물리적 저장소로 사용되는 경우 이러한 파일을 메모리 맵 파일이라고 부른다. 이에 대해 로드를 시도하면 시스템은 자동으로 프로세스 주소 공간에 영역을 예약하고 파일을 매핑한다. 이 외에 데이터 파일을 매핑하는 방법 또한 제공하고 있다. 

윈도우는 여러 개의 페이징 파일을 사용할 수 있는데, 물리적으로 다른 드라이브에 대해서 동시에 접근할 있으므로 서로 다른 하드 디스크 상에 다수의 페이징 파일을 구성할 경우 시스템을 더 빨리 구동할 수 있다. 제어판 - 사양 정보 및 도구 - 고급 도구 - 성능 조정 - 고급- 가상 메모리 영역에서 페이징 파일을 추가, 삭제할 수 있다. 

<br/>

# **3. 보호 특성**

## **1) 보호 특성**

물리적 저장소가 할당된 페이지들은 각각 보호특성을 가질 수 있는데, 이는 멜웨어의 공격으로부터 시스템을 보호하기 위한 기능이다. 보호 특성의 예시로는 PAGE_NOACCESS, PACE_READONLY, PAGE_EXECUTE 등이 있으며 이들은 대부분 램과 페이징 파일 모두에 대해 특성 값을 유지한다. (PAGE_WRITECOPY, PAGE_EXECUTE_WRITECOPY 제외) 

## **2) 카피 온 라이트**

윈도우는 프로세스 사이의 저장소 공유 기능을 제공하는데, 이는 기본적으로 읽기 전용, 실행 전용에 대해서만 가능하다. 만약 공유 된 블록에 대해 쓰기를 하면 카피 온 라이트 기능이 설정된다. exe, dll이 주소 공간에 매핑되면 시스템은 얼만큼의 페이지가 쓰기 가능한 상태인지 확인하여 그만큼의 저장소를 페이징 파일에 할당하는데, 이렇게 할당된 페이지는 쓰기 가능 페이지의 내용이 변경되는 경우에 한해서만 사용된다. 

스레드가 공유 블록에 쓰기를 시도하면 시스템은 우선 프리페이지를 찾은 뒤, 여기에 쓰기 작업을 수행할 페이지 내용을 복사한다. 프리페이지는 PAGE_READWRITE 혹은 PAGE_EXECUTE_READWRITE가 설정될 수 있으며 원래 페이지의 보호 특성과 데이터는 변경되지 않는다. 그 후 프로세스 페이지 테이블을 갱신하여 복사된 페이지에 접근한다. 이러한 단계가 수행되고 나면 프로세스는 물리적 저장소에 자신만의 고유한 데이터 페이지를 가지게 된다. 

이때 PAGE_WRITECOPY, PAGE_EXECUTE_WRITECOPY가 사용된다. 이 보호 특성들이 설정된 페이지에 쓰기를 시도할 경우, 시스템은 페이징 파일의 지원을 받아서 이 페이지의 복사본을 프로세스 고유의 페이지에 새로 구성해준다. 이 과정에서 PAGE_WRITECOPY는 접근 위반 에러를 일으키며, PAGE_EXECUTE_WRITECOPY는 에러를 일으키지 않는다. 

만약 VirtaulAlloc 함수로 예약하거나 커밋한 공간에 대해 PAGE_WRITECOPY, PAGE_EXECUTE_WRITECOPY를 사용하면 ERROR_INVALID_PARAMETER를 반환하면서 함수 호출에 실패하게 된다. 이 두가지 보호 특성은 운영체제가 exe, dll 파일 이미지를 매핑할 때만 사용하도록 지정해두었기 때문이다. 

## **3) 특수 접근 보호 특성 플래그**

PAGE_NOCACHE, PAGE_WRITECOMBINE, PAGE_GUARD라는 보호 특성 플래그들은 OR 연산을 통해 PAGE_NOACCESS를 제외한 나머지 보호특성들과 함께 사용될 수 있다. 

PAGE_NOCACHE는 커밋된 페이지에 대해 캐싱을 수행하지 않도록 한다. 이는 일반적인 용도로는 거의 사용되지 않으며 메모리 버퍼를 관리하는 하드웨어 디바이스 드라이버를 개발을 위해 존재한다.

PAGE_WRITECOMBINE는 단일 장치에 대한 여러번의 쓰기 작업을 하나로 결합할 수 있도록 해주는데 이 역시 디바이스 드라이버 개발에 주로 사용된다. 

PAGE_GUARD는 페이지에 내용이 쓰였을 때 애플리케이션이 예외를 통해 그것을 인지할 수 있도록 하기 위해 사용된다. 운영체제는 스레드의 스택을 생성할 때 이 플래그를 자체적으로 사용하고 있다. 

<br/>

# **4. 그 외**

## **1) 메모리 영역 타입**

가상 메모리 지도를 살펴보면 프로세스의 주소 공간을 아래와 같은 메모리 영역 타입으로 분류해서 알려준다. 

**Free**: 이 영역의 가상 주소는 어떠한 저장소로도 매핑되지 않은 상태로, 예약을 수행할 수 있다. 

**Private**: 이 영역의 가상 주소는 시스템의 페이징 파일에 매핑되어 있다. 

**Image**: 이 영역의 가상 주소는 이전에 exe, dll과 같은 메모리 맵 이미지 파일에 매핑되었으나, 현재는 아닐 수도 있다. 예를 들어 전역변수에 대해 쓰기가 시도된 경우 카피 온 라이트에 의해 이전 이미지 파일로부터 페이징 파일로 매핑 정보가 변경된다.  

**Mapped**: 이 영역의 가상 주소는 이전에 메모리 맵 데이터 파일에 매핑되었으나, 현재는 아닐 수도 있다. 예를 들어 이 영역에 대해 쓰기가 시도된 경우 카피 온 라이트에 의해 데이터 파일로부터 페이징 파일로 매핑 정보가 변경된다.  

**Reserve**: 이 영역의 가상 주소는 아직 어떠한 저장소도 매핑되지 않았지만 예약은 되어있는 상태이다. 

## **2) 데이터 정렬의 중요성**

메모리의 주소 값을 데이터의 크기로 나누었을 때 나머지가 0인 경우 데이터가 정렬되어 있다고 표현하는데, CPU는 이처럼 데이터가  정렬되어 있을 때 더욱 효율적으로 접근할 수 있다. CPU가 메모리 상 정렬되지 않은 데이터를 읽으려면 예외를 유발하거나, 정렬되지 않은 데이터를 모두 읽을 때까지 정렬된 위치들을 여러번 반복해서 읽기 때문이다. WinNT.h에서 제공하는 이러한 상황을 막고 싶다면 UNALIGNED, UNALIGNED64 매크로를 사용할 수 있다. 

<br/>

