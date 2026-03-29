---
layout: post
title: 低内存占用Astar算法-基于栅格地图
date: 2026-03-28 10:00:00
modify: 2026-03-28 10:00:00
categories: [planning]
tags: [astar, memory]
description: 低内存占用Astar算法-基于栅格地图
---

# 低内存占用Astar算法-基于栅格地图

在机器人导航（例如基于 ROS 的 30m×30m，分辨率 0.05m 的局部代价地图）或者游戏寻路开发中，A*(A-Star) 算法是最常用的寻路算法。然而，在标准的面向对象实现中，A* 算法的内存占用和频繁的动态内存分配往往会成为性能瓶颈。

先来算一笔账：在一个 600×600（总计 360,000 个栅格）的地图上，一个标准的 `Node` 节点通常这样定义：

```cpp
struct Node {
    uint16_t g;          // 2B (起点到当前节点的代价)
    uint16_t h;          // 2B (启发式预估代价)
    uint16_t x, y;       // 4B (坐标)
    Node *parent;        // 8B (64位机器上的指针)
    NodeState state;     // 4B (枚举类型通常占用4字节)
    // padding           // 根据字节对齐，可能还有额外填充
};  // 总计 ≈ 20~24 字节/节点
```

如果我们使用 `std::vector<Node *> mapping` 和 `std::priority_queue` 作为 Open List，主要内存消耗如下：

| 数据结构                 | 计算方式      | 占用大小             |
| :----------------------- | :------------ | :------------------- |
| `mapping` (节点指针数组) | 360,000 × 8B  | ~2,812 KB            |
| `Node` 实例对象          | 360,000 × 24B | ~8,437 KB (动态分配) |

> **痛点分析：**
>
> 1. `mapping` 和 `Node` 对象占用了高达 11MB 的内存。
> 2. 节点对象在探索过程中会非常频繁地 `new/delete`，不仅产生内存碎片，还会导致 CPU 缓存命中率（Cache Locality）极低，严重拖慢寻路速度。

本文将介绍一种**面向数据设计（Data-Oriented Design）**的低内存 A* 算法实现，通过消除指针、状态压缩和自定义索引优先队列，将内存占用**降低 80%**。

## 核心优化思想

### 1. 消除对象与指针 (AoS 转 SoA)

在 64 位系统中，指针占用 8 个字节，非常奢侈。我们可以**彻底抛弃 Node 结构体**，将结构体数组（AoS）转化为多个平行的基础类型数组（SoA），用数组索引（`index = y * width + x`）来代替指针和坐标。

对于原本 `Node` 中的属性，逐一进行压缩：

1. **`g` (实际代价)**：必须存储，在大多数场景中 `uint16_t` (2字节) 足够。
2. **`h` (启发代价)**：**不存储**。计算 `h` 只是简单的几步 CPU 运算（如曼哈顿或对角线距离），以现代 CPU 的算力，动态计算的代价远低于读取内存的代价（Cache Miss）。
3. **`x, y` (坐标)**：**不存储**。直接用一维数组的索引 `idx` 代替，用时可通过 `idx % W` 和 `idx / W` 还原。
4. **`parent` (父节点指针)**：原本占用 8 字节。由于栅格地图只有 8 个移动方向，我们只需用一个 `uint8_t` (1字节) 存储**来源方向 (0~7)**，回溯时反推即可。
5. **`state` (节点状态)**：原本需要 4 字节。实际上只需区分 未访问、Open、Closed，我们可以将其与后面的“堆索引”复用（见后文）。

| 原始字段  | 原大小 | 优化策略                             | 优化后大小 |
| --------- | ------ | ------------------------------------ | ---------- |
| `g`       | 2B     | **必须保留**                         | 2B         |
| `h`       | 2B     | **不存储，用时再算（以算力换空间）** | 0B         |
| `x, y`    | 4B     | **用数组全局索引代替**               | 0B         |
| `*parent` | 8B     | **仅存来源方向(0~7)，回溯时反推**    | 1B         |
| `state`   | 4B     | **复用堆位置编码**                   | 0B         |

> 在标准的 C++ 开发中，我们习惯使用 struct 或 class 将相关属性封装在一起，组成结构体数组 (AoS - Array of Structures)。这种做法极具封装性，对人类阅读非常友好。
> 
> 然而，当我们面临极端性能压榨（如游戏引擎或大规模寻路）时，面向对象往往会成为阻碍。在 A* 的扩展阶段，我们常常只需要查询节点的地形代价值（Terrain）或状态（Open/Closed），而不需要用到 g 值或父节点指针。如果使用 Node 对象，CPU 加载缓存线（Cache Line，通常 64 字节）时，会把当前用不到的冗余字段也加载进来，导致宝贵的缓存空间被浪费。
> 
> 本文的优化本质上是转向了面向数据设计 (DOD - Data-Oriented Design)，将结构体打散成了数组结构 (SoA - Structure of Arrays)。这种扁平化的连续内存排布，不仅大幅消灭了字节对齐产生的内存空洞，还能确保 CPU 预取数据时，每一次拉取的都是当前计算真正需要的紧凑数据，从而极大提升缓存命中率。

### 2. 弃用 `std::priority_queue`，手写索引二叉堆

C++ 标准库的 `std::priority_queue` 有一个致命弱点：**不支持高效的 Decrease-Key（更新节点代价）**。
标准的做法是采用“懒删除”——发现更优路径时，直接把新代价放入堆中，旧的不删。这会导致**堆膨胀**，不仅消耗额外内存，还增加了出队的无效判定。

为了解决这个问题，我们使用两个简单的数组来实现一个**支持 O(1) 定位、O(log N) 更新**的索引二叉堆（Indexed Priority Queue）：

```cpp
uint16_t heap_pos[MAP_SIZE]; // 记录地图上每个节点在堆数组中的下标位置
int32_t  heap_data[HEAP_CAP];  // 实际的堆数组，存储地图节点的索引 (index)
```

#### 高效的 Decrease-Key 实现

当我们发现了一条到达某个节点更优的路径时，直接通过 `heap_pos[node_idx]` 找到它在堆中的位置，然后执行**上浮（Sift Up）**操作。零冗余，没有无效探索！

```cpp
// O(1) 数组寻址 + O(log n) 上浮
void heap_decrease(int node_idx) {
    heap_sift_up(heap_pos[node_idx]);
}
```

### 3. 字段合并与状态压缩

在 `heap_pos` 数组中，我们只需记录 Open 列表中的节点在堆里的位置。因为 Open 列表的大小（通常对应搜索波前的边界环长 $O(\sqrt{N})$ ）远小于整个地图大小，`uint16_t` (最大 65535) 绰绰有余。

既然如此，我们可以巧妙地利用 `heap_pos` 的特殊值来替代原来的 `NodeState state` 字段：

```cpp
// 复用 heap_pos 的值域来表示状态
heap_pos[node] = 0xFFFF      // 等于 65535 → 状态：UNVISITED (未访问)
heap_pos[node] = 0 ~ 65533   // 正常堆位  → 状态：OPEN (在优先队列中)
heap_pos[node] = 0xFFFE      // 等于 65534 → 状态：CLOSED (已探索完毕)
```

### 内存对比总结

通过以上优化，我们彻底消灭了动态内存分配，来看看惊人的内存缩减：

```text
               原始对象方案           当前优化方案
           ────────────────      ────────────────
Node对象    8,437 KB                 0 KB    (彻底消除)
mapping     2,812 KB                 0 KB    (彻底消除)
g_cost          -                  703 KB
dir_from        -                  351 KB
heap_pos        -                  703 KB
heap_data       -                  256 KB
           ────────────────      ────────────────
总计        ~11,600 KB            ~2,364 KB
```

**结果：内存占用直接减少了约 80%！** 并且由于采用了一维连续数组，CPU 预取数据的缓存命中率大幅上升，寻路速度也会有质的飞跃。

---

## 完整的 C++ 实现代码

代码为了追求极致性能使用了全局静态数组（适用于单线程或嵌入式环境）。如果在多线程环境中使用，可以将其封装为类的成员变量。

```cpp
#include <cstdint>
#include <cstdio>
#include <cstdlib>
#include <cstring>

// ==================== 地图配置 ====================
static const int MAP_W = 600;
static const int MAP_H = 600;
static const int MAP_SIZE = MAP_W * MAP_H; // 360,000

// ==================== 堆容量 ====================
// A* 开放列表实际大小 ≈ 边界环 ≈ O(sqrt(N))
// 给 65535 绰绰有余（实测通常不超过 5000）
static const int HEAP_CAP = 65535;

// ==================== 移动代价 ====================
// 最坏 g 值验证:
// max_steps = 1200 (600格对角线)
// max_g = 1200 × 7 × 5 = 42,000 < 65,535 (uint16_t 防溢出验证 ✓)
static const int COST_STRAIGHT = 5;
static const int COST_DIAG = 7;

// 8方向
static const int DX[8] = {0, 0, -1, 1, -1, 1, -1, 1};
static const int DY[8] = {-1, 1, 0, 0, -1, -1, 1, 1};
static const int MOVE_COST[8] = {COST_STRAIGHT, COST_STRAIGHT, COST_STRAIGHT,
                                 COST_STRAIGHT, COST_DIAG,     COST_DIAG,
                                 COST_DIAG,     COST_DIAG};

// ==================== 坐标转换 ====================
inline int to_idx(int x, int y) { return y * MAP_W + x; }
inline int idx_x(int idx) { return idx % MAP_W; }
inline int idx_y(int idx) { return idx / MAP_W; }

// ==================== 全局预分配连续数组 (SoA) ====================
static uint8_t  terrain[MAP_SIZE];   // 地形代价 0=墙 1~5
static uint16_t g_cost[MAP_SIZE];    // 累计 g 值
static uint8_t  dir_from[MAP_SIZE];  // 来源方向 (0~7, 0xFF=起点)
static uint16_t heap_pos[MAP_SIZE];  // 堆位置 / 状态节点
static int32_t  heap_data[HEAP_CAP]; // 二叉堆数据数组

// ==================== 状态特殊值 ====================
static const uint16_t POS_CLOSED = 0xFFFE;
static const uint16_t POS_UNVISITED = 0xFFFF;
static const uint8_t  DIR_NONE = 0xFF;
static const uint16_t G_INF = 0xFFFF;

// ==================== 启发函数 ====================
static int s_goal; // 当前搜索的目标

inline uint16_t heuristic(int idx) {
  int dx = abs(idx_x(idx) - idx_x(s_goal));
  int dy = abs(idx_y(idx) - idx_y(s_goal));
  int mn = dx < dy ? dx : dy;
  int mx = dx > dy ? dx : dy;
  // h = straight*(mx-mn) + diag*mn
  return (uint16_t)(COST_STRAIGHT * (mx - mn) + COST_DIAG * mn);
}

// 动态计算 f 值，省去存储开销
inline int f_value(int idx) { return (int)g_cost[idx] + (int)heuristic(idx); }

// ==================== 索引最小堆实现 ====================
static int heap_size;

inline void heap_swap(int i, int j) {
  int a = heap_data[i], b = heap_data[j];
  heap_data[i] = b;
  heap_data[j] = a;
  heap_pos[b] = (uint16_t)i;
  heap_pos[a] = (uint16_t)j;
}

void heap_sift_up(int i) {
  while (i > 0) {
    int p = (i - 1) >> 1; // 父节点
    if (f_value(heap_data[p]) <= f_value(heap_data[i]))
      break;       
    heap_swap(i, p); 
    i = p;         
  }
}

void heap_sift_down(int i) {
  while (2 * i + 1 < heap_size) {
    int c = 2 * i + 1; // 左子节点
    if (c + 1 < heap_size && f_value(heap_data[c + 1]) < f_value(heap_data[c]))
      c++; // 选更小的子节点
    if (f_value(heap_data[i]) <= f_value(heap_data[c]))
      break; 
    heap_swap(i, c);
    i = c;
  }
}

void heap_push(int node_idx) {
  int i = heap_size++;
  heap_data[i] = node_idx;
  heap_pos[node_idx] = (uint16_t)i;
  heap_sift_up(i);
}

int heap_pop() {
  int top = heap_data[0];
  heap_pos[top] = POS_CLOSED; // 弹出即标记为 CLOSED
  heap_size--;
  if (heap_size > 0) {
    heap_data[0] = heap_data[heap_size];
    heap_pos[heap_data[0]] = 0;
    heap_sift_down(0);
  }
  return top;
}

// O(log N) 降低代价并调整堆
void heap_decrease(int node_idx) { heap_sift_up(heap_pos[node_idx]); }

// ==================== A* 搜索主逻辑 ====================
bool astar(int sx, int sy, int ex, int ey) {
  int start = to_idx(sx, sy);
  s_goal = to_idx(ex, ey);

  // 初始化 O(N) 但是非常快的内存块置位
  memset(g_cost, 0xFF, sizeof(g_cost));   
  memset(dir_from, 0xFF, sizeof(dir_from)); 
  memset(heap_pos, 0xFF, sizeof(heap_pos)); 
  heap_size = 0;

  g_cost[start] = 0;
  dir_from[start] = DIR_NONE;
  heap_push(start);

  while (heap_size > 0) {
    int cur = heap_pop(); 

    if (cur == s_goal) return true;

    int cx = idx_x(cur);
    int cy = idx_y(cur);

    for (int d = 0; d < 8; d++) {
      int nx = cx + DX[d];
      int ny = cy + DY[d];

      if (nx < 0 || nx >= MAP_W || ny < 0 || ny >= MAP_H) continue;

      int next = to_idx(nx, ny);
      if (terrain[next] == 0) continue; // 墙壁
      if (heap_pos[next] == POS_CLOSED) continue; // 已探索

      // 防止对角线穿墙现象
      if (d >= 4) {
        if (terrain[to_idx(nx, cy)] == 0 || terrain[to_idx(cx, ny)] == 0)
          continue;
      }

      uint16_t new_g = g_cost[cur] + MOVE_COST[d] * terrain[next];

      if (heap_pos[next] == POS_UNVISITED) {
        // 发现新节点，入堆
        g_cost[next] = new_g;
        dir_from[next] = (uint8_t)d;
        heap_push(next);
      } else if (new_g < g_cost[next]) {
        // 发现更优路径，更新并上浮
        g_cost[next] = new_g;
        dir_from[next] = (uint8_t)d;
        heap_decrease(next);
      }
    }
  }
  return false;
}

// ==================== 路径回溯 ====================
int path_buf[MAP_SIZE]; 

int trace_path(int ex, int ey, int *out_path) {
  int len = 0;
  int idx = to_idx(ex, ey);

  while (dir_from[idx] != DIR_NONE) {
    out_path[len++] = idx;
    int d = dir_from[idx];
    idx = to_idx(idx_x(idx) - DX[d], idx_y(idx) - DY[d]);
  }
  out_path[len++] = idx; // 包含起点

  // 翻转路径，变为从起点到终点
  for (int i = 0, j = len - 1; i < j; i++, j--) {
    int tmp = out_path[i];
    out_path[i] = out_path[j];
    out_path[j] = tmp;
  }
  return len;
}

// ==================== 测试主程序 ====================
int main() {
  printf("=== 内存占用评估 ===\n");
  printf("terrain:   %7zu KB\n", sizeof(terrain) / 1024);
  printf("g_cost:    %7zu KB\n", sizeof(g_cost) / 1024);
  printf("dir_from:  %7zu KB\n", sizeof(dir_from) / 1024);
  printf("heap_pos:  %7zu KB\n", sizeof(heap_pos) / 1024);
  printf("heap_data: %7zu KB\n", sizeof(heap_data) / 1024);
  printf("────────────────────\n");
  size_t total = sizeof(terrain) + sizeof(g_cost) + sizeof(dir_from) +
                 sizeof(heap_pos) + sizeof(heap_data);
  printf("总计:      %7zu KB (%.2f MB)\n", total / 1024, total / 1048576.0);

  // 地形初始化
  for (int i = 0; i < MAP_SIZE; i++) terrain[i] = 1;
  // 放置墙壁
  for (int y = 100; y < 500; y++) terrain[to_idx(300, y)] = 0;
  // 放置沼泽 (高代价区)
  for (int y = 50; y < 150; y++)
    for (int x = 200; x < 280; x++) terrain[to_idx(x, y)] = 5;

  int sx = 50, sy = 300;
  int ex = 550, ey = 300;

  if (astar(sx, sy, ex, ey)) {
    int len = trace_path(ex, ey, path_buf);
    printf("\n路径已找到! 长度=%d, 总代价=%u\n", len, g_cost[to_idx(ex, ey)]);
  
    printf("前5步: ");
    for (int i = 0; i < 5 && i < len; i++)
      printf("(%d,%d) ", idx_x(path_buf[i]), idx_y(path_buf[i]));
  
    printf("\n最后5步: ");
    for (int i = len - 5; i < len; i++)
      if (i >= 0) printf("(%d,%d) ", idx_x(path_buf[i]), idx_y(path_buf[i]));
    printf("\n");
  } else {
    printf("未找到路径!\n");
  }

  return 0;
}
```
