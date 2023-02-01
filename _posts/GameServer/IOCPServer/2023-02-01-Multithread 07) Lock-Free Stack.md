---
title: Multithread 07) Lock-Free Stack
categories: IOCPServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **0. Lock-Free 개요**

Lock을 사용하지 않고도 멀티스레드 환경에서 동기화가 이루어질 수 있는 Stack과 Queue를 구현한다. Lock을 명시적으로 사용하지는 않는다고 해도 compare_exchange_weak등과 같은 동기화 작업을 해주기 때문에 여러 스레드에서 동시에 push 하는 경우 lock-based에서와 마찬가지로 대기 지연 시간이 생기며, 성능 향상이 눈에 띄게 크게 일어나지는 않는다. 

실제로 멀티스레드 환경에서 사용가능한 Lock-free 자료구조를 구현하는 것에는 여러 알고리즘이 있으나, 여기서는 가장 단순하고 직관적인 방법들로 구현이 되었다. 

<br/>

# **1. Lock-Free Stack** 

## **1) 참조 Count를 직접 활용하는 방법**

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

## **2) Smart Pointer를 활용하는 방법**


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
Smart Pointer를 사용할 때 Atomic 하게 동작해야 하는 부분을 따로 처리해주는 것을 잊지 말아야 한다. 그러나 이 경우 shared_ptr 자체가 lock-free 하게 동작하지 않기 때문에, 명시적인 lock을 쓰지 않았을 뿐 lock-free 하게 구현했다고 할 수 없다. 따라서 Smart Pointer를 통해 lock-free stack을 만들고 싶다면 reference count 기능이 있는 pointer를 직접 구현해주어야 한다. 그 예제는 아래와 같다. 

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
