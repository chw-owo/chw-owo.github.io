---
title: Multithread 08) Lock-Free Queue
categories: RookissGameServer
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part4:  게임 서버> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Lock-Free Queue**

우선 멀티스레드를 고려하지 않고 큐를 구현하면 다음과 같다. 단, 이후 분리를 원활하게 하기 위해 dummy node를 만들어서 head, tail이 이를 가리키게끔 했다. 

**LockFreeQueue.h**

```c++
template<typename T>
class LockFreeQueue
{
    struct Node
    {
        shared_ptr<T> data;
        Node* next = nullptr;
    };

public:
    LockFreeQueue() : _head(new Node), _tail(_head)
    {
        
    }
    LockFreeQueue(const LockFreeQueue&) = delete;
    LockFreeQueue& operator=(const LockFreeQueue&) = delete;

public:
    void Push(const T& value)
    {
        shared_ptr<T> newData = make_shared<T>(value);
        Node* dummy = new Node();
        Node* oldTail = _tail;
        oldTail->data.swap(newData);
        oldTail->next = dummy;   
        _tail = dummy;
    }

    shared_ptr<T> TryPop()
    {
        Node* oldHead = PopHead();
        if(oldHead == nullptr)
            return shared_ptr<T>();

        shared_ptr<T> res(oldHead->data);
        delete oldHead;
        return res;
    }

private:
    Node* PopHead()
    {
        Node* oldHead = _head;
        if(oldHead == _tail)
            return nullptr;
        
        _head = oldHead->next;
        return oldHead;
    }

private:
    Node* _head = nullptr;
    Node* _tail = nullptr;

};
```
head에서만 경합이 일어났던 stack과 달리 queue는 push 시에 tail에, pop 시에 head에 좌우로 경합이 일어난다. 따라서 stack의 pop에서 해주었던 것과 같은 동기화 처리가 push에서도 필요하다. 이를 위해 internalCount와 externalCount를 관리해주는 NodeCounter과 CountedNodePtr을 만들고, Node의 포인터 대신 CountedNodePtr 단위로 Node를 관리하도록 한다. 

```c++
template<typename T>
class LockFreeQueue
{
    struct Node;
    struct CountedNodePtr
    {
        int32 externalCount;
        Node* ptr = nullptr;
    }
    struct NodeCounter
    {
        uint32 internalCount : 30;
        uint32 externalCountRemaining : 2;
    };

    struct Node
    {
        Node()
        {
            NodeCounter newCount;
            newCount.internalCount = 0;
            newCount.externalCountRemaining = 2;
            count.store(newCount);

            next.ptr = nullptr;
            next.externalCount = 0;
        }

        void ReleaseRef()
        {
            // TO-DO
        }

        atomic<T*> data;
        atomic<NodeCounter> count;
        CountedNodePtr next;
    };

public:
    LockFreeQueue() : _head(new Node), _tail(_head)
    {
        
    }
    LockFreeQueue(const LockFreeQueue&) = delete;
    LockFreeQueue& operator=(const LockFreeQueue&) = delete;

public:
    void Push(const T& value)
    {
        shared_ptr<T> newData = make_shared<T>(value);
        Node* dummy = new Node();
        Node* oldTail = _tail;
        oldTail->data.swap(newData);
        oldTail->next = dummy;   
        _tail = dummy;
    }

    shared_ptr<T> TryPop()
    {
        Node* oldHead = PopHead();
        if(oldHead == nullptr)
            return shared_ptr<T>();

        shared_ptr<T> res(oldHead->data);
        delete oldHead;
        return res;
    }

private:
    Node* PopHead()
    {
        Node* oldHead = _head;
        if(oldHead == _tail)
            return nullptr;
        
        _head = oldHead->next;
        return oldHead;
    }

private:
    atomic<CountedNodePtr> _head;
    atomic<CountedNodePtr> _tail;
};
```

이후 Push, Pop 코드도 멀티스레드에 적합하게 수정해야 한다. 

```c++
template<typename T>
class LockFreeQueue
{
    struct Node;
    struct CountedNodePtr
    {
        int32 externalCount;
        Node* ptr = nullptr;
    }
    struct NodeCounter
    {
        uint32 internalCount : 30;
        uint32 externalCountRemaining : 2;
    };

    struct Node
    {
        Node()
        {
            NodeCounter newCount;
            newCount.internalCount = 0;
            newCount.externalCountRemaining = 2;
            count.store(newCount);

            next.ptr = nullptr;
            next.externalCount = 0;
        }

        void ReleaseRef()
        {
            // TO-DO
        }

        atomic<T*> data;
        atomic<NodeCounter> count;
        CountedNodePtr next;
    };

public:
    LockFreeQueue() : _head(new Node), _tail(_head)
    {
        
    }
    LockFreeQueue(const LockFreeQueue&) = delete;
    LockFreeQueue& operator=(const LockFreeQueue&) = delete;

public:
    void Push(const T& value)
    {
        unique_ptr<T> newData = make_unique<T>(value);

        CountedNodePtr dummy;
        dummy.ptr = new Node();
        dummy.externalCount = 1;

        CountedNodePtr oldTail = _tail.load(); // ptr == nullptr;

        while(true)
        {
            // 참조권 획득: externalCount를 +1 한 스레드가 참조권을 차지
            IncreaseExternalCount(_tail, oldTail);

            // 소유권 획득: data를 먼저 교환한 스레드가 소유권을 차지
            T* oldData = nullptr;
            if(oldTail.ptr->data.compare_exchange_strong(oldData, newData.get()))
            {
                oldTail.ptr->next = dummy;
                oldTail = _tail.exchange(dummy);
                FreeExternalCount(oldTail);
                newData.release(); // unique_ptr의 소유권을 포기
                break;
            }

            oldTail.ptr->ReleaseRef();
        }
    }

    shared_ptr<T> TryPop()
    {
        CountedNodePtr oldHead = _head.load();

		while (true)
		{
			// 참조권 획득: externalCount를 +1 한 스레드가 참조권을 차지
			IncreaseExternalCount(_head, oldHead);

			Node* ptr = oldHead.ptr;
			if (ptr == _tail.load().ptr)
			{
				ptr->ReleaseRef();
				return shared_ptr<T>();
			}

			// 소유권 획득: head = ptr->next
			if (_head.compare_exchange_strong(oldHead, ptr->next))
			{
				T* res = ptr->data.load(); // exchange(nullptr); 로 하면 버그
				FreeExternalCount(oldHead);
				return shared_ptr<T>(res);
			}

			ptr->ReleaseRef();
		}
    }

private:
    static void IncreaseExternalCount(atomic<CountedNodePtr>& counter, CountedNodePtr& oldCounter)
    {
        while(true)
        {
            CountedNodePtr newCounter = oldCounter;
            newCounter.externalCount++;

            if(counter.compare_exchange_strong(oldCounter, newCounter))
            {
                oldCounter.externalCount = newCounter.externalCount;
                break;
            }
        }
    }

    static void FreeExternalCount(CountedNodePtr& oldNodePtr)
    {
        Node* ptr = oldNodePtr.ptr;
        const int32 countIncrease = oldNodePtr.externalCount - 2;

        NodeCounter oldCounter = ptr->count.load();
        
        while(true)
        {
            NodeCounter newCounter = oldCounter;
            newCounter.externalCountRemaining--;
            newCounter.internalCount += countIncrease;

            if(ptr->count.compare_exchange_strong(oldCounter, newCounter))
            {
                if(newCounter.internalCount == 0 && newCounter.externalCountRemaining)
            }
        }
    }

private:
    atomic<CountedNodePtr> _head;
    atomic<CountedNodePtr> _tail;
};
```

이후 소유권 획득에 실패한 노드에 대해 internalCount 감소를 해주어야 한다. 이를 ReleaseRef 함수를 통해 처리한다. 

**완성된 코드**

```c++
template<typename T>
class LockFreeQueue
{
	struct Node;

	struct CountedNodePtr
	{
		int32 externalCount; // 참조권
		Node* ptr = nullptr;
	};

	struct NodeCounter
	{
		uint32 internalCount : 30; // 참조권 반환 관련
		uint32 externalCountRemaining : 2; // Push & Pop 다중 참조권 관련
	};

	struct Node
	{
		Node()
		{
			NodeCounter newCount;
			newCount.internalCount = 0;
			newCount.externalCountRemaining = 2;
			count.store(newCount);

			next.ptr = nullptr;
			next.externalCount = 0;
		}

		void ReleaseRef()
		{
			NodeCounter oldCounter = count.load();

			while (true)
			{
				NodeCounter newCounter = oldCounter;
				newCounter.internalCount--;

				// 끼어들 수 있음
				if (count.compare_exchange_strong(oldCounter, newCounter))
				{
					if (newCounter.internalCount == 0 && newCounter.externalCountRemaining == 0)
						delete this;

					break;
				}
			}
		}

		atomic<T*> data;
		atomic<NodeCounter> count;
		CountedNodePtr next;
	};

public:
	LockFreeQueue()
	{
		CountedNodePtr node;
		node.ptr = new Node;
		node.externalCount = 1;

		_head.store(node);
		_tail.store(node);
	}

	LockFreeQueue(const LockFreeQueue&) = delete;
	LockFreeQueue& operator=(const LockFreeQueue&) = delete;
	
	void Push(const T& value)
	{
		unique_ptr<T> newData = make_unique<T>(value);

		CountedNodePtr dummy;
		dummy.ptr = new Node;
		dummy.externalCount = 1;

		CountedNodePtr oldTail = _tail.load(); // ptr = nullptr

		while (true)
		{
			// 참조권 획득
			IncreaseExternalCount(_tail, oldTail);

			// 소유권 획득
			T* oldData = nullptr;
			if (oldTail.ptr->data.compare_exchange_strong(oldData, newData.get()))
			{
				oldTail.ptr->next = dummy;
				oldTail = _tail.exchange(dummy);
				FreeExternalCount(oldTail);
				newData.release(); // 데이터에 대한 unique_ptr의 소유권 포기
				break;
			}

			// 소유권 경쟁 패배
			oldTail.ptr->ReleaseRef();
		}
	}

	shared_ptr<T> TryPop()
	{
		CountedNodePtr oldHead = _head.load();

		while (true)
		{
			// 참조권 획득

			Node* ptr = oldHead.ptr;
			if (ptr == _tail.load().ptr)
			{
				ptr->ReleaseRef();
				return shared_ptr<T>();
			}

			// 소유권 획득
			if (_head.compare_exchange_strong(oldHead, ptr->next))
			{
				T* res = ptr->data.load(); 
				FreeExternalCount(oldHead);
				return shared_ptr<T>(res);
			}

			ptr->ReleaseRef();
		}
	}

private:
	static void IncreaseExternalCount(atomic<CountedNodePtr>& counter, CountedNodePtr& oldCounter)
	{
		while (true)
		{
			CountedNodePtr newCounter = oldCounter;
			newCounter.externalCount++;

			if (counter.compare_exchange_strong(oldCounter, newCounter))
			{
				oldCounter.externalCount = newCounter.externalCount;
				break;
			}
		}
	}

	static void FreeExternalCount(CountedNodePtr& oldNodePtr)
	{
		Node* ptr = oldNodePtr.ptr;
		const int32 countIncrease = oldNodePtr.externalCount - 2;

		NodeCounter oldCounter = ptr->count.load();

		while (true)
		{
			NodeCounter newCounter = oldCounter;
			newCounter.externalCountRemaining--; // TODO
			newCounter.internalCount += countIncrease;

			if (ptr->count.compare_exchange_strong(oldCounter, newCounter))
			{
				if (newCounter.internalCount == 0 && newCounter.externalCountRemaining == 0)
					delete ptr;

				break;
			}
		}
	}

private:
	atomic<CountedNodePtr> _head;
	atomic<CountedNodePtr> _tail;
};
```
<br/> 

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part4: 게임 서버, Rookiss
