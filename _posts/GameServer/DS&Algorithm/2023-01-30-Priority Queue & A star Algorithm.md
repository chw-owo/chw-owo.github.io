---
title: Priority Queue &  A* Algorithm
categories: DS&Algorithm
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part3: 자료구조와 알고리즘> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Tree Basic**

트리란 데이터를 표현하는 노드(Node)와 노드의 계층 구조를 표현하기 위한 간선(Edge)로 구성되어, 계층적 구조의 데이터를 표현할 수 있는 자료구조이다. 이때, 루트에서 특정 노드에 도달하기 위해 거쳐야 하는 간선의 수를 depth, 가장 깊숙히 있는 노드의 깊이를 height라고 한다.

트리 중 각 노드가 최대 두개의 자식 노드를 가지는 트리를 두고 이진 트리라 부르고, 그 중에서도 마지막 레벨을 제외한 모든 레벨에 노드가 꽉 차 있는 것을 두고 완전 이진 트리라고 부른다. 또, 루트를 중심으로 현재 값보다 작은 경우 왼쪽에, 큰 경우 오른쪽에 배치하여 탐색을 쉽게끔 만든 구조는 이진 탐색 트리라고 부른다. 이 경우 데이터가 갱신되다보면 트리가 한쪽으로 기울어질 위험성이 있는데, 이렇게 되면 검색의 효율성이 떨어지기 때문에 트리 재배치를 통해 균형을 유지해야 한다. 

<br/>

# **2. Heap Tree**

힙 트리는 이진 트리의 일종으로 아래와 같은 두가지 요건을 충족해야 한다. 첫째로, 부모 노드의 값이 항상 자식 노드가 가진 값보다 커야한다. 이때 형제끼리 값을 비교할 필요는 없다. 둘째로, 왼쪽부터 순서를 채우는 완전 이진 트리여야 한다. 이 요건 덕분에 노드 개수를 알면 트리 구조를 확정할 수 있게 된다. 

힙 트리에서 새로운 노드를 추가할 경우, 트리의 구조를 맞춘 뒤 부모와 비교하며 노드를 한칸씩 올리면 된다. 반대로 최대값을 pop 할 경우, 우선 루트 노드를 제거한 뒤 제일 마지막에 위치한 데이터를 루트로 옮긴다. 그리고 자식들과 하나씩 비교하면서 위치를 재조정한다. 

이렇게 노드 채우는 순서 및 배치가 고정되어 있는 경우 배열을 이용하여 힙 구조를 바로 표현할 수 있게 된다. 루트의 index를 0이라고 했을 때, i번 노드의 왼쪽 자식은 i * 2 + 1를 인덱스로, 오른쪽 자식은 i * 2 + 2를 인덱스로, 특정 노드의 부모는 (i-1) / 2 의 몫을 인덱스로 갖게 된다고 확정지을 수 있기 때문이다. 

<br/>

# **3. Priority Queue 원리**

stl에서 priority_queue를 사용할 수 있다. priority_queue의 경우 일반 queue와 달리 가장 먼저 넣은 값이 가장 앞에 오는 대신, 가장 큰 값이 가장 앞에 오도록 정렬된다. 이때, 세번째 인자를 greater로 설정해주면 앞에서부터 작은 순서대로, less(default)로 설정해주면 기존대로 앞에서부터 큰 순서대로 정렬된다. PriorityQueue의 원리를 이해하기 위해 간단하게 구현해보면 아래와 같다. 

```c++
template <typename T, typename Container>
class PriorityQueue
{
public:
    void push (const T& data)
    {
        _heap.push_back(data);

        int now = static_cast<int>(_heap.size())-1;
        while(now > 0)
        {
            int next = (now - 1) / 2;
            if(_heap[now]<_heap[next])
                break;
            ::swap(_heap[now], _heap[next]);
            now = next;
        }
    }

    void pop()
    {
        _heap[0] = _heap.back();
        _heap.pop_back();

        int now = 0;
        while(true)
        {
            int next = now;
            int left = 2 * now + 1;
            int right = 2 * now + 2;

            if(left >= _heap.size())
                break;

            if(_heap[next] < _heap[left])
                next = left;

            if(right < _heap.size() && _heap[next] < _heap[right])
                next = right;

            if(next == now)
                break;

            ::swap(_heap[now], _heap[next]);
            now = next;
        }
    }

    T& top()
    {
        return _heap[0];
    }

    bool empty()
    {
        return _heap.empty();
    }

private:

}
```

<br/>

# **4. A* Algorithm**

다익스트라 알고리즘도 길찾기에 활용할 수 있지만, 이는 갈 수 있는 모든 루트를 계산한 뒤, 특정 루트의 최단 거리를 구하기 때문에 오픈 맵과 같이 갈 수 있는 루트가 많을 경우 비효율적으로 동작할 수 있다. 반면 A* 알고리즘의 경우 처음부터 출발지와 목적지를 설정하기에 목적지가 명확할 경우 더 효율적으로 사용할 수 있다. 

다익스트라 알고리즘과의 큰 차이점은 목적지를 정하고 현 위치에서 목적지 위치까지 거리(예상 소요 cost, h)를 계산하여 이를 최단 거리 연산에 반영한다는 것이다. 이럴 경우 목적지와의 거리를 고려하여 점차 가까운 방향으로 이동하는 경로를 고르게 된다. 이를 구현한 예시는 아래와 같다. 

```c++
struct PQNode
{
    bool operator<(const PQNode& other) const { return f < other.f; }
    bool operator>(const PQNode& other) const { return f > other.f; }

    int32   g; // 시작 노드부터 목적지 노드까지 소요된 cost
    int32   f; // f = g + h, h =소요될 것으로 예상된 cost
    Pos     pos;
};

void Player::Astar()
{
    Pos start = _pos;
    Pos dest = _board -> GetExitPos();

    enum
    {
        DIR_COUNT = 8 
        // 대각선을 제외하고 싶으면 4
    };

    Pos front[4] = 
    {
        Pos { -1, 0 }, //UP
        Pos { 0, -1 }, //LEFT
        Pos { 1, 0 }, //DOWN
        Pos { 0, 1 }, //RIGHT

        Pos { -1, -1 }, //UP_LEFT
        Pos { 1, -1 }, //DOWN_LEFT
        Pos { 1, 1 }, //DOWN_RIGHT
        Pos { -1, 1 }, //UP_RIGHT
    };

    int32 cost[] = 
    {
        10, 
        10,
        10,
        10,

        14,
        14,
        14,
        14
    };

    const int32 size = _board -> GetSize();
    vector<vector<int32>> best(size, vector<int32>(size, INT32_MAX)); // 최단 거리 저장
    map<Pos, Pos> parent; // 부모를 추적하기 위한 map
    vector<vector<bool>> closed(size, vector<int32>(size, false));  // Closed List, 처리가 완료된 노드 저장
    priority_queue<PQNode, vector<PQNode>, greater<PQNode>> pq; // Open List, 최단 경로 분석을 위한 상태값 갱신

    // 초기값 세팅
    {
        int32 g = 0;
        int32 h = 10 * (abs(dest.y - start.y) + abs(dest.x - start.x));
        pq.push(PQNode { g + h, g, start});
        best[start.y][start.x] = g + h;
        parent[start] = start;
    }

    // A star 알고리즘
    while(pq.empty() == false)
    {
        PQNode node = pq.top();
        pq.pop();

        if(closed[node.pos.y][node.pos.x])
            continue;
        
        closed[node.pos.y][node.pos.x] = true;

        if(node.pos == dest)
            break;

        for(int32 dir = 0; dir < DIR_COUNT; dir++)
        {
            Pos nextPos = node.pos + front[dir];
            if(CanGo(nextPos) == false)
                continue;

            if(closed[nextPos.y][nextPos.x])
                continue;

            int32 g = node.g + cost[dir]; // 실제 소요 cost
            int32 h = 10 * (abs(dest.y - start.y) + abs(dest.x - start.x)); // 예상 소요 cost, 거리로 계산

            if(closed[nextPos.y][nextPos.x] <= g + h)
                continue;

            best[nextPos.y][nextPos.x] = g + h;
            pq.push(PQNode{ g + h, g, nextPos });
            parent[nextPos] = node.pos;
        }
    }

    // 최단 경로를 역으로 추출
    Pos pos = dest;
    _path.clear();
    _pathIndex = 0;

    while(true)
    {
        _path.push_back(pos);
        if(pos == parent[pos])
            break;
        pos = parent[pos];
    }
    std::reverse(_path.begin(), _path.end());
}
```


<br/>

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part3: 자료구조와 알고리즘, Rookiss