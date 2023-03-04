---
title: Procademy, Assignment 14) Linked List Stl
categories: ProcademyAssignment
tags: 
toc: true
toc_sticky: true
---

# **과제**

아래 형식에 맞추어 stl처럼 사용할 수 있는 링크드 리스트 자료구조를 구현한다. 

```c++
template <typename T>

class CList
{	
public:		
	struct Node	
	{	
		T _data;
		Node *_Prev;
		Node *_Next;	
	};

	class iterator	
	{
	private:
		Node *_node;
	public:
		iterator (Node *node = nullptr);
		iterator operator ++(int);
		iterator& operator++();
		iterator operator --(int);
		iterator& operator--();
		T& operator *();
		bool operator ==(const iterator& other);
		bool operator !=(const iterator& other);
	};

public:
	CList();
	~CList();
	
	iterator begin ();
	iterator end ();

	void push_front(T data);
	void push_back (T data);
	void pop_front();
	void pop_back();
	void clear();
	int size() { return _size; };
	bool empty() { };

	iterator erase(iterator iter);
	void remove(T Data);

private:
	int _size = 0;
	Node _head;
	Node _tail;
};
```

<br/>

# **과제 코드**

**main.cpp**

```c++
#include "LinkedList.cpp"
#include <stdio.h>

class Player
{
public:
	Player(int ID) : _ID(ID) {}
	int GetID() { return _ID; }

private:
	int _ID;
};


int main()
{
	List<Player*> ListPlayer;
	ListPlayer.push_back(new Player(0));
	ListPlayer.push_back(new Player(1));
	ListPlayer.push_back(new Player(2));

	List<Player*>::iterator iter;
	for (iter = ListPlayer.begin(); iter != ListPlayer.end(); )
	{
		printf("%d\n", (*iter)->GetID());
		if ((*iter)->GetID() == 1)
		{
			iter = ListPlayer.erase(iter);
		}
		else
		{
			++iter;
		}
	}
	printf("size: %d\n", ListPlayer.size());
	printf("empty: %d\n", ListPlayer.empty());
	
	printf("======================================\n");

	ListPlayer.pop_front();
	iter = ListPlayer.begin();
	for (; iter != ListPlayer.end(); ++iter)
	{
		printf("%d\n", (*iter)->GetID());
	}
	printf("size: %d\n", ListPlayer.size());
	printf("empty: %d\n", ListPlayer.empty());

	printf("======================================\n");

	ListPlayer.clear();
	iter = ListPlayer.begin();
	for (; iter != ListPlayer.end(); ++iter)
	{
		printf("%d\n", (*iter)->GetID());
	}

	printf("size: %d\n", ListPlayer.size());
	printf("empty: %d\n", ListPlayer.empty());
}
```

**LinkedList.cpp**
```c++
#pragma once
template <typename T>
class List
{
public:
	struct Node
	{
		T _data;
		Node* _Prev;
		Node* _Next;
	};

	class iterator
	{
	private:
		Node* _node;
	public:
		friend class List;
	public:
		iterator(Node* node = nullptr) : _node(node)
		{

		}
		iterator operator ++(int)
		{
			iterator origin = *this;
			this->_node = _node->_Next;
			return origin;
		}
		iterator& operator++()
		{
			this->_node = _node->_Next;
			return *this;
		}
		iterator operator --(int)
		{
			iterator origin = *this;
			this->_node = _node->_Prev;
			return origin;
		}
		iterator& operator--()
		{
			this->_node = _node->_Prev;
			return *this;
		}
		T& operator *()
		{
			return this->_node->_data;
		}
		bool operator ==(const iterator& other)
		{
			return (this->_node == other._node);
		}
		bool operator !=(const iterator& other)
		{
			return (this->_node != other._node);
		}
	};

public:
	List()
	{
		_head._Prev = nullptr;
		_head._Next = &_tail;
		_tail._Prev = &_head;
		_tail._Next = nullptr;
	}
	~List()
	{
		clear();
	}

	iterator begin()
	{ 
		return iterator(_head._Next);
	}
	iterator end()
	{
		return iterator(&_tail);
	}

	void push_front(T data)
	{
		Node* node = new Node;
		node->_data = data;
		node->_Prev = &_head;
		node->_Next = _head._Next;
		(_head._Next)->_Prev = node;
		_head._Next = node;
		_size++;
	}
	void push_back(T data)
	{
		Node* node = new Node;
		node->_data = data;
		node->_Prev = _tail._Prev;
		node->_Next = &_tail;
		(_tail._Prev)->_Next = node;
		_tail._Prev = node;
		_size++;
	}
	void pop_front()
	{
		Node* nodePtr = _head._Next;
		_head._Next = nodePtr->_Next;
		(_head._Next)->_Prev = &_head;
		delete(nodePtr);
		_size--;
	}
	void pop_back()
	{
		Node* nodePtr = _tail._Prev;
		_tail._Prev = nodePtr->_Prev;
		(_tail._Prev)->_Next = &_tail;
		delete(nodePtr);
		_size--;
	}
	void clear()
	{
		Node* nodePtr = _head._Next;
		Node* prevNodePtr;

		while (nodePtr != &_tail)
		{
			prevNodePtr = nodePtr;
			nodePtr = prevNodePtr->_Next;
			delete(prevNodePtr);
		}

		_head._Next = &_tail;
		_tail._Prev = &_head;
		_size = 0;
	}

	int size()
	{
		return _size;
	}
	bool empty()
	{
		return (_size == 0);
	}

	iterator erase(iterator iter)
	{
		Node* nodePtr = iter._node->_Next;
		(iter._node->_Next)->_Prev = iter._node->_Prev;
		(iter._node->_Prev)->_Next = iter._node->_Next;
		delete(iter._node);
		_size--;

		iter._node = nodePtr;
		return iter;
	}
	void remove(T Data)
	{
		List<T>::iterator iter;
		for (iter = begin(); iter != end(); )
		{
			if ((*iter) == Data)
			{
				iter = erase(iter);
			}
			else
			{
				++iter;
			}
		}
	}

private:
	int _size = 0;
	Node _head;
	Node _tail;
};
```