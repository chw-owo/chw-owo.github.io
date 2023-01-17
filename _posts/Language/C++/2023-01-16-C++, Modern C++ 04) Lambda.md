---
title: C++, Modern C++ 04) Lambda
categories: cpp
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part1: C++ 프로그래밍 입문> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 
## **1. Lambda**

함수 객체를 빠르게 만들 수 있는 문법. 코드로 예를 들어보자.

```c++
enum class ItemType
{
    None,
    Armor,
    Weapon,
    Portion
}

enum class Rarity
{
    Common, 
    Rare,
    Unique
}

class Item
{
public:
    Item() {}
    Item(int itemId, Rarity rarity, ItemType type)
    {
        _itemId = itemId;
        _rarity = rarity;
        _type   = type;
    }

public:
    int _itemId = 0;
    Rarity _rarity = Rarity::Common;
    ItemType _type = ItemType::None;
}

int main()
{
    vector<Item> v;
    v.push_back(Item(1, Rarity:Common, ItemType::Weapon));
    v.push_back(Item(2, Rarity:Common, ItemType::Armor));
    v.push_back(Item(3, Rarity:Rare,   ItemType::Portion));
    v.push_back(Item(4, Rarity:Unique, ItemType::Weapon));

    ...

    return 0;
}
```
위 상황에서 지금까지 배운 문법을 이용하여 Rarity==Unique인 아이템을 고르려면 아래와 같이 작성해야 한다. 

```c++
struct IsUnique
{
    bool operator()(Item& item)
    {
        return item._rarity == Rarity::Unique;
    }
}

auto findIt = std::find_if(v.begin(), v.end(), IsUnique());
```
람다식을 사용하면 struct IsUnique 부분을 아래와 같은 한줄로 줄일 수 있다. 

```c++
auto isUnique = [](Item& item){ return item._rarity == Rarity::Unique;};
auto findIt = std::find_if(v.begin(), v.end(), isUnique);
```
<br/>

## **2. Closure**

람다식에서 대괄호에 해당하는 부분을 capture라고 부른다. 함수 객체 내부에 변수를 저장하여 사용하듯이, capture를 이용해 외부 값을 저장하여 사용할 수 있다. 캡쳐를 활용하면 외부의 자유변수를 람다식 내부에 가져와서 그 값을 사용할 수 있게 되는데, 이렇게 외부의 자유변수를 참조하는 람다를 두고 Closure라고 부른다.

```c++
int itemId = 4;
auto findById = [=](Item& item){ return item._itemId == itemId;};
auto findIt = std::find_if(v.begin(), v.end(), findById);
```

위 예시처럼 \[=]를 사용하는 것은 외부 변수를 값 복사 방식으로 사용하겠다는 것을 의미한다. 

```c++
auto findById = [&](Item& item){ return item._itemId == itemId;};
auto findIt = std::find_if(v.begin(), v.end(), findById);
```

\[&]를 사용하는 것은 외부 변수를 참조 방식으로 사용하겠다는 것을 의미한다. 

```c++
auto findById = [&itemId, rarity, type](Item& item){ return ...; };
auto findIt = std::find_if(v.begin(), v.end(), findById);
```
위와 같이 섞어서 사용할 수도 있다. 이 경우 itemId만 참조로 받고 나머지는 복사 방식으로 받게 된다. C++에서는 가독성 및 실수 방지를 위해 하나하나 명시해주기를 권장하고 있다. 

이러한 Closure expression은 주의깊게 사용해야 한다. lambda에서 참조 및 주소를 받아 사용할 때는 해당 람다식이 실행되는 시점에 참조값이 제대로 들어있는지 잘 확인해야 한다. 만약 class 멤버 함수에서 람다식을 사용하면서 해당 객체의 멤버 함수를 가리키고 있었다고 가정해보자. 만약 해당 객체를 동적 할당 후 이미 해제하였는데 그걸 인지하지 못하고 람다 객체를 다시 호출하게 되면 람다식 안에서 오염된 메모리, 이미 값이 날라간 멤버 변수의 메모리를 가리키게 된다. 그러한 상황은 이후에 큰 피해를 불러올 수 있으니 반드시 유의해야 한다. 

<br/>

## **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part1: C++ 프로그래밍 입문, Rookiss
