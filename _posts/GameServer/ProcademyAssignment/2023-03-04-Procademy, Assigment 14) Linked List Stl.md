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
		iterator (Node *node = nullptr)
		{
			//인자로 들어온 Node 포인터를 저장
		}

		iterator operator ++(int) 
		{
			//현재 노드를 다음 노드로 이동
		}
    		
		iterator& operator++() 
		{
       			 return *this;
    		}

		iterator operator --(int) 
		{
		
		}
    		
		iterator& operator--() 
		{
       			 return *this;
    		}

		T& operator *() 
		{
			//현재 노드의 데이터를 뽑음
		}
		bool operator ==(const iterator& other)
		{
		}
		bool operator !=(const iterator& other) 
		{
		}
	};

public:
	CList();
	~CList();
	
	iterator begin ()
	{
		//첫번째 데이터 노드를 가리키는 이터레이터 리턴
	}
	iterator end ()
	{
		//Tail 노드를 가리키는 (데이터가 없는 진짜 더미 끝 노드) 이터레이터를 리턴
		//또는 끝으로 인지할 수 있는 이터레이터를 리턴
	}

	void push_front(T data);
	void push_back (T data);
	void pop_front();
	void pop_back();
	void clear();
	int size() { return _size; };
	bool empty() { };



	iterator erase(iterator iter) 
	// 이터레이터의 그 노드를 지움.
	// 그리고 지운 노드의 다음 노드를 카리키는 이터레이터 리턴

	void remove(T Data)
	{
		CList<int>::iterator iter;
		for ( iter = ListInt.begin(); iter != ListInt.end(); ++iter )
		{
			if ( *iter == Data )
				erase(iter);
		}
	}

private:
	int _size = 0;
	Node _head;
	Node _tail;
};

//////////// 순회 샘플 코드 /////////////

CList<int> ListInt;


CList<int>::iterator iter;
for ( iter = ListInt.begin(); iter != ListInt.end(); ++iter )
{
	printf("%d", *iter);
}

////////// 객체 삭제 샘플 코드 ////////////

list.begin()   // 첫 요소의 iter
list.end()     // 더미 마지막 iter 


CList<CPlayer *> ListPlayer;
ListPlayer.push_back(new CPlayer);
ListPlayer.push_back(new CPlayer);


for ( CList<CPlayer *>::iterator iter = List.begin(); iter != List.end(); ++iter )
{
	CPlayer *p = *iter;
}

CList<CPlayer *>::iterator iter;

for ( iter = List.begin(); iter != List.end(); )
{
	data = *iter;

	if ( (*iter)->GetID() == /*삭제대상*/ )
	{
		delete *iter;
		iter = List.erase(iter);
	}
	else
	{
		++iter;
	}
}
```

## **new, delete 오버로딩 문법**

```c++
void * operator new (size_t size, char *File, int Line)
{
}

void * operator new[] (size_t size, char *File, int Line)
{
	ptr = malloc(size);
	..
	return ptr;
}

void operator delete (void * p, char *File, int Line)
{
}
void operator delete[] (void * p, char *File, int Line)
{
}

// 실제로 사용할 delete	
void operator delete (void * p)
{
}
void operator delete[] (void * p)
{
}
```
<br/>

# **과제 코드**
