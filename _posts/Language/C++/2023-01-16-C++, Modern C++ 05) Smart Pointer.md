---
title: C++, Modern C++ 05) Smart Pointer
categories: Cpp
tags: 
toc: true
toc_sticky: true
---
## **1. 스마트 포인터란?**

```c++
class Knight
{
public:
    ~Knight()
    {
        cout << "Knight 소멸" << endl;
    }

    void Attack()
    {
        if(_target)
        {
            _target->_hp -= _damage;
        }
    }
public:
    int _hp = 10;;
    int _ damage = 10;
    Knight* _target = nullptr;
};

int main()
{
    Knight* k1 = new Knight();
    {
        Knight* k2 = new Knight();
        k1->_target = k2;   
    } 
    k1->Attack();

    return 0;
}
```
<br/>

k1가 _target으로 삼았던 k2가 사라졌지만, 그럼에도 여전히 k1는 k2가 이전에 있었던 포인터를 가리키고 있다. 그러면서 k2와는 상관 없는 이상한 주소값을 건드리고 있는 것이다. 이런 상황은 후에 수정하기 어려운 심각한 피해를 가져올 수 있다. 이처럼 생포인터를 직접적으로 사용하는 것은 위험한 상황을 야기할 수 있다. 그래서 요즈음은 스마트 포인터를 이용하여 프로그램의 안전성을 보장하려는 추세이다. 

스마트포인터란 포인터를 알맞은 정책에 따라 관리하는 객체로, 포인터를 래핑해서 사용한다. 핵심적인 스마트 포인터로는 shared_ptr, weak_ptr, unique_ptr 세가지가 있다. 

## **2. shared_ptr**

shared_pt는 포인터의 레퍼런스 카운트를 관리한다. 객체를 delete하기 전에 해당 객체를 참조하고 있는 객체들을 체크하여, 더 이상 참조하는 객체가 없을 때 해당 객체를 delete한다. 이 기능을 간단하게 구현해보자면 아래와 같다. 
```c++
class RefCountBlock
{
public:
    int _refCount = 1;
}

template<typename T>
class SharedPtr
{
public:
    SharedPtr(){}
    SharedPtr(T* ptr): _ptr(ptr)
    {
        if(_ptr != nullptr)
        {
            _block = new RefCountBlock();
            cout<< _block -> _refCount << endl;
        }
    }

    SharedPtr(const SharedPtr& sptr): _ptr(sptr._ptr), _block(sptr._block)
    {
        if(_ptr != nullptr)
        {
            _block -> _refCount++;
            cout<< _block -> _refCount << endl;
        }
    }

    void operator=(const SharedPtr& sptr)
    {
         _ptr = sptr._ptr;
         _block = sptr._block;

        if(_ptr != nullptr)
        {
            _block -> _refCount++;
            cout<< _block -> _refCount << endl;
        }
    }

    ~SharedPtr()
    {
        if(_ptr != nullptr)
        {
            _block -> _refCount--;
            cout<< _block -> _refCount << endl;

            if(_block -> _refCount == 0)
            {
                delete _ptr;
                delete _block;
                cout<< "Delete Data!" << endl;
            }
        }
    }
}

...

int main()
{
    SharedPtr<Knight>k1;
    {
        SharedPtr<Knight>k2(new Knight());
        k2 = k1;
    }
    return 0;
}
```

위와 같이 shared pointer를 구현하여 Knight를 생성하게 되면, 소멸되더라도 바로 삭제되는 게 아니라 참조하고 있는 객체를 체크한 뒤에 삭제되게 된다. 실제 shared pointer는 아래와 같이 사용한다. 

```c++
int main()
{
    shared_ptr<Knight> k1 = make_shared<Knight>();
    {   
        shared_ptr<Knight> k2 = make_shared<Knight>();
        k1 -> _target = k2;
    }
    k1->Attack();

    ...
}
```
그러나 shared_ptr의 동작 원리 상, 만약 Player가 서로를 참조하는 등의 순환이 발생했을 때, 프로그래머가 직접 참조하는 포인터를 nullptr로 처리해주지 않으면 데이터가 영영 사라지지 않는 문제가 발생할 수 있다. 이러한 문제를 보완하기 위해 나온 것이 weak_ptr이다 

<br/>

## **3. weak_ptr** 

_refCount와 함께 메모리가 날라갔는지 날라가지 않았는지 확인하기 위핸 _weakCount가 등장한다. 

```c++
class Knight
{
public:
    ~Knight()
    {
        cout << "Knight 소멸" << endl;
    }

    void Attack()
    {
        if(_target.expired() == false)
        {
            shared_ptr<Knight> sptr = _target.lock();
            sptr->_hp -= _damage;
        }
    }
public:
    int _hp = 10;;
    int _ damage = 10;
    Knight* _target = nullptr;
};
```

_target.expired로 참조하는 대상이 유효한 메모리인지 체크하고, 유효한 메모리일 경우 _target.lock()으로 변환하는 과정이 추가된다. 직접적으로 객체의 생명 주기에 관여하지는 않지만, 해당 객체가 유효한지 간접적으로 체크하고 관리하는 역할을 한다. 이를 통해 더 이상 유효한 메모리가 아닌 경우 순환 참조를 끊을 수 있게 된다. 

<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part1: C++ 프로그래밍 입문, Rookiss
