---
title: Memory 02) Allocator
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Allocator**

최근에는 컴퓨터 성능이 좋아서 커널이 메모리를 할당해주는 대로 사용하는 경우가 많지만, 메모리를 효율적으로 사용하기 위해 유저가 메모리 할당을 간접적으로 통제할 수도 있다. 이때, new-delete의 기능을 오버로딩하여 래핑한 Allocator를 많이 사용되게 되는데, 간단하게 구현한 Allocator 예제는 아래와 같다. 

**Allocator.h**
```c++
#pragma once
class BaseAllocator
{
public:
    static void*    Alloc(int32 size);
    static void     Release(void* ptr);
}
```

**Allocator.cpp**
```c++
void* BaseAllocator::Alloc(int32 size)
{
    return ::malloc(size);
}

void BaseAllocator::Release(void* ptr)
{
    ::free(ptr);
}
```

**Memory.h**
```c++
#pragma once
#include "Allocator.h"

template<typename Type, typename ... Args>
Type* xnew(Args&& ... args)
{   
    Type* memory = static_cast<Type*>(BaseAllocator::Alloc(sizeof(Type)));
    new(memory)Type(forward<Args>(args)...);
    return memory;
}

template<typename Type>
void xdelete(Type* obj)
{   
    obj-> ~Type();
    BaseAllocator::Release(obj);
}
```

**Main.cpp**
```c++
int main()
{
    Player* player = xnew<Knight>(100);
    xdelete(Knight);
}
```

<br/> 

# **2. Stomp Allocator**

Stomp Allocator는 메모리 관련 버그를 잡을 때 유용하게 쓰일 수 있다. 이는 일반적으로 VirtualAlloc - VirtualFree을 직접 호출함으로써 구현한다. VirtualFree를 하게 되면 OS에서 해당 메모리 접근 불가능한 영역으로 만들기 때문에 Free를 마친 영역에 접근했을 때 바로 CRASH가 나게 된다. 이에 대한 자세한 설명은 [이 포스트를][1] 참조. 이렇게 사용할 경우 성능 저하는 생기겠지만 보다 예외가 생길 수 있는 상황을 미리 감지할 수 있기 때문에 안전하게 작업할 수 있다. StompAllocator의 간단한 구현 예제는 아래와 같다. 


**Allocator.h**
```c++
#pragma once
class StompAllocator
{
    enum { PAGE_SIZE = 0x1000 };

public:
    static void*    Alloc(int32 size);
    static void     Release(void* ptr);
}
```

**Allocator.cpp**
```c++
void* StompAllocator::Alloc(int32 size)
{
    const int64 pageCount = (size + PAGE_SIZE - 1)/PAGE_SIZE;
    const int64 dataOffset = pageCount * PAGE_SIZE - size;

    void* baseAddress = ::VirtualAlloc(NULL, pageCount * PAGE_SIZE, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);
    return static_cast<void*>(static_cast<int8*>(baseAddress) + dataOffset);
    
}

void StompAllocator::Release(void* ptr)
{
    const int64 address = reinterpret_cast<int64>(ptr);
    const int64 baseAddress = address - (address % PAGE_SIZE);
    ::VirtualFree(reinterpret_cast<void*>(baseAddress), 0, MEM_RELEASE);
}
```

그 외의 파일에서도 BaseAllocator을 StompAllocator로 변환해주면 new, delete를 할 때마다 VirtualAlloc - VirtualFree 를 하며 메모리 침범 상황을 미리 감지할 수 있다. 그러나 VirtualAlloc - VirtualFree를 사용하는 것만으로는 클래스 간의 typecast로 인한 오버플로우는 감지할 수 없다. 만약 이를 감지하고 싶다면 위 예제처럼 자료형 크기에 맞게 할당되는 메모리의 끝부분에 할당할 경우 그 크기를 초과하면 에러를 띄워줄 것이다. 물론 이 경우 언더플로우를 감지할 수 없다는 한계점이 있지만 일반적으로 typecast로 인한 언더플로우 문제는 거의 발생하지 않기 때문에 첫 주소에 맞추는 것보다는 안전할 것이다. 언리얼에서는 이러한 StmpAllocator을 지원한다. 

<br/> 

# **3. Allocator for STL Container**

위에서 만든 Allocator을 STL Container에서도 사용하기 위해서는 별도의 처리가 필요하다. 이를 구현한 예제는 아래와 같다. 

**CoreMacro.h**
```c++
#define xalloc(size)    StompAllocator::Alloc(size);
#define xrelease(ptr)   StompAllocator::Release(ptr);
```

**Allocator.h**
```c++
#pragma once
class StompAllocator
{
    enum { PAGE_SIZE = 0x1000 };

public:
    static void*    Alloc(int32 size);
    static void     Release(void* ptr);
}

template<typename T>
class StlAllocator
{
public:
    StlAllocator(){}
    template<typename Other>
    StlAllocator(const StlAllocator<Ohter>&){}

    T* allocate(size_t count)
    {
        const int32 size = static_cast<int32>(count * sizeof(T));
        return static_cast<T*>(xalloc(size));
    }

    void deallocate(T* ptr, size_t count)
    {
        xrelease(ptr);
    }

public:
    using value_type = T;
}
```


**Container.h**
```c++
#pragma once
//#include ...

template<typename Type>
using Vector = vector<Type, StlAllocator<Type>>;

template<typename Key, typename type, typename Pred = less<Key>>
using Map = map<Key, Type, Pred, StlAllocator<pair<const Key, Type>>>;

template<typename Type>
using Deque = deque<Type, StlAllocator<Type>>

template<typename Type, typename Container = Deque<Type>>
using Queue = queue<Type, Container>
```
**Main.cpp**
```c++
int main()
{
    Vector<int32> v;
}
```
<br/> 

STL Container의 기본 구조를 살펴보면 allocator 를 설정해주는 부분이 있다. 이를 바탕으로 어떤 요소가 있어야 Container에 넣을 수 있는지 (ex value_type) 확인한 뒤, 그에 맞게 구현해주면 위와 같은 예제가 나온다. 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss

[1]: https://chw-owo.github.io/procademyreview/Procademy,-Q1,-09)-Cache-Memory-MESI,-Align/