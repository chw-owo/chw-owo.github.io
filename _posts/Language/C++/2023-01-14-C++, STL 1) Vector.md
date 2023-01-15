---
title: C++, STL 1) Vector
categories: Cpp
tags: 
toc: true
toc_sticky: true
---
## **1. STL와 컨테이너란?**

**1) STL, Standard Template Library**

프로그래밍 할 때 기본적으로 필요한 자료구조, 알고리즘들을 템플릿으로 제공하는 라이브러리. 

**2) 컨테이너** 

데이터를 저장하는 객체, 일종의 자료구조. 

**3) 시퀀스 컨테이너**

데이터가 삽입 순서대로 나열되는 형태의 컨테이너. 예시로 vector, list, deque가 있다. 

**4) 연관 컨테이너**

key-value처럼 관련있는 데이터를 쌍으로 저장하는 컨테이너. 예시로 map, set, multimap, multiset 등이 있다. 

<br/>

## **2. Vector의 동작 원리**

Vector은 동적 배열로 동작한다. 기존에 쓰던 일반적인 배열은 선언할 때 사이즈가 정해지기 때문에 숫자가 변하는 플레이어, 몬스터 등의 데이터를 담기엔 부적합하다. 이를 보완할 수 있는 자료형이 STL의 Vector다. Vector는 다음과 같이 선언할 수 있다. 

```c++
using namespace std;
#include<vector>

int main()
{
    vector<int> v;
    return 0;
}
```
Vector는 크기가 가변적이기 때문에 선언할 때 크기를 입력해주지 않는다. 그럼 Vector는 어떤 원리로 가변적인 크기를 갖는 것일까?

Vector가 가변적인 크기를 갖는 원리는 크게 두가지가 있다. 첫째로, Vector는 선언할 때 여유분을 두고 메모리를 할당한다. 둘째로, 여유분이 가득 차면 메모리를 증설한다. 이때, Size는 Vector에 실제로 들어있는 데이터의 개수를 Capacity는 여유분을 포함한 용량 개수를 의미한다. 

```c++
int main()
{
    vector<int> v;

    for(int i = 0; i < 1000; i++)
    {
        v.push_back(100);
        cout << v.size() << " " << v.capacity() << endl; 
    }
    return 0;
}
```
위 코드를 실행해보면 데이터를 넣었을 때 Vector의 Size와 Capacity가 어떻게 변하는지 확인할 수 있다. 출력해보면 Size는 1,2,3,4,5... 와 같이 실제 넣은 데이터의 개수만큼 변화하지만 Capacity는 1,2,3,4,5,9,13,19,28,42,63... 과 같이 점차 큰 용량만큼 늘려가는 것을 확인할 수 있다. 대략적으로 이전 capacity에서 1.5배정도 늘리는 것을 확인할 수 있다. 얼마나 더 늘리는지에 대한 건 컴파일러마다 조금씩 다르지만 점차 한번에 늘리는 크기를 더 키워가는 것은 동일하다. 

메모리를 증설할 때, 기존 Vector의 뒷부분에 사용할 수 있는 공간이 있다면 좋겠지만 그렇지 못할 확률이 높다. 그래서 Vector의 경우 메모리를 한번 늘릴 때 늘어난 크기 만큼 담을 수 있는 새 공간을 찾은 뒤, 기존에 있던 값을 그곳에 복사하고, 그 뒷부분에 값을 마저 넣는 방식으로 동작하게 된다. 만약 복사가 너무 자주 일어나게 되면 여기에 많은 시간이 소요된다. 따라서 Vector가 일정 이상 크기가 커지면 한번에 많은 Capacity를 늘려서 복사에 드는 연산 시간을 줄이는 원리로 동작하는 것이다. 

만약 한번에 최소 몇개 이상의 데이터를 넣을 것인지 정해져있다면 데이터를 넣기 전에 Capacity를 미리 설정해줄 수도 있다. 

```c++
int main()
{
    vector<int> v;
    v.reserve(1000);

    for(int i = 0; i < 1000; i++)
    {
        v.push_back(100);
        cout << v.size() << " " << v.capacity() << endl; 
    }
    return 0;
}
```

vector.reserve로 초기 capacity의 값을 정해줄 수 있다. 그러면 복사 연산에 드는 시간을 단축할 수 있다. 

```c++
int main()
{
    vector<int> v;
    v.resize(1000);

    for(int i = 0; i < 1000; i++)
    {
        v.push_back(100);
        cout << v.size() << " " << v.capacity() << endl; 
    }
    return 0;
}
```

유사하게 vector.resize로 초기 size의 값을 정해줄 수도 있다. 이때 capacity도 같이 resize된 값만큼 늘어나게 된다. 

```c++
using namespace std;
#include<vector>

int main()
{
    vector<int> v(1000);
    return 0;
}
```
만약 선언할 때 크기를 정해주고 싶다면 위와 같이 입력할 수도 있다. 
```c++
using namespace std;
#include<vector>

int main()
{
    vector<int> v(1000,0);
    return 0;
}
```
또, 동일한 값으로 모든 데이터를 초기화 하고 싶다면 위와 같이 입력하면 된다. 이 경우 vector v에 1000개의 0이 들어가게 되고, capacity 역시 1000개로 늘어난다. 

이때 capacity는 자동으로 늘어나긴 하지만 자동으로 줄어들진 않는다는 특징이 있다. 

```c++
int main()
{
    vector<int> v;
    for(int i = 0; i < 1000; i++)
    {
        v.push_back(100);
    }
    v.clear();
    cout << v.size() << " " << v.capacity() << endl; 
    return 0;
}
```
위 경우에 size와 capacity를 찍어보면 size는 0으로 돌아왔지만 capacity는 여전히 1000인 것을 확인할 수 있다. capacity도 0으로 되돌리는 문법은 없기 때문에, 만약 해당 vector의 capacity도 0으로 만들어주고 싶다면 간접적인 방법을 사용해야 한다. 

```c++
int main()
{
    vector<int> v;
    for(int i = 0; i < 1000; i++)
    {
        v.push_back(100);
    }
    v.clear();
    vector<int>().swap(v);
    cout << v.size() << " " << v.capacity() << endl; 
    return 0;
}
```
새로운 빈 vector를 만들어서 기존 vector와 swap 해주는 것이다. 그러나 capacity가 이미 늘어났다는 것은 해당 vector가 그만큼 늘어날 가능성이 또 있다는 것이기 때문에 이렇게 날려줘야 하는 경우는 흔하지 않다. 

<br/>

## **3. Vector 기초 문법**

**1) 복사**

```c++
vector<int> v1(100, 1);
vector<int> v2 = v1;
```
위와 같은 방법으로 복사할 수 있다. 

<br/>

**2) 삽입/삭제**

**-끝 삽입/삭제**

```c++
v.push_back(1);
v.pop_back();
```

끝 삽입 삭제의 경우 중간, 처음 삽입 삭제보다 효율적으로 동작한다. 따라서 vector에서는 push_back, pop_back을 이용하여 vector의 맨 뒤에 값을 삽입, 삭제할 수 있다. 

<br/>

**-처음/중간 삽입/삭제**

```c++
v.insert(v.begin() + 2, 5);             // 2번째 위치에 5 삽입
v.erase(v.begin() + 2);                 // 2번째 위치의 값 제거
v.erase(v.begin() + 2, v.begin() + 5);  // 2-4번째 위치의 값 연속 제거
```
insert, erase를 이용해 처음/중간 삽입/삭제가 가능하다. 이는 iterator을 사용하여 동작하는 것을 확인할 수 있다.  

그러나 이들은 push_back, pop_back만큼 효율적으로 동작하진 않는다. Vector는 원소가 하나의 메모리 블록에 연속하게 저장되어야 한다는 특징이 있다. 따라서 처음 혹은 중간에 값을 삽입할 경우 이후의 값들이 하나씩 모두 뒤로 이동해야 하므로 연산이 많이 필요한 비효율적인 동작이 된다. 또, 삭제의 경우에도 삭제된 영역을 빈칸으로 냅둘 수 없으므로 이후의 값들이 하나씩 모두 앞으로 이동해야하는데 이것도 연산이 많이 필요하다. 따라서 c++의 stl vector에서는 push_back, pop_back과 달리 처음/중간 삽입/삭제를 위한 명령어를 따로 제공하지 않고 있다.

<br/>

**3) 값 확인**

**-처음/끝 확인**

```c++
v.front();
v.back();
```
front, back을 이용하여 삭제를 하지 않고 첫 값, 마지막 값을 확인할 수 있다.

<br/>

**- 임의 접근**

```c++
int i = v[3];
```
일반적인 배열처럼 인덱스를 이용해 임의 접근 할 수 있다. 

<br/>

## **4. 반복자**

**1) 반복자 기본 문법**

iterator는 일종의 포인터와 유사하게 작동한다. 포인터와 비교해서 살펴보자. 
```c++

vector<int>::iterator it;
int* ptr;

it = v.begin();
ptr = &v[0];

cout << (*it) << endl;
cout << (*ptr) << endl;

it++;
++it
ptr++;
++ptr

it += 2;
it -= 2;
ptr += 2;
ptr -= 2;
```

이때 it를 확인해보면 내부적으로 주소값을 갖고 있다. 따라서 ++, -- 와 같은 덧셈 뺄셈 연산의 경우 포인터에서의 연산과 동일하게 동작한다. 그리고 주소값에 더해서, 추가적으로 Myproxy, Mynextiter와 같은 값을 통해 자신이 어떤 vector에 속해있는지 등 반복에 필요한 정보를 담고 있다. 

```c++
vector<int>::iterator itBegin = v.begin();
vector<int>::iterator itEnd = v.end(); 
```

메모리에서 확인해보면 v.begin()은 vector의 시작값을, v.end()는 vector 마지막 값의 바로 다음 값, 즉 유효한 범위의 바로 다음 주소를 갖고 있다. 따라서 v.end()가 가리키는 주소에는 쓰레기 값이 들어있는 것에 유의해야 한다. 

이걸 활용하여 반복을 수행하는 예시 코드를 보자.

```c++
for(vector<int>::iterator it = v.begin(); it != v.end(); ++it)
{
    cout << (*it) << endl;
}
```

참고로 it++보다 ++it가 조금 더 성능이 좋다. 연산자 오버로딩을 했을 때처럼, it++의 경우 기존 값을 복사하는 과정이 포함되기 때문이다. 

내부적으로는 아래 과정과 유사하게 동작한다. 

```c++
int* ptrBegin = &v[0];          // = v.begin().Ptr;
int* ptrEnd = ptrBegin + 10;    // = v.end()._Ptr;
for(int* ptr = ptrBegin; ptr != ptrEnd; ++ptr)
{
    cout << (*ptr) << endl;
}
```

일반 반복문과 비교했을 때, 이렇게 만들 경우의 장점은 iterator가 다른 컨테이너에서도 공유할 수 있는 문법이라는 것에 있다. Vector만 고려하면 인덱스 접근이 가능하기 때문에 일반적인 반복문을 사용하는 것이 더 나을 수도 있다. 그러나 인덱스 접근이 불가능한 컨테이너의 경우 iterator가 필요하게 된다. 

또, 역방향 iterator 역시 존재한다. 예시 코드는 아래와 같다. 

```c++
for(vector<int>::reverse_iterator it = v.rbegin(); it != v.rend(); ++it)
{
    cout << (*it) << endl;
}
```
reverse_iterator는 많이 사용되는 문법은 아니다.

<br/>

**2) 반복자 사용시 유의사항**

vector에는 특정한 값만 삭제하는 명령어가 따로 없기 때문에, 그러기 위해서는 iterator를 이용해야 한다. iterator를 이용해 특정 값을 삭제하는 코드를 보고, 반복자 사용시 유의사항을 살펴보자. 

```c++
for(vector<int>::iterator it = v.begin(); it != v.end(); ++it)
{
    int data = *it;
    if(data == 3)
        v.erase(it);
}
```

위 코드는 vector 전체에 반복문을 돌면서 값이 3이면 지워주는 코드이다. 그러나 실제로 실행해보면 에러가 나는 것을 볼 수 있다. 앞에서 언급했듯이 iterator는 단순히 주소를 가리키는 포인터가 아니라 해당 벡터에 대한 정보를 담고 있다. 따라서 erase를 해버리면 it는 더 이상 그 벡터에 대한 정보를 담고 있지 않게 되고, 그 벡터의 반복자로 쓰일 수 없게 된다. 따라서 아래와 같이 수정해주어야 한다. 

```c++
for(vector<int>::iterator it = v.begin(); it != v.end(); ++it)
{
    int data = *it;
    if(data == 3)
        it = v.erase(it);
}
```
erase는 삭제하는 순간 자신이 삭제한 위치에 대한 iterator값을 반환하기 때문에, 위와 같이 수정해주면 에러 없이 작동하는 것을 확인할 수 있다. 그러나 의도대로 동작작하지는 않을 것이다. 

만약 vector가 \[1][3][3][2][1]... 와 같이 이루어져 있었다고 해보자, 그럼 1번 index의 it가 지워지고, 다시 1번 index의 it를 반환할 것이다. 그리고 동시에 앞으로 값이 하나씩 밀려서 \[1][3][2][1]...과 같이 변할 것이다. 그러나 그 직후 ++it가 실행되면서 2번 index의 it를 검사하게 되기 때문에 결국 새롭게 1번 index에 들어온 3은 체크되지 못하고 넘어가는 것이다. 

```c++
for(vector<int>::iterator it = v.begin(); it != v.end();)
{
    int data = *it;
    if(data == 3)
        it = v.erase(it);
    else
        ++it;
}
```
이를 의도대로 작성하고 싶다면 위와 같이 수정해주어야 할 것이다. 

이처럼 반복자를 사용할 때는, 반복자가 일반적인 포인터가 아니라는 점, 벡터에 대한 정보를 갖고 있기 때문에 한번 삭제되면 더 이상 반복자의 역할을 할 수 없다는 점에 유의하여 사용해야 한다. 

<br/>

## **5. Vector 직접 구현해보기**

**Iterator**
```c++
template<typename T>
class Iterator
{
public:
    Iterator(): _ptr(nullptr)
    {

    }

    Iterator(T* ptr): _ptr(ptr)
    {
        
    }

    Iterator& operator++()
    {
        _ptr++;
        return *this;
    }

    Iterator operator++(int)
    {
        Iterator tmp = *this;
        _ptr++;
        return tmp;
    }

    Iterator& operator--()
    {
        _ptr--;
        return *this;
    }

    Iterator operator--(int)
    {
        Iterator tmp = *this;
        _ptr--;
        return tmp;
    }

    bool operator==(const Iterator& right)
    {
        return _ptr == right._ptr;
    }

    bool operator!=(const Iterator& right)
    {
        return _ptr != right._ptr;
    }

    Iterator operator+ (const int count)
    {
        Iterator tmp = *this;
        tmp._ptr += count;
        return tmp;
    }

    Iterator operator- (const int count)
    {
        Iterator tmp = *this;
        tmp._ptr -= count;
        return tmp;
    }
     
    T& operator*()
    {
        return *_ptr;
    }

public:
    T* _ptr;
}
```

**Vector**
```c++
template<typename T>
class Vector
{
public:

    Vector(): _data(nullptr), _size(0), _capacity(0)
    {

    }

    ~Vector()
    {
        if(_data)
            delete[] _data;
    }

    int size() { return _size; }
    int capacity() { return _capacity; }

    void push_back(const T& val)
    {
        if(_size == _capacity)
        {
            int newCapacity = static_cast<int>(_capacity * 1.5);
            if(newCapacity == _capacity)
                newCapacity++;

            reserve(newCapacity);
        }

        _data[_size] = val;
        _size++;
    }

    void reserve(int capacity)
    {
        _capacity = capacity
        T* newData = new T[_capacity];

        // 새 공간에 데이터 복사
        for(int i = 0; i< _size; i++)
            newData[i] = _data[i];

        // 기존 공간 삭제
        if(_data)
            delete[] _data;

        // 주소 교체
        _data = newData;
    }

    T& operator[](const int pos)
    {
        return _data[pos];
    }

    void clear() { _size = 0; }

public:
    typedef Iterator<T> iterator;
    iterator begin() { return iterator(&_data[0]); }
    iterator end()   { return begin() + _size; }

private:
    T* _data;
    int _size;
    int _capacity;

}
```


## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part1: C++ 프로그래밍 입문, Rookiss
