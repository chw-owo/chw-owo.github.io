---
title: C++, Modern C++ 01) Basic
categories: Cpp
tags: 
toc: true
toc_sticky: true
---
## **1. auto**

```c++
int a = 3; 
Knight b = Knight();
const char* c = "chw-owo";
```
```c++
auto a = 3; 
auto b = Knight();
auto c = "chw-owo";
```
이 두가지 경우 모두 동일하게 동작한다. 데이터를 보고 컴파일러가 알아서 판단해서 자료형을 정해주는 것이다. 이처럼 컴파일러가 형식을 추론해주는 것을 두고 형식 연역, type deduction이라고 한다. 위의 예시에서는 추론이 간단하지만, 경우에 따라 추론 규칙이 복잡해질 수도 있다. 만약 어떻게 해도 맞는 자료형을 찾을 수 없는 경우에는 에러가 뜬다. 

auto를 쓸 때 유의해야 할 점은, 기본 auto는 const와 &를 무시한다는 것이다. 

```c++
int& ref = a;
const int cst = a;

auto test1 = ref;
auto test2 = cst;
```
위의 경우에서 test1, test2의 자료형을 확인해보면 int&, const int가 아니라 일반적인 int로 되어있다. 따라서 auto로 const, & 자료형을 받으려고 한다면 아래처럼 따로 명시해주어야 한다. 

```c++
int& ref = a;
const int cst = a;

auto& test1 = ref;
const auto test2 = cst;
```
또, auto의 경우 기존의 자료형에 비해 데이터에 대한 정보를 직관적으로 전달해주지 못한다. 따라서 STL iterator를 사용할 때처럼, 자료형이 너무 길어져서 가독성이 떨어지는 경우에는 적절하게 사용하면 좋을 수 있지만, 일반적인 상황에서는 놔두는 게 좋다. 

<br/>

## **2. 중괄호 초기화 { }**

**1)** vector와 같은 컨테이너를 초기화할 때 사용할 수 있다.
```c++
vector<int> v3{1,2,3,4};
```

**2)** 축소 변환을 방지해준다. 
```c++
int x = 0;
double y{x};
```
위 코드와 같이 초기화 하게 되면 에러가 나면서 축소 변환을 방지해준다. 

**3)** class 초기화도 가능하다. 

```c++
Knight k{};
```
위의 경우 Knight의 일반 생성자를 사용하겠다는 걸 의미한다. 

```c++
class Knight
{
public:
    Knight (initializer_list<int> li)
    {
        ...
    }
}

int main()
{
    Knight k{1,2,3,4,5};
    return 0;
}
```
위와 같은 방식의 초기화도 가능해진다. 

```c++
class Knight
{
public:
    Knight (int a, int b)
    {
        ...
    }

    Knight (initializer_list<int> li)
    {
        ...
    }
}

int main()
{
    Knight k{1,2};  // Knight (initializer_list<int> li)가 호출된다 
    return 0;
}
```
class에서 중괄호 초기화를 사용할 때 주의점은, initializer_list가 우선시 되기 때문에 인자를 여러개 받게 되면 무조건 initializer_list가 실행되는 일이 생길 수 있다. 위 코드에서도 Knight (int a, int b)가 아니라 Knight (initializer_list<int> li)가 호출된다. 따라서 Knight (int a, int b)를 호출하고 싶다면 기본처럼 소괄호 초기화를 기본으로 가야 한다. 

이러한 단점들 때문에 vector와 같은 특수한 경우에만 중괄호 초기화를 선택하는 걸 선호하는 사람들도 있다. 반면 중괄호 초기화를 기본으로 가면 축소 변환을 방지하고, 초기화 방식을 통일할 수 있다는 장점이 있기 때문에 중괄호 초기화를 선호하는 사람들도 있다. 

<br/>

## **3. nullptr**

Modern C++이전에는 pointer에 null 값을 넣어주고 싶을 때 NULL 혹은 0 값을 사용했다. 지금도 NULL, 0을 넣어도 동작이 달라지지는 않는다. 그럼에도 nullptr을 사용하는 것은 함수 오버로딩과 관련있다. 

```c++
void Test(int a){ ... }
void Test(void* ptr) { ... }

int main()
{
    int* ptr = NULL;
    Test(NULL);
    return 0;
}
```
NULL은 내부적으로 0이기 때문에 위와 같은 상황에서 Test(int a)가 호출된다. 

```c++
void Test(int a){ ... }
void Test(void* ptr) { ... }

int main()
{
    int* ptr = nullptr;
    Test(nullptr);
    return 0;
}
```
위와 같이 바꿔줄 경우, NULL과 nullptr 둘 다 같은 값을 갖지만 위 상황에서는 Test(void* ptr)가 호출된다. 그럼 내부적으로 NULL과 nullptr은 어떻게 다른 걸까? 내부를 뜯어보면 0이라는 단순한 숫자 값보다는 객체에 가까운 형식을 띄고 있다. nullptr을 간단하게 구현해보면 아래와 같다. 

```c++
const
class 
{
public:
    // 어떤 타입의 포인터와도 치환 가능
    template<typename T>   
    operator T* () const
    {
        return 0;
    }

     // 어떤 타입의 멤버 포인터와도 치환 가능
    template<typename C, typename T>    
    operator T C::* () const
    {
        return 0;
    }

private:
    // private에 선언하여 주소값 &를 막는다
    void operator&() const; 
} _Nullptr;
```
class의 중괄호 끝에 이름을 붙이면 class가 만들어짐과 동시에 객체를 선언하는 것과 같은 효과를 갖는다. 이에 const를 더해 상수처럼 쓸 수 있도록 만들 수 있다. 

이처럼 nullptr은 단순한 0과는 다른 것을 확인할 수 있다. 가독성을 위해서도, 오작동 방지를 위해서도 pointer에서는 nullptr을 쓰는 것이 좋다. 

<br/>

## **4. using**

using은 Modern C++에서 도입된 typedef과 유사하게 사용할 수 있는 문법이다. using은 typedef와 비교했을 때 아래와 같은 장점을 지니기 때문에 최근에 많이 활용되고 있다.  

```c++
typedef int id1;
using id2 = int;
```
**1)** typedef 보다 나은 직관성

```c++
typedef void (*VoidFunc)();
using VoidFunc = void(*)();
``` 
일반적인 변수의 경우에는 큰 차이가 없지만, 함수 포인터를 정의해줄 때는 더 나은 직관성을 보인다. 

**2)** template 활용의 편리함

```c++
template<typename T>
using List = std::list<T>;

int main()
{
    List<int> li;
    ...
}

```
typedef는 template을 받는 자료형을 새롭게 정의하기 어려운데, using은 이러한 작업을 쉽게 할 수 있다. 참고로 typedef로 위와 같은 작업을 하기 위해서는 아래와 같이 코드가 다소 복잡해진다. 

```c++
template<typename T>
struct List
{
    typedef std::list<T> type;
}

int main()
{
    List<int>::type li;
    ...
}
```

당연히 typedef를 이해할 수는 있어야 겠지만, 최근에는 using을 더 많이 이용하는 추세라고 한다. 

<br/>

## **5. enum class**

기존의 enum은 아래와 같이 사용되었다.  

```c++
enum PlayerType: char
{
    PT_Knight,
    PT_Archor,
    PT_Mage
}
```

이때, enum안에서 사용한 이름은 밖에서 사용할 수 없다. 그래서 enum이라는 표시로 PT_ 와 같이 약자를 관습적으로 앞에 붙여주었다. 이처럼 범위가 정해져있지 않은 enum을 두고 unscoped enum이라고 부른다. 이의 한계점을 보완한 것이 enum class이다. 

```c++
enum class PlayerType
{
    Knight,
    Archor, 
    Mage
}
```

<br/>

enum class의 특징은 두가지가 있다. 

**1)** 이름 공간 관리 (scoped enum)

이는 enum class의 장점 중 하나이다. enum과 달리 영역 밖에서는 쓰이지 않기 때문에, enum 앞에 Knight가 있어도 밖에서 다른 Knight를 선언할 수 있다. 

<br/>

**2)** 암묵적인 변환 금지

기존 enum
```c++
double value = PT_Knight;
```
enum class
```c++
double value = static_cast<double>(PlayerType::Knight);
```

이는 enum class의 장점이자 단점이다. 변환을 위해서는 cast를 명시해주어야만 한다. 

<br/>

## **6. delete**

동적 할당에서 쓰이는 new/delete와는 다르다. 여기서의 delete는 삭제된 함수를 표시하기 위해 사용된다. 간혹 컴파일러가 자동으로 생성해주지만 생성되지 않기 바라는 함수가 있을 수 있다. 예를 들어, 위의 nullptr 구현 예시에서 private 처리한 & 연산자가 그에 해당할 것이다. 비록 private을 이용해 밖에서 쓰이지 못하게 만들었지만, 이는 여전히 class 내부에서 사용할 수 있기에 완벽히 막았다고 할 수 없다. 실제로 그 상황에서 해당 함수를 호출하려고 하면, 애초에 실행이 안되는 게 아니라 Link 에러가 뜨는 걸 확인할 수 있다. 문제를 조기에 발견하지 못한 셈이다. 이런 상황에서 delete를 사용할 수 있다. 

예를 들어, Admin 같이 복사가 절대 되면 안되는 class가 있다고 가정해보자. 

```c++
class Admin
{
public:
    void operator=(const Admin& a) = delete;

private:
    ...
}
```
위와 같이 delete 표시를 해주면, 해당 함수를 호출하는 부분에서 "삭제된 함수를 참조하려고 했다"며 명확한 에러 메세지를 띄워주는 것을 확인할 수 있다. 

<br/>

## **7. override, final**

이 부분은 가상함수와 연관이 있다. 

```c++
class Player
{
public:
    void Attack()
    {
        cout << "Player Attack" << endl;
    }
};

class Knight
{
public:
    void Attack()
    {
        cout << "Knight Attack" << endl;
    }
};

int main()
{
    Player* p = new Knight();
    p -> Attack();
    delete p;

    return 0;
}
```
위처럼 부모의 멤버 함수를 재정의 하는 것을 override라고 한다. 위의 경우에서 p는 Knight로 동적할당을 했음에도 Attack을 실행해보면 override 이전엔 Player의 Attack을 호출하는 것을 확인할 수 있다. 

```c++
class Player
{
public:
    virtual void Attack()
    {
        cout << "Player Attack" << endl;
    }
};

class Knight : public Player
{
public:
    virtual void Attack()
    {
        cout << "Knight Attack" << endl;
    }
};

int main()
{
    Player* p = new Knight();
    p -> Attack();
    delete p;

    return 0;
}
```

이때, Attack 앞에 virtual을 붙여서 가상 함수로 만들어주면, Player*로 선언했어도 자동으로 Knight에 맵핑되는 걸 확인할 수 있었다. [객체지향 포스트의 다형성 파트 참고][1] 부모 클래스의 멤버 함수에만 virtual을 붙여도, 자식 클래스의 해당 멤버 함수도 자동으로 가상 함수로 등록이 되었다. 그런데 상속 받은 함수의 시그니쳐가 달라지면 부모의 멤버 함수와 아예 다른 함수로 인식된다. 

```c++
class Player
{
public:
    virtual void Attack()
    {
        cout << "Player Attack" << endl;
    }
};

class Knight : public Player
{
public:
    virtual void Attack() const
    {
        cout << "Knight Attack" << endl;
    }
};

int main()
{
    Player* p = new Knight();
    p -> Attack();
    delete p;

    return 0;
}
```

위의 경우 const 하나가 붙은 거지만 이전처럼 "Player Attack"이 출력되게 된다. Knight의 Attack이 가상함수로 등록되어있긴 하지만 Player의 Attack을 상속 받았다고 인식되지 않는 것이다. 이런 실수를 방지하기 위해 override 한 멤버 함수의 경우 뒤에 override를 붙이는 방식으로 표시해줄 수 있다. 

```c++
class Knight : public Player
{
public:
    virtual void Attack() const override
    {
        cout << "Knight Attack" << endl;
    }
};
```
이 경우 부모클래스가 void Attack() const를 갖고 있지 않다는 에러와 함께 실행할 수 없게 된다. override 붙이는 습관을 들일 경우 실수도 줄이고, 가독성도 더 높일 수 있다. 만약 더 이상 이 함수는 override 하지 않겠다고 선언하고 싶다면 함수 뒤에 final을 붙이면 된다. 

```c++
class Player
{
public:
    virtual void Attack() final
    {
        cout << "Player Attack" << endl;
    }
};
```
<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part1: C++ 프로그래밍 입문, Rookiss

[1]:https://chw-owo.github.io/cpp/C++,-%EA%B0%9D%EC%B2%B4%EC%A7%80%ED%96%A5-2\)-%EA%B0%9D%EC%B2%B4%EC%A7%80%ED%96%A5%EC%9D%98-%ED%8A%B9%EC%A7%95%EA%B3%BC-Class-%EC%8B%AC%ED%99%94/#3-%EB%8B%A4%ED%98%95%EC%84%B1