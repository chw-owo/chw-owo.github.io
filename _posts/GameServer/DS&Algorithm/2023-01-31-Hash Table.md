---
title: Hash Table
categories: DS&Algorithm
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part3: 자료구조와 알고리즘> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Map vs Hash Map**

Map은 레드 블랙 트리, 자동으로 균형을 잡는 이진 탐색 트리로 구현되었다. 따라서 빠른 추가, 삭제, 탐색이 가능하며 일반적으로 이에 O(logN)의 시간이 소요된다.

Hash Map은 C++ 표준에서 unordered_map으로 구현되어있다. 이는 추가, 탐색, 삭제에 O(1)의 시간이 소요되며, 메모리를 더 많이 차지하는 대신 더 빠른 속도로 동작한다. 

# **2. Hash Table**

아주 기본적인 테이블을 만든다고 하면, 실제 사용할 만큼 메모리 공간을 잡고 각 공간에 해당하는 index로 찾아갈 수 있도록 구성될 것이다. 이를 코드로 구현하면 다음과 같다. 

```c++
void TestTable()
{
    struct User
    {
        int userId = 0;
        string username;
    }

    vector<User> users;
    users.resize(1000);
    users[777] = User{ 777, "chw-owo" };

    string name = users[777].username'
    cout << name << endl;
}
```
이 경우 index를 기반으로 찾아가기 때문에 빨리 찾아갈 수 있지만, 1000개 이상의 데이터를 담을 수 없다. 많은 데이터를 담으려면 반드시 그만큼 많은 메모리를 차지해야 하는데 데이터가 늘어날 경우 메모리 차지가 심해질 수 있다. 이를 해결하기 위해 Hash Table에서는 Hash 기법을 사용한다.

Hash는 임의의 길이의 데이터를 고정된 길이의 데이터로 매핑하는  함수를 의미한다. Hash를 거칠 경우, 입력 값을 알아볼 수 없는 아예 다른 값을 출력하게 되며, 새롭게 들어온 값을 Hash 하여 비교함으로써 기존 값과 일치하는지 확인할 수는 있지만 역으로 복호화하여 기존 값이 무엇이었는지는 알 수 없게 된다. 이에 대한 간단한 예제는 아래와 같다.

```c++
void TestSimpleHashTable()
{
    struct User
    {
        int userId = 0;
        string username;
    }

    vector<User> users;
    users.resize(1000);

    const int userId = 123456789;
    int key = (userId % 1000); //789

    users[key] = User{ userId, "chw-owo" };

    User& user = users[key];
    if(user.userId == userId)
    {
        string name = users[key].username'
        cout << name << endl;
    }
}
```
이렇게 할 경우 더 많은 값을 저장할 수 있게 된다. 그러나 위 예시에서는 userId로 123456789 대신 11111789가 주어져도 둘 중 어디에 해당하는지 알 수 없다는 단점이 생긴다. 이렇게 hash 결과를 key로 갖는 요소가 중복되는 현상을 Hash 충돌이라고 부른다. 

이를 해결하기 위한 기본 방법으로는 선형 조사법과 이차 조사법을 사용한다. 선형조사법의 경우 충돌난 지점에서 hash(key)에 1, 2, 3 ... 등을 더해서 그 옆자리에 데이터를 넣는 것을 의미한다. 이차 조사법의 경우 데이터 간에 간격을 주기 위해 hash(key) 에 2의 제곱, 3의 제곱 등 ... 을 더해서 그 옆자리에 데이터를 넣는 것을 의미한다. 그 외에도 Hash 충돌에 대처하는 여러 알고리즘이 있다. 

그 중에서도, 충돌 난 지점에 연결하여 데이터를 넣는 Chaining 기법으로 Hash 충돌을 해결한 예제를 들어보면 아래와 같다. 


```c++
void TestSimpleHashTable_Chaining()
{
    struct User
    {
        int userId = 0;
        string username;
    }

    vector<vector<User>> users;
    users.resize(1000);

    const int userId = 123456789;
    int key = (userId % 1000); //789

    users[key].push_back(User{ userId, "chw-owo" });

    vector<User>& bucket = users[key];
    for(User& user : bucket)
    {
        if( user.userId == userId )
        {
            string name = users[key].username'
            cout << name << endl;
        }
    }
}
```
이렇게 부가적인 기법을 사용한다고 해도 tree, list 등을 이용한 자료구조와 달리 자료량에 비례하는 연산을 거칠 일이 많지 않기 때문에 여전히 O(1)의 시간 복잡도를 갖는다.

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part3: 자료구조와 알고리즘, Rookiss
