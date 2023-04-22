---
title: Memory 01) Smart Pointer
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Reference Counting**

객체에서 다른 객체를 참조하는 상황의 경우, 만약 참조하던 객체가 사라지면 문제가 발생할 수 있다. 이 경우 Reference Counting, 즉 몇번이나 참조되었는지 확인하는 코드를 통해 문제를 예방할 수 있다. 

**RefCounting.h**

```c++
class RefCountable
{
public:
    RefCountable() : _refCount(1) {}
    virtual ~RefCountable() {}

    int32 GetRefCount() { return _refCount; }
    int32 AddRef() { return ++_refCount; }
    int32 ReleaseRef() 
    {
        int32 refCount = --_refCount;
        if(refCount == 0)
            delete this;

        return refCount; 
    }
protected:
    atomic<int32> _refCount;
}
```

**Main.cpp**

```c++
class Wraight: public RefCountable
{
public:
    //...
}

class Missile: public RefCountable
{
public:
    void SetTarget(Wraight* target)
    {
        _target = target;
        target -> AddRef();
    }

    void Update()
    {
        if(_target == nullptr)
            return;
        
        //...

        if(_target -> _hp == 0)
        {
            _target -> ReleaseRef();
            _target = nullptr;
        }
    }

    Wraight* _target = nullptr;
}

```

그러나 위와 같은 코드는 싱글 스레드에서는 잘 동작하지만 멀티 스레드에서는 Reference Counting을 잘못하는 문제가 발생할 수 있다. 이러한 상황을 막기 위해 Smart Pointer를 사용할 수 있다. 

<br/>

# **2. Smart Pointer 구현**

**RefCounting.h**

```c++
template<typename T>
class TSharedPtr
{
public: 

TSharedPtr() { }
	TSharedPtr(T* ptr) { Set(ptr); }

	
	TSharedPtr(const TSharedPtr& rhs) { Set(rhs._ptr); } // 복사
	TSharedPtr(TSharedPtr&& rhs) { _ptr = rhs._ptr; rhs._ptr = nullptr; } // 이동
	template<typename U>
	TSharedPtr(const TSharedPtr<U>& rhs) { Set(static_cast<T*>(rhs._ptr)); } // 상속 관계 복사

	~TSharedPtr() { Release(); }

public:
	// 복사 연산자
	TSharedPtr& operator=(const TSharedPtr& rhs)
	{
		if (_ptr != rhs._ptr)
		{
			Release();
			Set(rhs._ptr);
		}
		return *this;
	}

	// 이동 연산자
	TSharedPtr& operator=(TSharedPtr&& rhs)
	{
		Release();
		_ptr = rhs._ptr;
		rhs._ptr = nullptr;
		return *this;
	}

	bool		operator==(const TSharedPtr& rhs) const { return _ptr == rhs._ptr; }
	bool		operator==(T* ptr) const { return _ptr == ptr; }
	bool		operator!=(const TSharedPtr& rhs) const { return _ptr != rhs._ptr; }
	bool		operator!=(T* ptr) const { return _ptr != ptr; }
	bool		operator<(const TSharedPtr& rhs) const { return _ptr < rhs._ptr; }
	T*			operator*() { return _ptr; }
	const T*	operator*() const { return _ptr; }
				operator T* () const { return _ptr; }
	T*			operator->() { return _ptr; }
	const T*	operator->() const { return _ptr; }

	bool IsNull() { return _ptr == nullptr; }

private:
    inline void Set(T* ptr)
    {
        _ptr = ptr
        if(ptr)
            ptr -> AddRef();
    }

    inline void Release()
    {
        if(_ptr != nullpty)
        {    
            _ptr -> ReleaseRef();
            _ptr = nullptr;
        }
    }
private:
    T* _ptr = nullptr;
}
```
**Main.cpp**

```c++
class Wraight : public RefCountable
{
public:
	int _hp = 150;
	int _posX = 0;
	int _posY = 0;
};

using WraightRef = TSharedPtr<Wraight>;

class Missile : public RefCountable
{
public:
	void SetTarget(WraightRef target)
	{
		_target = target;
	}

	bool Update()
	{
		if (_target == nullptr)
			return true;

		int posX = _target->_posX;
		int posY = _target->_posY;

		if (_target->_hp == 0)
		{
			_target = nullptr;
			return true;
		}

		return false;
	}

	WraightRef _target = nullptr;
};

using MissileRef = TSharedPtr<Missile>;

int main()
{	
	WraightRef wraight(new Wraight());
	wraight->ReleaseRef();
	MissileRef missile(new Missile());
	missile->ReleaseRef();

	missile->SetTarget(wraight);

	wraight->_hp = 0;
	wraight = nullptr;
	
	while (true)
	{
		if (missile)
		{
			if (missile->Update())
			{
				missile = nullptr;
			}
		}
	}

	missile = nullptr;
}
```

위와 같은 과정을 통해 Smart Pointer를 간단하게 구현할 수 있다. 그러나 위와 같은 구현은 상속을 기반으로 동작하기 때문에 이미 만들어진 클래스 대상으로는 사용이 불가능하다. 이때 C++에서 제공하는 Shared_ptr을 사용할 수 있다. 

<br/>

# **3. 표준 smart pointer**

자세한 설명은 [스마트 포인터 포스트][1] 참조

**unique_ptr**

```c++
unique_ptr
```
이동을 할 수 없는 유일한 ptr로, 이동을 시키기 위해서는 이동 연산자를 사용해야 한다. 

**shared_ptr**
```c++
shared_ptr<Palyer> spr(new Player());
shared_ptr<Palyer> spr = make_shared<Player>();
```

new를 이용해 할당할 경우, Player를 할당한 다음 Player와 RefCountingBlock을 포함한 shared_ptr spr을 할당하게 된다. 반면 make_shared를 사용할 경우 처음부터 Player와 RefCountingBlock을 포함한 shared_ptr spr을 할당한다. RefCountingBlock의 경우 shared_ptr, weak_ptr이 공유하여 사용하게 되며, _Uses, _Weaks라는 두가지 atomic 정보를 갖고 있다. 

**weak_ptr**

```c++
weak_ptr<Palyer> wpr = spr; 
```
weak_ptr에서는 위와 같이 spr를 받아서 저장할 수 있다. 반면 wpr을 spr에 넣기 위해서는 아래와 같이 wpr이 있다는 보장을 받은 후 사용해야 한다. 
```c++
bool expired = wpr.expired();
shared_ptr <Knight> spr2 = wpr.lock();
```
만약 wpr이 없을 경우 spr2에 null이 들어가게 된다. 

<br/>

# **4. 참조 순환**

위 예제도, Shared_ptr도 모두 순환 (Cycle) 문제를 피하지 못한다. 이는 객체가 서로를 참고하고 있을 때 참조 순환이 발생하여 영원히 소멸되지 않는 것을 의미한다. 이 경우 각 객체를 없애기 전에 서로 참조하고 있던 ptr 값을 nullptr로 명시적으로 밀어주는 방식을 통해 해결할 수 있다. 

이런 일이 소유 관계에서 실수로 발생하기도 한다. 예를 들어 플레이어와 아이템이 서로를 참조하고 있다면, 아이템은 플레이어의 소유이기 때문에 일반적으로 플레이어에 대한 reference counting을 하지 않는다. 따라서 소유 관계에서 실수로 참조하는 일이 생기지 않도록 주의해야 한다. 

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss

[1]: https://chw-owo.github.io/cpp/C++,-Modern-C++-05)-Smart-Pointer/