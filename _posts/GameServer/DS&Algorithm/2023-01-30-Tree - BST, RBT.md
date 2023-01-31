---
title: Tree - BST, RBT
categories: DS&Algorithm
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part3: 자료구조와 알고리즘> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. BST,Binary Search Tree**

## **1) 정의**
트리 중 각 노드가 최대 두개의 자식 노드를 가지는 트리를 두고 이진 트리라 부르고, 그 중에서도 루트를 중심으로 현재 값보다 작은 경우 왼쪽에, 큰 경우 오른쪽에 배치하여 탐색을 쉽게끔 만든 구조는 이진 탐색 트리라고 부른다. 이를 간단하게 구현해보면 아래와 같다. 

## **2) Insert**

```c++
struct Node
{
    Node*   parent = nullptr;
    Node*   left = nullptr;
    Node*   right = nullptr;
    int     key = {};
}

class BST
{
public:
    Node*   Search(Node* node, int key);
    Node*   Min(Node* node);
    Node*   Max(Node* node);
    Node* Next(Node* node);
    void Insert(int key);
    void Delete(int key);
    void Replace(Node* u, Node* v);

private:
    Node* _root = nullptr;
}

void BST::Insert(int key)
{
    Node* newNode = new Node();
    newNode -> key = key;

    if(_root == nullptr)
    {
        _root = newNode;
        return;
    }

    Node* node = _root;
    Node* parent = nullptr;

    while(node)
    {
        // Key를 바탕으로 어느 node를 
        // newNode의 parent로 삼아야 하는지 탐색
        parent = node;
        if(key < node -> key)
            node = node -> left;
        else
            node = node -> right;
    }

    newNode -> parent = parent;

    if(key < parent->key)
        parent -> left = newNode;
    else
        parent -> right = newNode;
}

void BST::Delete(int key)
{
    Node* deleteNode = Search(_root, key);
    Delete(deleteNode);
}

void BST::Delete(Node* node)
{
    if(node == nullptr)
        return;

    if(node->left == nullptr)
        Replace(node, node -> right);
    else if(node->right == nullptr)
        Replace(node, node -> left);  
    else
    {
        Node* next = Next(node);
        node -> key = next -> key;
        Delete(next);
    } 
}

void BST::Replace(Node* u, Node* v)
{
    if (u->parent == nullptr)
        _root = v;
    else if (u == u -> parent -> left)
        u -> parent -> left = v;
    else
        u -> parent -> right = v;

    if (v)
        v -> parent = u -> parent;
    
    delete u;
}

Node* BST::Search(Node* node, int key)
{
    if( node == nullptr || key == node -> key)
        return node;
    
    if(key < node -> key)
        return Search(node-> left, key);
    else
        return Search(node-> right, key);
}

Node* BST::Min(Node* node)
{
    while(node->left)
        node = node->left;
    return node;
}

Node* BST::Max(Node* node)
{
    while(node->right)
        node = node->right;
    return node;
}

Node* BST::Next(Node* node)
{
    if(node->right)
        return Min(node->right);

    Node* parent = node -> parent;
    while(parent && node == parent -> right)
    {
        node = parent;
        parent = parent -> parent;
    }
    return parent;
}

```

## **3) Traverse**

**Pre-order Traverse, 전위 순회**: root를 제일 먼저 방문한 뒤, left, right 순으로 순회한다. 

**In-order Traverse, 중위 순회**: left쪽 하위 트리를 방문한 후 root를 방문하고, right쪽 하위 트리를 순회한다.

**Post-order Traverse, 후위 순회**: 하위 트리를 left, right순으로 모두 방문한 후 root를 순회한다. 

**Level-order Traverse, 층별 순회**: 위쪽 node들부터 아래 방향으로 순회한다. 


<br/>

# **2. RBT, Red Black Tree**

이진 탐색 트리의 경우 데이터가 갱신되다보면 트리가 한쪽으로 기울어질 위험성이 있는데, 이렇게 되면 검색의 효율성이 떨어지기 때문에 트리 재배치를 통해 균형을 유지해야 한다. 레드 블랙 트리는 균형을 맞추기 위한 방법 중 하나를 적용한 이진트리이다.

![image](https://user-images.githubusercontent.com/96677719/215699238-26e0800f-643c-4ef3-83e0-113e983287e6.png)

레드 블랙 트리에서 모든 노드는 레드 혹은 블랙 중 하나의 색을 가지며, 모든 리프 노드는 블랙이 되고 레드 노드의 자식 양쪽은 언제나 블랙이 되어야 한다. 즉, 레드 노드는 연달아 나타날 수 없으며 블랙 노드만이 레드 노드의 부모 노드가 된다. 루트 노드는 블랙이며, 어떤 노드이든 그에 속한 하위 리프 노드에 도달하는 모든 경로에는 리프 노드를 제외하고 같은 개수의 블랙 노드가 있어야 한다. 

위의 이진 탐색 트리를 활용하여 이를 구현한 예제는 아래와 같다. 

```c++
enum Color
{
    Red = 0,
    Black = 1,
};

struct Node
{
    Node*   parent = nullptr;
    Node*   left = nullptr;
    Node*   right = nullptr;
    int     key = {};
    Color   color = Color::Black;
}

class BST
{
public:
    BST();
    ~BST();

    Node*   Min(Node* node);
    Node*   Max(Node* node);
    Node*   Next(Node* node);
    void    Insert(int key);

    void    Print() { Print(_root, 10, 0); }
    void    Print(Node* node, int x, int y);
    void    InsertFixup(Node* node);
    void    LeftRotate(Node* node);
    void    RightRotate(Node* node);


private:
    Node* _root = nullptr;
    Node* _nil = nullptr;
}

enum class ConsoleColor
{
    BLUE = FOREGROUND_BLUE,
    RED = FOREGROUND_RED
    WHITE = FOREGROUND_RED | FOREGROUND_GREEN | FOREGROUND_BLUE;
};

void SetCursorColor(ConsoleColor color)
{
    HANDLE output = ::GetStdHandles(STD_OUTPUT_HANDLE);
    ::SetConsoleTextAttribute(output, static_cast<SHORT>(color));
}

BST::BST()
{
    _nil = new Node();
    _root = _nil;
}

BST::~BST()
{
    delete _nil;
}

void BST::Print(Node* node, int x, int y)
{
    if(node == nullptr)
        return;

    SetCursorPosition(x, y);
    if(node->color == Color::Black)
        SetCursorColor(ConsoleColor::BLUE);
    else
        SetCursorColor(ConsoleColor::RED);

    cout << node -> key;
    Print(node->left, x - (5/(y+1)), y+1);
    Print(node->right, x + (5/(y+1)), y+1);

    SetCursorColor(ConsoleColor::WHITE);
}

void BST::Insert(int key)
{
    Node* newNode = new Node();
    newNode -> key = key;
    Node* node = _root;
    Node* parent = nullptr;

    while(node != _nil)
    {
        // Key를 바탕으로 어느 node를 
        // newNode의 parent로 삼아야 하는지 탐색
        parent = node;
        if(key < node -> key)
            node = node -> left;
        else
            node = node -> right;
    }

    newNode -> parent = parent;

    if(parent == _nil)
        _root = newNode;
    else if(key < parent->key)
        parent -> left = newNode;
    else
        parent -> right = newNode;

    newNode -> left = _nil;
    newNode -> right = _nil;
    newNode -> color = Color::Red;

    InsertFixup(newNode);
}

void BST::InsertFixup(Node* node)
{
    while(node -> parent -> color == Color::Red)
    {
        if(node -> parent == node -> parent -> parent -> left)
        {
            Node* uncle = node -> parent -> parent -> right;
            
            // 색이 잘못되었을 때 노드의 색을 바꾸어준다.
            if(uncle -> color == Color::Red)
            {
                node -> parent -> color = Color::Black;
                uncle -> color = Color::Black;
                node -> parent -> parent -> color = Color::Red;
                node = node -> parent -> parent;
            }
            else
            {
                // 마름모꼴을 이루고 있을 때 
                // 회전을 위해 위치 이동
                if(node == node -> parent -> right)
                {
                    node = node->parent;
                    LeftRotate(node);
                }

                // 자식이 일직선인 상태에서  
                // 회전으로 루트를 바꾸어준다.
                node -> parent -> color = Color::Black;
                node -> parent -> parent -> color = Color::Red;
                RightRotate(node -> parent -> parent);

            }
        }
        else
        {
            Node* uncle = node -> parent -> parent -> left;
            
            // 색이 잘못되었을 때 노드의 색을 바꾸어준다.
            if(uncle -> color == Color::Red)
            {
                node -> parent -> color = Color::Black;
                uncle -> color = Color::Black;
                node -> parent -> parent -> color = Color::Red;
                node = node -> parent -> parent;
            }
            else
            {
                // 마름모꼴을 이루고 있을 때 
                // 회전을 위해 위치 이동
                if(node == node -> parent -> left)
                {
                    node = node->parent;
                    RightRotate(node);
                }

                // 자식이 일직선인 상태에서  
                // 회전으로 루트를 바꾸어준다.
                node -> parent -> color = Color::Black;
                node -> parent -> parent -> color = Color::Red;
                LeftRotate(node -> parent -> parent);

            }
        }
    }

    _root -> cokor = Color::Black;
}

void BST::LeftRotate(Node* x)
{
    Node* y = x -> right;
    x -> right = y -> left;
    if(y -> left != _nil)
        y -> left -> parent = x;
    y -> parent = x -> parent;

    if(x->parent == _nil)
        _root = y;
    else if (x == x -> parent -> left)
        x -> parent -> left = y;
    else
        x -> parent -> right = y;

    y -> left = x;
    x -> parent = y;
}

void BST::RightRotate(Node* y)
{
    Node* x = y -> right;
    x -> left = y -> right;
    if(y -> right != _nil)
        y -> right -> parent = x;
    x -> parent = y -> parent;

    if(y->parent == _nil)
        _root = x;
    else if (y == y -> parent -> left)
        y -> parent -> left = x;
    else
        y -> parent -> right = x;

    x -> right = y;
    y -> parent = x;
}

Node* BST::Min(Node* node)
{
    while(node->left != _nil)
        node = node->left;
    return node;
}

Node* BST::Max(Node* node)
{
    while(node->right != _nil)
        node = node->right;
    return node;
}

Node* BST::Next(Node* node)
{
    if(node->right != _nil)
        return Min(node->right);

    Node* parent = node -> parent;
    while(parent != _nil && node == parent -> right)
    {
        node = parent;
        parent = parent -> parent;
    }
    return parent;
}

```

위 예제에서는 구현하지 않았지만 InsertFixup과 마찬가지로 레드 블랙 트리의 조건을 충족시키기 위해 delete 시에도 수정 작업을 거쳐야 한다. 

<br/>

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part3: 자료구조와 알고리즘, Rookiss
