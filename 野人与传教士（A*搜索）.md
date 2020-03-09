# 用A*搜索解决野人与传教士问题（M-C问题）

## 题目描述：
有N个传教士和N个野人来到河边渡河，河岸有一条船，每次至多可供k人乘渡。为了传教士安全起见，在任何时刻河两岸以及船上的野人数目总是不超过传教士的数目(否则不安全，传教士有可能被野人吃掉)。求解最小的渡河次数。

### 输入与输出：
输入为一行，有两个数：野人和传教士的数目N 以及 船每次可乘度的人k
输出为一行，即能让野人与传教士安全渡河的最小渡河次数

#### 输入输出样例：
- case1:  
3 2  
11  
- case2:  
6 4  
9  
- case3:  
100 10  
49  

## M-C问题的搜索节点表示

### 如何描述一个搜索节点
描述一个搜索节点需要四个参数：到达这一节点的运载次数g，左岸的传教士人数missionaries，左岸的野人人数cannibals，船是否在左岸boat

### 目标节点的参数
missionaries和cannibals都为0

### 节点安全的条件
一共有三种情况：
- 传教士都在左岸
- 传教士都不在左岸
- 左岸传教士与野人数量相等

## A*搜索
- A*算法是一种静态路网中求解最短路最有效的直接搜索方法，是启发式搜索的一种。  
- 对于每一种可行的搜索节点，进行如下的估值： f(n)=g(n)+h(n) ，  
    - 其中 f(n) 是从初始点经由节点n到目标点的估价函数，  
    - g(n) 是在状态空间中从初始节点到n节点的实际代价，  
    - h(n) 是从n到目标节点估计的代价下界。h(n)的设置，目的在于提升搜索的效率。  
- 一个完备的A*搜索，需要保证：h(n) 是从n到目标节点估计的代价**下界**，否则不能保证能搜索到最优解。
- 在进行A*搜索时，首先搜索估值函数最小的节点。

### A*搜索的具体实现
- 考虑到估值函数之后，还需要在一个节点的描述中加入估值函数的值，即描述一个节点，需要5个参数
- 利用**优先队列**
  - 将估值函数小的排在前，估值函数大的排在后
  - 如何进行一次搜索
    - 若队列为空，则结束搜索，且不存在成功的方案
    - 若队列不为空
      - 如果队首节点的状态是抵达目标节点，则搜索结束
      - 若不然，对于队首节点，遍历其所有的子节点，并push进优先队列中，最后弹出队首节点
  
### M-C问题中的估值函数
- g(n)即为到达节点n，所需要的最少运载次数 
- 分两种情况考虑h(n)：船在左岸和船在右岸  
    - 若船在左岸：  
        不考虑安全问题，一次来回最多可以运输k-1个人  
        因此h(n)为((missionaries + cannibals)/(k-1))向上取整，乘以2，再加1  
        化简得h = ((missionaries + cannibals - 2) / (k - 1)) * 2 + 1
    - 若船在右岸：
        首先需要至少一人把船划回来  
        因此h(n)为((missionaries + cannibals + 1)/(k-1))向上取整，乘以2，再加2  
        化简得h = ((missionaries + cannibals - 1) / (k - 1)) * 2 + 2
    - 对于以上的两个公式，需要特殊考虑被除数小于零的情况
- 最后估值函数如下：
    ```
    if (boat) // 船在左岸
    {
        if (missionaries + cannibals >= 2)
            h = ((missionaries + cannibals - 2) / (k - 1)) * 2 + 1;
        else // 注意讨论被除数小于零的情况，至少需要一次运载过河
            h = 1;
    }
    else
    {
        if (missionaries + cannibals >= 1)
            h = ((missionaries + cannibals - 1) / (k - 1)) * 2 + 2;
        else // 注意讨论被除数小于零的情况，至少需要两次运载过河
            h = 2;
    }
    // 运载次数是父节点运载次数加一
    g = last_g + 1;
    // 总估值函数
    f = h + g;
    ```

### 搜索中的剪枝
- 在M-C问题中，部分节点无需重新搜索。例如两个节点 temp1(2,95,95,True,g1) 和 temp2(4,95,95,True,g2) ，搜索过temp1之后，不必再搜索temp2。
- 一个比较直接的想法是，建立一个数组 bool searched[missionaries][cannibals][boat] 记录一个状态 (missionaries_, cannibals_, boat_) 是否已经被搜索过，如果搜索过，则剪去这个节点。
- 但是以上的做法有一个问题：在实际搜索过程中，无法保证对于两个相同的状态，抵达这一状态运载次数少的先被搜索。例如，temp1(4,92,92,True,g1) 并不一定先于 temp2(6,92,92,True,g2) 被搜索。因此，会导致节点的丢失。
- 因此，需要建立数组 int searched[missinaries][cannibals][boat] 记录在之前所有的搜索过的节点中，到达状态 (missionaries_, cannibals_, boat_) 所需要的最小步数。
- 若在后续的搜索中，再次遇到状态 (missionaries_, cannibals_, boat_)
  - 若当前的 g >= searched[missinaries_][cannibals_][boat_]， 则剪去这个节点。
  - 若当前的 g < searched[missinaries_][cannibals_][boat_]，则令 searched[missinaries_][cannibals_][boat_] = g ，且把这个节点push进优先队列中。
- 代码示例如下：
  ```
  // 检验一个节点是否不必搜索下去
    bool searchCheck() // g为当前运载次数
    {
        // 如果 已经搜索过 且 当前运载次数大于等于之前的最小次数
        if (searched[missionaries][cannibals][boat] != 0 && g >= searched[missionaries][cannibals][boat])
            return true;
        else
        {
            searched[missionaries][cannibals][boat] = g; // 重新设置到达此状态的最小运载次数
            return false;
        }
    }
    ```

## 代码示例：
```
#include <iostream>
#include <queue>

using namespace std;

int go[6000][2];                   // 对于一个船的容量，可行的移动人数：missionaries, cannibals
int pos_go = 0;                    // 可行的移动方案数
int m;                             // 传教士和野人的个数
int searched[1500][1500][2] = {0}; // 标记到达每一个搜索过的情况，所需要的最小步数

class bank
{
public:
    static int capacity; // 船的容量
    int missionaries;
    int cannibals;
    bool boat;   // 船是否在左岸
    int f, h, g; // f = h + g , f为局面估值
    bank(int last_g, int missionaries_, int cannibals_, bool boat_) : missionaries(missionaries_), cannibals(cannibals_), boat(boat_)
    {
        // 计算启发式函数H
        if (boat) // 船在左岸
        {
            if (missionaries + cannibals >= 2)
                h = ((missionaries + cannibals - 2) / (capacity - 1)) * 2 + 1;
            else // 注意讨论被除数小于零的情况，至少需要一次运载过河
                h = 1;
        }
        else
        {
            if (missionaries + cannibals >= 1)
                h = ((missionaries + cannibals - 1) / (capacity - 1)) * 2 + 2;
            else // 注意讨论被除数小于零的情况，至少需要两次运载过河
                h = 2;
        }
        // 运载次数是父节点运载次数加一
        g = last_g + 1;
        // 总估值函数
        f = h + g;
    }
    // 用于优先队列中的节点排序
    friend bool operator<(bank a, bank b) // 如果采用注释中的方法排列优先队列中的节点，可以使情况更理想
    {
        // if (a.f != b.f)
        return a.f > b.f; // f小的优先级高
        // else if (a.g != b.g)
        // return a.g > b.g;
        // else
        // return a.missionaries > b.missionaries;
    }
    // 检验一个节点是否不必搜索下去
    bool searchCheck() // g为当前运载次数
    {
        // 如果 已经搜索过 且 当前运载次数大于等于之前的最小次数
        if (searched[missionaries][cannibals][boat] != 0 && g >= searched[missionaries][cannibals][boat])
            return true;
        else
        {
            searched[missionaries][cannibals][boat] = g; // 重新设置到达此状态的最小运载次数
            return false;
        }
    }
};
int bank::capacity = 0;

priority_queue<bank> node; // 存放搜索节点的优先队列

bool is_safe(const bank &a) // 是否是安全的渡河方案 且 渡河方案是否可行
{
    if (a.missionaries < 0 || a.cannibals < 0 || a.missionaries > m || a.cannibals > m)
        return false;
    if (a.missionaries == a.cannibals || a.missionaries == 0 || a.missionaries == m)
        return true;
    else
        return false;
}

bool is_success(const bank &a) // 是否成功渡河
{
    if (a.missionaries == 0 && a.cannibals == 0)
        return true;
    else
        return false;
}

void find_go(int capacity) // 列出对于一个船的容量，可行的移动人数
{
    for (int i = capacity; i >= 1; i--) // 总过河人数
    {
        for (int j = 0; j <= i; j++) // 过河传道士
        {
            go[pos_go][0] = j;
            go[pos_go][1] = i - j;
            // cout<<go[pos_go][0]<<" "<<go[pos_go][1]<<endl;
            pos_go++;
        }
    }
}
int search() // 进行A*的搜索
{
    while (!node.empty())
    {
        bank top = node.top();
        if (is_success(top))
            return top.g;
        else
        {
            node.pop();
            if (top.boat) // 船在左岸
            {
                for (int i = 0; i < pos_go; i++)
                {
                    bank temp(top.g, top.missionaries - go[i][0], top.cannibals - go[i][1], false);
                    if (is_safe(temp) && !temp.searchCheck())
                        node.push(temp);
                }
            }
            else // 船在右岸
            {
                bool flag = false;
                for (int i = pos_go - 1; i >= 0; i--)
                {
                    bank temp(top.g, top.missionaries + go[i][0], top.cannibals + go[i][1], true);
                    if (is_safe(temp) && !temp.searchCheck())
                        node.push(temp);
                }
            }
        }
    }
    return -1;
}

int main()
{
    cin >> m >> bank::capacity;
    find_go(bank::capacity); // 寻找可能走法
    node.push(bank(-1, m, m, true));
    cout << search() << endl;

    return 0;
}
```
### 部分注意事项：
- 可以在枚举时贪心一些：
  - 将人运到右岸时，从总人数最多的开始枚举
  - 将人运回左岸时，从总人数最少的开始枚举
- 其实这道题目可以直接使用广度优先搜索，这时候标注已经被枚举过的情况就会比较容易
- 注意数组不要越界