---
title: Procademy, Q1, 25) Design Pattern
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 옵저버 패턴**

한 객체가 바뀌었을 때 그에 의존하는 객체들한테 바뀐 것을 직접 전달하고 갱신시키는 패턴이다. 의존하는 객체(Observer)측에서 주기적으로 폴링하는 것이 구조상 더 간단하겠지만, 이 경우 낭비가 생길 수 있기 때문에 효율성을 위해 Subject에서 전파를 시키는 것이다. 구현 방식에는 여러가지가 있는데 이벤트 방식으로 시그널을 보낼 수도 있고, 상속 관계를 통해 구현할 수도 있다. 

```c++
class Observer
{
public:
    virtual void Update(float fData) = 0;
    std::string _szName;
}

class Subject
{
public:
    virtual void RegisterObserver(Observer* pO) = 0;
    virtual void RemoveObserver(Observer* pO) = 0;
    virtual void NotifyObservers() = 0;
} 
```

```c++
class WeatherData: public Subject
{
public:
    void RegisterObserver(Observer* pO) 
    {
        _mapObservers.insert(STL_MAP_OBSERVERS_VT(pO->_szName, pO));
    }

    void RemoveObserver(Observer* pO)
    {
        _mapObservers.erase( pO->_szName );
    }

    void NotifyObservers() 
    {
        STL_MAP_OBSERVERS_ITR itr =  _mapObservers.begin();
        while(itr !=  _mapObservers.end())
        {
            Observer* pObserver = itr->second;
            pObserver->Update(_fData);
            ++itr;
        }
    }

public:
    float GetData() { return _fData; }
    void SetData(float fData)
    {
        _fData = fData;
        NotifyObservers();
    }

private:
    float _fData;

private:
    typedef std::map <std::string, Observer*>   STL_MAP_OBSERVERS;
    typedef STL_MAP_OBSERVERS::iterator         STL_MAP_OBSERVERS_ITR;
    typedef STL_MAP_OBSERVERS::value_type       STL_MAP_OBSERVERS_VT;
    STL_MAP_OBSERVERS                           _mapObservers;
}

class ConditionDisplay : public Observer
{
public:
    ConditionDisplay(Subject* pSubject)
    {
        pSubject->RegisterObserver(this);
        _pSubject = pSubject;
    }

    void Update(float fData)
    {
        _fData = fData;
    }

    void Display()
    {
        cout << "Temperature: " << _fData << endl;
    }

private:
    Subject*    _pSubject;
    float       _fData;
}

int main()
{
    WeatherData* pWeatherData = new WeatherData;
    ConditionDisplay* pConditionDisplay = new ConditionDisplay(pWeatherData);

    pWeatherData->SetData(10.5f);
    pConditionDisplay->Display();
}
```

이 코드는 상속을 이용해 Subject 측에서 코드를 수정하지 않아도 Observer를 추가할 수 있게끔 한 것이다. 우선 Observer, Subject 추상 클래스를 구현한 후, Subject가 Observer의 포인터들을 갖고 있다가 Update로 데이터를 갱신하게끔 만든다. 이 예시의 경우 생성자에서 Register 하고 있지만 상황에 따라서는 외부에서 생성, 소멸을 관리하는 것이 더 관리하기 좋을 수 있다.

이런 구조는 Observer를 추가할 일이 많을 경우 코드 재사용성 측면에서 장점이 있다. 그러나 실제로 작업을 하다보면 Observer를 추가할 일이 많이 생기지는 않는다고 한다. 상속이 늘어날 수록 유지 보수, 확장하는 입장에서 분석하기 어려워지므로 차라리 Observer를 Subject의 멤버로 직접 추가하는 것이 더 직관적이고 깔끔할 수도 있으며, 상황에 맞게 적절히 사용해야 한다.

<br/>

# **2. 커맨드 패턴**

특정 스레드에게 어떤 함수를 실행하도록 시키고 싶을 때 일반적으로는 이를 메시지로 전달한다. 이때 메시지 큐를 많이 쓰는데, 큐에 메시지를 넣게 되면 스레드에서 하나씩 deque해서 순차적으로 실행한다.  

```c++
// Allocate msg
msg.Type = 0;
msg.Param = "...";
msgQ.Enque(msg);
```

```c++
msgQ.Deque(&msg);

switch (msg.Type)
{
case 0:
    //...
    break;
case 1:
    //...
    break;
}
```

이 과정을 간단하게 적어보면 위와 같다. 여기에 커맨드 패턴을 적용하면 타입을 지정하고 체크하는 과정을 간단하게 줄일 수 있다. 

```c++
class ICommand
{
public:
    virtual void Exec () = 0;
    //...
}
```
```c++
//allocate msg
msg -> SetMsg();
msgQ.Enqueue(msg);
```
```c++
msgQ.Deque(&msg);
msg->Exec();
```

커맨드 패턴의 간단한 예시는 위와 같다. 가상함수를 이용해 각자의 작업을 각자의 클래스에서 실행하게 되므로 코드가 단순해진다. 

**+)** 요즘은 현업에서도 커맨드 패턴을 많이 사용하지만, 유지 보수 측면에서 여전히 switch case를 더 선호하는 경우도 있다. 문제가 생겼을 때 switch case는 switch 내부만 보면 되는 반면 커맨드 패턴은 클래스를 찾아들어가서 확인해야 되기 때문이다. 
