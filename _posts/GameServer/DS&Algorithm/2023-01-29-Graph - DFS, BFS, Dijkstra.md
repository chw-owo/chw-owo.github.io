---
title: Graph 1) DFS, BFS, Dijkstra
categories: DS&Algorithm
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part3: 자료구조와 알고리즘> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. Graph**

현실 세계의 사물이나 추상적인 개념간의 연결관계를 표현하는 것으로, 정점(vertex)로 데이터를, 간선(edge)으로 데이터 간의 관계를 표현한다. 간선에 가중치를 주는 weighted graph, 관계 간 방향이 있는 directed graph 등이 있다. 

<br/>

# **2. DFS**

DFS는 Depth First Search, 깊이 우선 탐색을 의미한다. 시작 노드에서 한 노드으로 들어가고 나면 다음 분기로 넘어가기 전에 노드의 가장 마지막 깊이까지 완벽히 탐색한 뒤, 다음 분기를 탐색한다. 자기 자신을 호출하는 순환 알고리즘의 형태를 가지고 있으며, 어떤 노드를 방문했는지 여부를 검사해야 한다. 그러지 않을 경우 무한 루프에 빠질 위험이 있다.  

![image](https://user-images.githubusercontent.com/96677719/215647927-a034bf0e-83ae-4b31-93fe-7bbc3ce8b583.png)

정점의 수가 N, 간선의 수가 E 일 때, 인접 리스트로 표현된 경우 O(N+E)의 시간 복잡도를, 인접 행렬로 표현된 경우 O(N^2)의 복잡도를 가진다. 따라서 간선이 적은 희소 그래프의 경우 인접 리스트가 더 유리할 수 있다. 인접 리스트로 구현한 예시는 아래와 같다.

```c++
void Dfs(int here)
{
    visited [here] = true;
    cout << "visited: " << here << endl;

    for(int i = 0; i < adjacent[here].size() ; i++)
    {
        int there = adjacent[here][i];
        if(visited[there] == false)
            Dfs(there);
    }
}

void DfsAll()
{
    visited = vector<bool>(NODE_CNT,false);

    for(int i = 0; i < NODE_CNT; i++)
        if(visited[i]==false)
            Dfs(i);
}
```

<br/>

# **3. BFS**

BFS는 Breadth First Search, 너비 우선 탐색을 의미한다. 시작 노드으로부터 같은 깊이에 있는 인접한 노드들 먼저 순차적으로 탐색한다. 결과적으로 가장 멀리 떨어진 정점은 가장 나중에 방문하게 된다. DFS와 달리 재귀적으로 동작하지 않으며, 이 역시 어떤 노드를 방문했는지 여부를 검사하지 않으면 무한 루프에 빠질 위험이 있다. 

![image](https://user-images.githubusercontent.com/96677719/215648441-5756b36c-baff-480c-a237-01e4bcb50e0c.png)

정점의 수가 N, 간선의 수가 E 일 때, 인접 리스트로 표현된 경우 O(N+E)의 시간 복잡도를, 인접 행렬로 표현된 경우 O(N^2)의 복잡도를 가진다. 따라서 간선이 적은 희소 그래프의 경우 인접 리스트가 더 유리할 수 있다. 인접 리스트로 구현한 예시는 아래와 같다.

```c++
void Bfs(int here)
{
    queue<int> q;
    q.push(here);
    discovered[here] = true;

    while(q.empty() == false)
    {
        here = q.front();
        cout << "visited: " << here << endl;
        q.pop();

        for(int there : adjacent[here])
        {
            if(discovered[there])
                continue;

            q.push(there);
            discovered[there] = true;
        }
    }
}

void BfsAll()
{
    visited = vector<bool>(NODE_CNT,false);

    for(int i = 0; i < NODE_CNT; i++)
        if(visited[i]==false)
            Bfs(i);
}
```

<br/>

# **4. Dijkstar**

다익스트라 알고리즘은 그래프 특정 노드에서 갈 수 있는 모든 노드들까지의 최단 경로를 구하는 알고리즘으로, 큰 그래프에서도 효율적으로 동작하는 장점이 있는 반면 간선의 가중치가 음수인 것이 하나라도 있다면 사용할 수 없다는 한계점도 있다. 따라서 인공위성, 네비게이션의 위치 연산 등 현실 상황에서 많이 사용된다. 

이 알고리즘은 가장 가까운 경로들을 거치다보면 최단 거리로 도착할 수 있다는 논리를 바탕으로 한다. 특정 정점까지의 최단 경로를 구한다고 했을 때, 우선 출발 노드를 기준으로 연결되어 있는 각 노드의 최소 비용을 저장한다. 그 후, 방문하지 않은 노드 중에서 가장 비용이 적은 노드를 선택하는 것을 반복하여, 현재까지 알고 있던 최단 경로를 지속적으로 갱신한다. 이때 한번 탐색된 노드는 최단 거리를 갱신한 뒤 다시 갱신되지 않는다. 

다익스트라를 구현할 때 방문하지 않은 노드 처리할 때 순차 탐색할 것인지, 우선순위 큐를 사용할 것인지 정해야 한다. 순차 탐색, 즉 단순 선형 탐색으로 구현할 경우 일반적으로 O(N^2)의 시간 복잡도를 가진다. 반면 우선순위 큐, 힙 구조를 활용해 구현할 경우 O(N * logN)의 시간 복잡도를 갖는다. 순차 탐색으로 구현한 예시는 아래와 같다. 

```c++
void Dijikstra(int here)
{
    struct VertexCost
    {
        int vertex;
        int cost;
    };

    list<VertexCost> discovered;
    vector<int> best (VERTEX_CNT, INT32_MAX);
    discovered.push_back(VertexCost{here, 0});
    best[here] = 0;

    while (dicovered.empty() == false)
    {
        list<VertexCost>::iterator bestIt;
        int bestCost = INT32_MAX;

        for(auto it = discovered.begin(); it != discovered.end(); it++)
        {
            // 제일 좋은 후보를 찾는다
            if(it -> cost < bestCost)
            {
                bestCost = it -> cost;
                bestIt = it;
            }
        }

        int cost = bestIt -> cost;
        here = bestIt->vertex;
        discovered.erase(bestIt);

        // 새로 찾은 cost가 현재 최단 거리보다 
        // 가깝다면 아래 코드를 실행한다.
        if(best[here] < cost)
            continue;
        
        for (int there = 0; there < VERTEX_CNT; there++)
        {
            if(adjacent[here][there] == -1)
                continue;

            // 최단 거리와 최단거리를 합친 거리가
            // 기존 cost보다 가깝다면 아래 코드를 실행한다.
            int nextCost = best[here] + adjacent[here][there];
            if(nextCost >= best[there])
                continue;

            //최단 거리로 갱신한 뒤 discovered에 push 한다.
            best[there] = nextCost;
            discovered.push_back(VertecCost { there, nextCost });

        }
    }

}
```

<br/>

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part3: 자료구조와 알고리즘, Rookiss

https://gmlwjd9405.github.io/2018/08/14/algorithm-dfs.html

https://gmlwjd9405.github.io/2018/08/15/algorithm-bfs.html

https://velog.io/@717lumos/%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%EB%8B%A4%EC%9D%B5%EC%8A%A4%ED%8A%B8%EB%9D%BCDijkstra-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98