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
		Node* _prev;
		Node* _next;
	};

	class iterator
	{
		friend class List;
	private:
		Node* _node;
	public:
		iterator(Node* node = nullptr) : _node(node)
		{

		}
		const iterator operator ++(int)
		{
			iterator origin = *this;
			_node = _node->_next;
			return origin;
		}
		iterator& operator++()
		{
			_node = _node->_next;
			return *this;
		}
		const iterator operator --(int)
		{
			iterator origin = *this;
			_node = _node->_prev;
			return origin;
		}
		iterator& operator--()
		{
			_node = _node->_prev;
			return *this;
		}
		T& operator *()
		{
			return _node->_data;
		}
		bool operator == (const iterator& other)
		{
			return (_node == other._node);
		}
		bool operator !=(const iterator& other)
		{
			return (_node != other._node);
		}
	};

public:
	List()
	{
		_head._prev = nullptr;
		_head._next = &_tail;
		_tail._prev = &_head;
		_tail._next = nullptr;
	}
	~List()
	{
		clear();
	}

	iterator begin()
	{ 
		return iterator(_head._next);
	}
	iterator end()
	{
		return iterator(&_tail);
	}

	void push_front(T data)
	{
		Node* node = new Node;
		node->_data = data;
		node->_prev = &_head;
		node->_next = _head._next;
		(_head._next)->_prev = node;
		_head._next = node;
		_size++;
	}
	void push_back(T data)
	{
		Node* node = new Node;
		node->_data = data;
		node->_prev = _tail._prev;
		node->_next = &_tail;
		(_tail._prev)->_next = node;
		_tail._prev = node;
		_size++;
	}
	void pop_front()
	{
		Node* nodePtr = _head._next;
		_head._next = nodePtr->_next;
		(_head._next)->_prev = &_head;
		delete(nodePtr);
		_size--;
	}
	void pop_back()
	{
		Node* nodePtr = _tail._prev;
		_tail._prev = nodePtr->_prev;
		(_tail._prev)->_next = &_tail;
		delete(nodePtr);
		_size--;
	}
	void clear()
	{
		Node* nodePtr = _head._next;
		Node* prevNodePtr;

		while (nodePtr != &_tail)
		{
			prevNodePtr = nodePtr;
			nodePtr = prevNodePtr->_next;
			delete(prevNodePtr);
		}

		_head._next = &_tail;
		_tail._prev = &_head;
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
		Node* nodePtr = iter._node->_next;
		(iter._node->_next)->_prev = iter._node->_prev;
		(iter._node->_prev)->_next = iter._node->_next;
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