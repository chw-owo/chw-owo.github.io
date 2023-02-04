---
title: Sort Algorithm
categories: DS-Algorithm
tags: 
toc: true
toc_sticky: true
---

이 포스트는 Rookiss님의 \<C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈 Part3: 자료구조와 알고리즘> 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 버블 정렬**

0번 인덱스와 1번 인덱스를 비교하여 큰 수를 뒤로 보낸다. 그 후 2번 인덱스와 3번 인덱스를 비교하여 큰 수를 뒤로 보낸다. 이때 1번 인덱스와 2번 인덱스를 비교하는 게 아니다. 이렇게 두 개씩 비교해서 가장 큰 수를 맨 뒤로 정렬함으로써 하나씩 자리를 확정 짓다. 이중 for 문을 이용하여 비교하며, O(N^2)의 시간 복잡도를 갖는다.

<br/>

# **2. 선택 정렬**

0번 인덱스와 1번, 2번, 3번...n번 인덱스까지 순차적으로 비교해서 가장 작은 수를 맨 앞에 정렬한다. 이렇게 하나씩 비교하여 자리를 확정 짓는다. 이중 for 문을 이용하여 비교하며, O(N^2)의 시간 복잡도를 갖는다.

<br/>

# **3. 삽입 정렬**

한 배열을 둘로 분류하여 한 집단에 있는 값을 다른 집단의 값들과 하나씩 비교하며 정렬을 진행한다. 예제 코드는 아래와 같다. 

```c++
void InsertionSort(vector<int>& v)
{
    constint n = (int)v.size();

    for(int i = 1; i < n; i++)
    {
        int insertData = v[i];
        int j;
        for(j = i-1; j>= 0; j--)
        {
            if(v[j]>insertData)
                v[j+1] = v[j];
            else
                break;    
        }

        v[j+1] = insertData;
    }
}
```

예제에서 보이듯이 이중 for 문을 이용하여 비교하며, O(N^2)의 시간 복잡도를 갖는다. 하지만 대부분의 데이터가 정렬되어 있는 상태에서는 버블 정렬, 선택 정렬보다 빠르게 동작할 수 있다.

<br/>

# **4. 힙 정렬**

```c++
void HeapSort(vector<int>& v)
{
    priority_queue<int, vector<int>, greater<int>> pq;

    for (int num: v)
        pq.push(num);

    v.clear()

    while(qp.empty() == false)
    {
        v.push_back(pq.top());
        pq.pop;
    }
}
```

힙 이진 트리의 특성을 이용하여 정렬하는 것으로, c++에서는 priority_queue를 이용해 쉽게 구현할 수 있다. 이진 트리를 활용하기 정렬하는 것이기 때문에 O(NlogN)의 시간 복잡도를 갖는다.

<br/>

# **5. 병합 정렬**

```c++ 
void MergeSort(vector<int>& v, int left, int right)
{
    if(left >= right)
        return;

    int mid = (left + right) / 2;
    MergeSort(v, left, mid);
    MergeSort(v, mid + 1, right);

    MergeResult (v, left, mid, right);
}

void MergeResult(vector<int>& v, int left, int, mid, int right)
{
    int leftIdx = left;
    int rightIdx = mid + 1;
    vector<int> tmp;

    while(leftIdx <= mid && rightIdx <= right)
    {
        if(v[leftIdx]<= v[rightIdx])
        {
            tmp.push_back(v[leftIdx]);
            leftIdx++;
        }
        else
        {
            tmp.push_back(v[rightIdx]);
            rightIdx++;
        }
    }

    if(leftIdx > mid)
    {
        while(rightIdx <= right)
        {
            tmp.push_back(v[rightIdx]);
            rightIdx++;
        }
    }
    else
    {
        while(leftIdx <= mid)
        {
            tmp.push_back(v[leftIdx]);
            leftIdx++;
        }
    }

    for(int i = 0; i < tmp.size(); i++)
    {
        v[left + i] = tmp[i];
    }
}
```

분할 - 정복 - 결합의 과정을 거쳐 재귀적으로 정렬한다. 예를 들어 크기가 8인 배열이라면 4 크기의 두 집단, 그걸 다시 2 크기의 네 집단, 이를 한 개씩 총 8개로 나누고, 이를 두 집단씩 다시 묶어 차례로 비교한다. 비교했을 때 값이 작은 순서대로 tmp vector에 push 하고 정렬이 끝났을 때 이를 최종 결과로 출력한다. 

병합 정렬도 힙 정렬과 마찬가지로로 O(NlogN)의 시간 복잡도를 갖는다. 그러나 멀티스레드로 실행했을 때, 분할된 각 영역을 별도로 작업할 수 있기 때문에 상황에 따라 더 빠르게 동작할 수 있다. 

<br/>

# **6. 퀵 정렬**

```c++
int Partirion()(vector<int>& v, int left, int right)
{
    int pivot = v[left];
    int low = left + 1;
    int high = right;

    while(low <= high)
    {
        while(low <= right && pivot >= v[low])
            low++;

        while(high >= left + 1 && pivot <= v[high])
            high--;

        if(low < high)
            swap(v[low], v[high]);
    }

    swap(v[left], v[high]);
    return high;
}

void QuickSort(vector<int>& v, int left, int right)
{
    if(left > right)
        return;

    int pivot = Partition(v, left, right);
    QuickSort(v, left, pivot - 1);
    QuickSort(v, pivot - 1, right);
}
```

배열의 0번 인덱스 값을 피봇으로 잡고, 1번 인덱스 값을 low, 맨 마지막 인덱스 값을 high로 잡는다. low가 가리키는 값이 pivot보다 클 때까지 low를 오른쪽으로, high가 가리키는 값이 pivot보다 작을 때까지 high를 왼쪽으로 이동하여 low, high가 각자 가리키는 값의 위치를 바꾼다. 이를 쭉 이어가다가 high <= low가 되면 pivot과 high의 위치를 교체한다. 그 결과로 pivot은 제 위치를 찾아가게 되고 pivot의 왼쪽에는 pivot보다 작은 값이, 오른쪽에는 pivot보다 큰 값이 정렬된다. 그럼 pivot을 기준으로 왼쪽, 오른쪽에 각각 재귀적으로 퀵 정렬을 반복한다. 

이는 평균적으로 O(NlogN)의 시간 복잡도를 갖는다. 그러나 만약 pivot 값이 중앙값으로 잘 잡힌 경우 O(N)의 시간복잡도를 갖게 된다. 반면 이미 거의 정렬된 배열이어서 가장 작은 값을 계속 pivot으로 잡게 되는 경우, O(N^2)의 시간복잡도를 갖게 될 수도 있다. 

<br/>

# **출처**

[C++과 언리얼로 만드는 MMORPG 게임 개발 시리즈] Part3: 자료구조와 알고리즘, Rookiss
