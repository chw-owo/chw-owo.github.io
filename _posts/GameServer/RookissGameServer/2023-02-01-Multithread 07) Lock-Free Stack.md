---
title: Multithread 07) Lock-Free Stack
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **0. Lock-Free 개요**

Lock을 사용하지 않고도 멀티스레드 환경에서 동기화가 이루어질 수 있는 Stack과 Queue를 구현한다. Lock을 명시적으로 사용하지는 않는다고 해도 compare_exchange_weak등과 같은 동기화 작업을 해주기 때문에 여러 스레드에서 동시에 push 하는 경우 lock-based에서와 마찬가지로 대기 지연 시간이 생기며, 성능 향상이 눈에 띄게 크게 일어나지는 않는다. 멀티스레드 환경에서 사용가능한 Lock-free 자료구조를 구현하는 것에는 여러 알고리즘이 있으나, 여기서는 가장 단순하고 직관적인 방법들로 구현이 되었다. 

<br/>

# **1. 참조 Count를 직접 활용하는 방법**

## **1) 전체 코드**

```c++
template<typename T>
class LockFreeStack
{
    struct Node
    {
        Node (const T& value): data(value), next(nullptr){ }
        T data;
        Node* next;
    };

public:
    void Push(const T& value)
    {
        Node* node = new Node(value);
        node-> next = _head;

        while( _head.compare_exchange_weak(node->next, node) == false) { }
    }

    bool TryPop(T& value)
    {
        ++_popCount;

        Node* oldHead = _head;
        while(oldHead && _head.compare_exchange_weak(oldHead, oldHead -> next) == false){ }

        if(oldHead == nullptr)
        {
            --_popCount;
            return false;
        }
            
        value = oldHead -> data;
        TryDelete(oldHead);
        return true;
    }

    void TryDelete(Node* oldHead)
    {
        if(_popCount == 1)
        {
            Node* node = _pendingList.exchange(nullptr);

            if(--_popCount == 0)
                DeleteNodes(node);
            else if (node)
                ChainPendingNodeList(node);

            delete oldHead;
        }
        else
        {
            ChainPendingNodeList(oldHead);
            --_popCount;
        }
    }

    void ChainPendingNodeList(Node* first, Node* last)
    {
        last -> next = _pendingList;
        while(_pendingList.compare_exchange_weak(last-> next, first)==false) { }
    }

     void ChainPendingNodeList(Node* node)
    {
        Node* last = node;
        while(last -> next)
            last = last -> next;

        ChainPendingNodeList(node, last);
    }

    static void DeleteNodes(Node* node)
    {
        while(node)
        {
            Node* next = node -> next;
            delete node;
            node = next;
        }
    }

private:
    atomic<Node*>   _head;
    atomic<uint32>  _popCount = 0;
    atomic<Node*>   _pendingList;
}
```

## **2) Push**

```c++
template<typename T>
class LockFreeStack
{
    struct Node
    {
        Node (const T& value): data(value), next(nullptr)
        { 

        }

        T data;
        Node* next;
    };

public:
    void Push(const T& value)
    {
        Node* node = new Node(value);
        node-> next = _head;
        _head = node;
    }

private:
    atomic<Node*>   _head;
}
```

일반적인 Stack 구조는 위와 같이 Push를 구현한다. 새 노드를 만들고, 새 노드의 next를 기존 head로 연결하고, head를 새로 만든 노드로 교체한다. 멀티스레드에서도 새 노드를 만드는 것까지는 문제가 없다. new 자체는 힙에서 일어나지만 node 변수는 각 스레드의 스택에 들어있기 때문에 다른 스레드가 영향을 미칠 수 없기 때문이다. 같은 이유로 node의 next를 head로 바꾸는 것도 문제가 없다. 그러나 head는 내부적으로 LockFreeStack에 접근하는 모든 스레드들이 공유할 수 있는 전역 변수처럼 동작하기에 동기화 작업이 필요하다.

요약하자면 node를 만드는 것은 공유 데이터를 건드리지 않는 일, node의 next를 기존 head로 바꾸는 것은 공유 데이터를 읽기만 하는 일이므로 문제가 없다. 반면 기존 head를 node로 바꾸는 것은 공유 데이터에 쓰기를 하는 것이므로 동기화 작업이 필요하다. 

```c++
template<typename T>
class LockFreeStack
{
    struct Node
    {
        Node (const T& value): data(value), next(nullptr)
        { 

        }

        T data;
        Node* next;
    };

public:
    void Push(const T& value)
    {
        Node* node = new Node(value);
        node-> next = _head;
        while( _head.compare_exchange_weak(node->next, node) == false) { }
    }

private:
    atomic<Node*>   _head;
}
```

compare_exchange를 통해 위와 같이 동기화할 수 있다. _head.compare_exchange_weak(node->next, node) 내부에서 일어나는 일을 풀어쓰면 아래와 같다. 

```c++
if(_head == node->next)
{
    _head = node;
    return true;
}
else
{
    node->next = _head;
    return false;
}
```

_head가 node->next와 같다면, 즉 node->next = head 코드가 직전에 무사히 실행 되었고 그 사이에 다른 스레드의 개입이 없었다면 _head를 node로 바꾼다. 사이에 다른 스레드의 개입이 있었다면 node->next = _head를 다시 실행하고 while문을 돌려서 재시도 한다. 만약 스레드가 매우 많고 각 스레드들이 계속 push를 시도한다면 이곳의 while 문에서 오래 대기하게 될 수도 있다. 

## **3) Pop & Delete**

Pop은 head를 읽기, head->next를 읽기, head를 head->next로 바꿔주기, 기존 head의 데이터를 추출해서 반환하기, 기존 head 노드를 삭제하기 다섯 단계로 이루어진다. 우선 Push에서 했던 정도로만 동기화에 신경쓴 코드는 아래와 같다. 

```c++
template<typename T>
class LockFreeStack
{
public: 
    bool TryPop(T& value)
    {
        Node* oldHead = _head;

        while(oldHead && _head.compare_exchange_weak(oldHead, oldHead -> next) == false){ }
        if(oldHead == nullptr)
            return false;

        value = oldHead -> data;
        delete oldHead;
        return true;
    }

private:
    atomic<Node*>   _head;
}
```

다섯 단계를 그대로 따르되, head 변경 시 compare_exchange 함수를 사용하고, 또 oldHead가 다른 스레드에 의해 nullptr이 되었을 경우를 고려하여 그에 대한 예외처리를 해주고 있다. 그러나 이 코드는 미세한 경합으로 다른 스레드가 oldHead를 delete 한 후 다시 해당 포인터에 대해 delete 하게 될 가능성이 있다. 따라서 이에 대한 처리를 동기화 하기 위해 TryDelete를 분리해주도록 한다. 

TryDelete는 pop을 요청 받은 Count (popCount), 그리고 삭제 요청 받은 노드의 리스트 (pendingList)를 참조하여 처리한다. 

```c++
template<typename T>
class LockFreeStack
{
public: 

    bool TryPop(T& value)
    {
        ++_popCount;

        Node* oldHead = _head;
        while(oldHead && _head.compare_exchange_weak(oldHead, oldHead -> next) == false){ }

        if(oldHead == nullptr)
        {
            --_popCount;
            return false;
        }
        
        value = oldHead -> data;
        TryDelete(oldHead);
        return true;
    }

    void TryDelete(Node* oldHead)
    {
        if(_popCount == 1)
        {
            delete oldHead;
        }
        else
        {
            // pendingList에 추가
            --_popCount;
        }
    }

private:
    atomic<Node*>   _head;
    atomic<uint32>  _popCount = 0;
    atomic<Node*>   _pendingList;
}
```

popCount가 1이면 해당 노드의 pop을 요청한 게 현 스레드 하나인 것이므로 그대로 삭제를 시도한다. 반면 1이 아니라면 pendingList에 노드를 넣고 popCount만 줄이고 반환한다. 데이터 분리 -> count 체크 순서를 반드시 지켜주어야 동기화를 보장할 수 있다. 

```c++
template<typename T>
class LockFreeStack
{
public: 

    bool TryPop(T& value)
    {
        ++_popCount;

        Node* oldHead = _head;
        while(oldHead && _head.compare_exchange_weak(oldHead, oldHead -> next) == false){ }

        if(oldHead == nullptr)
        {
            --_popCount;
            return false;
        }
            
        value = oldHead -> data;
        TryDelete(oldHead);
        return true;
    }

    void TryDelete(Node* oldHead)
    {
        if(_popCount == 1)
        {
            Node* node = _pendingList.exchange(nullptr);

            if(--_popCount == 0)
                DeleteNodes(node);
            else if (node)
                // pendingList에 가져왔던 node list 포인터를 다시 넣기

            delete oldHead;
        }
        else
        {
            // pendingList 마지막에 oldHead만 추가
            --_popCount;
        }
    }

private:
    atomic<Node*>   _head;
    atomic<uint32>  _popCount = 0;
    atomic<Node*>   _pendingList;
}
```

이때 바로 delete를 실행하는 대신 위와 같이 pendingList의 첫번째 데이터를 node에 받아오고, pendingList는 nullptr로 밀어주는 작업을 수행한다. 이후 --popCount가 0이라면 그 사이 개입한 스레드가 없다는 의미이므로 그대로 Delete를 수행하면 된다. 반면 popCount가 0이 아니라면 사이에 어떤 스레드가 개입한 것이니 다시 pendingList에 넣어야 한다. 이때 popCount는 atomic 변수이므로 --_popCount는 sub부터 반환 작업까지 원자적으로 이루어진다. 

```c++
template<typename T>
class LockFreeStack
{
public: 

    bool TryPop(T& value)
    {
        ++_popCount;

        Node* oldHead = _head;
        while(oldHead && _head.compare_exchange_weak(oldHead, oldHead -> next) == false){ }

        if(oldHead == nullptr)
        {
            --_popCount;
            return false;
        }
            
        value = oldHead -> data;
        TryDelete(oldHead);
        return true;
    }

    void TryDelete(Node* oldHead)
    {
        if(_popCount == 1)
        {
            Node* node = _pendingList.exchange(nullptr);

            if(--_popCount == 0)
                DeleteNodes(node);
            else if (node)
                ChainPendingNodeList(node);

            delete oldHead;
        }
        else
        {
            ChainPendingNodeList(oldHead);
            --_popCount;
        }
    }

    static void DeleteNodes(Node* node)
    {
        while(node)
        {
            Node* next = node -> next;
            delete node;
            node = next;
        }
    }

    void ChainPendingNodeList(Node* first, Node* last)
    {
        last -> next = _pendingList;
        while(_pendingList.compare_exchange_weak(last-> next, first)==false) { }
    }

    void ChainPendingNodeList(Node* node)
    {
        Node* last = node;
        while(last -> next)
            last = last -> next;

        ChainPendingNodeList(node, last);
    }

private:
    atomic<Node*>   _head;
    atomic<uint32>  _popCount = 0;
    atomic<Node*>   _pendingList;
}
```

실질적인 DeleteNode 작업과 pendingList에 node를 추가하는 작업은 위와 같이 진행한다. 이때도 타 스레드의 개입으로 데이터가 오염될 수 있으니 compare_exchange를 이용해서 node 추가 작업을 실행한다. 

<br/>

# **2. Smart Pointer를 활용하는 방법**

```c++
template<typename T>
class LockFreeStack
{
    struct Node
    {
        Node (const T& value): data(make_shared<T>(value)), next(nullptr) { }
        shared_ptr<T> data;
        shared_ptr<Node> next;
    };

public:
    void Push(const T& value)
    {
        shared_ptr<Node> node = make_shared<Node>(value);
        node-> next = std::atomic_load(&_head);

        while(std::atomic_compare_exchange_weak(&_head, &node->next, node) == false) { }
    }

    shared_ptr<T> TryPop()
    {
        shared_ptr<Node> oldHead = std::atomic_load(&_head);

        while(oldHead && std::atomic_compare_exchange_weak(&_head, &oldHead, oldHead -> next) == false){ }

        if(oldHead == nullptr)
            return shared_ptr<T>();
            
        return oldHead -> data;
    }

private:
    shared_ptr<Node>   _head;
}
```

Smart Pointer를 사용할 때 Atomic 하게 동작해야 하는 부분을 따로 처리해주는 것을 잊지 말아야 한다. 또, compare_exchange의 경우 shared_ptr을 받지 못하기 때문에 atomic_compare_exchange로 대체해주어야 한다. _head를 oldHead로 가져올 때도 atomic 하게 가져오기 위해 atomic_load를 사용해야 한다. 이 경우 reference 관리 및 delete 처리를 알아서 해주기 때문에 DeleteNode, ChainPendingNodeList를 구현하지 않아도 된다. 

단, 이 경우 shared_ptr 자체가 내부적으로 lock을 사용하기 때문에 명시적인 lock을 쓰지 않았을 뿐 lock-free 하게 구현했다고 할 수 없다. 따라서 Smart Pointer를 통해 lock-free stack을 만들고 싶다면 reference count 기능이 있는 pointer를 직접 구현해주어야 한다. 그 예제는 아래와 같다. 

```c++
template<typename T>
class LockFreeStack
{
    struct Node;
    struct CountedNodePtr
    {
        int32 externalCount = 0;
        Node* ptr = nullptr;
    };
    
    struct Node
    {
        Node (const T& value): data(make_shared<T>(value)), next(nullptr) { }
        shared_ptr<T> data;
        atomic<int32> internalCount = 0;
        CountedNodePtr next;
    };

public:
    void Push(const T& value)
    {
        CountedNodePtr node;
        node = new Node (value);
        node.externalCount = 1;

        node.ptr -> next = _head;
        while(_head.atomic_compare_exchange_weak(node,ptr->next, node) == false) { }
    }

    shared_ptr<T> TryPop()
    {
        CountedNodePtr oldHead = _head;

        while (true)
        {
            IncreaseHeadCount(oldHead);
            Node* ptr = oldHead.ptr;

            if(ptr == nullptr)
                return shared_ptr<T>();

            if(_head.compare_exchange_weak(oldHead, ptr -> next) == false)
            { 
                shared_ptr<T> res;
                res.swap(ptr->data);

                const int32 countIncrease = oldHead.externalCount - 2;
                if(ptr->internalCount.fetch_add(countIncrease) == - countIncrease)
                    delete ptr;

                return res;
            }
            else if(ptr->internalCount.fetch_sub(1) == 1)
            {
                delete ptr;
            }
        }

        return oldHead -> data;
    }

    void IncreaseHeadCount(CountedNodePtr& oldCounter)
    {
        while(true)
        {
            CountedNodePtr newCounter = oldCounter;
            newCounter.externalCount++

            if(_head.atomic_compare_exchange_weak(oldCounter, newCounter))
            { 
                oldCounter.externalCount = newCounter.externalCount;
                break;
            }
        }
    }

private:
    atomic<CountedNodePtr>   _head;
}
```

<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
