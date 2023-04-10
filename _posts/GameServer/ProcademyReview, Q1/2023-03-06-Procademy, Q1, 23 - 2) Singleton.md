---
title: Procademy, Q1, 23 - 2) Singleton
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 싱글톤이란?**

유일하게 하나만 존재해야 하는, 하나 이상 생성되면 큰 문제가 발생하는 경우 애초에 두개 이상의 객체를 선언할 수 없도록 클래스를 설계할 수 있는데 이러한 디자인 패턴을 Singleton이라고 부른다. 싱글톤 객체를 사용할 경우 쓰기는 조금 불편하지만 객체가 하나만 존재함을 보장할 수 있으며 객체를 별도로 선언하지 않고도 실제 객체를 사용할 때 생성할 수 있다. 단, 객체 파괴 시점을 컨트롤하기 어렵기 때문에 이에 대한 해결책도 함께 구현해야 한다. 

<br/>

# **2. GetInstance 안에 두는 방법**

```c++
class System
{
private:
    int _Data;
    //...

private:
    System() : _Data(0) {};
    ~System() {};

public:
    static System* GetInstance()
    {
        static System sys;
        return &sys;
    }
};

int main()
{
    System *pSys = System::GetInstance();
}
```

위처럼 static을 통해 전역적으로 동작하도록 만들면 GetInstance를 호출할 때 중복 객체를 안 만들면서도 객체에 접근할 수 있다. 객체를 지역적 혹은 동적으로 생성하는 것을 막기 위해 생성자는 private으로 하며, GetInstance도 static 함수로 만든다. thiscall이 없는 함수이기 때문에 이런 처리가 가능하다.

이 경우 메모리 확보는 프로세스 시작 시에, 객체 생성은 GetInstance 첫 호출 시에 이루어진다. 만약 호출하지 않으면 메모리는 확보하되 생성자, 소멸자는 호출하지 않는다. GetInstance 호출 순간에 객체가 생성되기 때문에 초기화 순서를 보장할 수 있다. 

그러나 위처럼 static을 함수 안에 만들 경우, 내부적으로 flag를 세워서 함수가 호출될 때마다 처음인지 아닌지를 판단하는 과정을 거치며 이것이 성능 저하를 유발할 수 있다. 이를 막기 위해 static System sys를 클래스 멤버 변수에 둘 수 있다. 

<br/>

# **3. 클래스 멤버 변수에 두는 방법**

```c++
class System
{
private:
    int _Data;
    static System _System;
    //...

private:
    System() : _Data(0) {};
    ~System() {};

public:
    static System* GetInstance()
    {
        return &_System;
    }
};

System System::_System;

int main()
{
    System *pSys = System::GetInstance();
}
```

과거에 인라인 문법이 안될 때는 System System::_System;를 꼭 선언해야 한다는 점이 부담이 되어서 잘 사용되지 않았으나, 최근에는 해당 문장을 인라인 문법으로 처리한 형태로 많이 사용한다. 이 경우 생성자 호출 시점을 컨트롤 할 순 없지만 GetInstance에서 분기를 하지 않기 때문에 성능면에서 이롭다. 

단점으로는 static 변수의 초기화 순서와 파괴 순서가 보장되지 않기에 다른 전역 변수나 다른 싱글톤이 이걸 사용하게 된다면 오작동이 날 수 있다. C++에서는 선언 순서에 대해 밝히지 않았으며, MS에서는 선언 순서로 생성, 역순으로 파괴된다고 밝히긴 했으나 이는 표준이 아니므로 언제든 바뀔 수 있다. 

따지자면 애초에 싱글톤끼리 강한 연관을 맺는 것 자체가 다소 위험한 설계 방식이다. 하지만 로그 싱글톤과 같이 시스템적인 역할을 하는 객체는 간혹 연관을 맺을 수 밖에 없기 때문에 그런 경우엔 동적 할당 혹은 지역 static으로 만드는 것이 좋다. 이를 해결하기 위해 피닉스싱글톤을 사용하기도 한다. 

<br/>

# **4. 동적 할당을 통한 구현 방법**

```c++
class System
{
private:
    int _Data;
    static System * _pSystem;
    //...

private:
    System() : _Data(0) {};
    ~System() {};

public:
    static System* GetInstance()
    {
        if (_pSystem == nullptr)
            _pSystem = new System;
        return _pSystem;
    }
};
System * System::_pSystem = nullptr;

int main()
{
    System *pSys = System::GetInstance();
}
```

이 경우 static 변수를 멤버로 두면서도 생성자 호출 시점을 컨트롤 할 수 있다. 단 시스템이 소멸자를 자동 호출해주지 못하기 때문에 메모리 누수가 생긴다. 어차피 누수가 생기는 시점이 프로세스가 끝나는 시점이므로 이를 그냥 감수하기도 하고, atexit에 소멸자를 등록시키는 방식을 이용하기도 한다.

<br/>

# **5. template을 이용한 구현 방법**

```c++
template <typename T>
class SingletonTemplate
{
protected:
    SingletonTemplate(){}
    virtual ~SingletonTemplate() {}

public:
    static T* GetInstance()
    {
        if (_pInstance == nullptr)
            _pInstance  = new T;
        return _pInstance;
    }

    static void DestroyInstance()
    {
        if(_pInstance)
        {
            delete _pInstance;
            _pInstance = nullptr;
        }
    }

private:
    static T* _pInstance; 
};
```
```c++
template <typename T> T* SingletonTemplate<System>
class System: public SingletonTemplate<System>
{
    friend SingletonTemplate<System>;

private:
    System() {};
    ~System() {};
    int _Data;
    // ...
}

int main()
{
    System *pSys = System::GetInstance();
}
```

상속을 통한 구현의 경우 성능 상의 이점은 없지만 클라이언트 개발에서는 지연된 처리, 즉 미리 호출하는 대신 필요한 시점에서 호출하는 것을 지향하는데 이러한 지향점과 방향성이 일치한다. 반면 서버 개발은 되도록 처음에 다 호출하여 일정하게 유지하는 것을 지향하기에 이런 방법이 유용하진 않다. 
