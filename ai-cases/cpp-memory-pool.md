# 用 Claude 设计游戏服务器内存池

> **一句话总结：** 想用一个内存池管理 10 万个 NPC 对象，Claude 第一版代码在 x86 上完美运行，但搬到 ARM 移动端直接崩了——因为我没告诉它目标平台是手机。

---

## 背景

我们团队在开发一款**开放世界 SLG 手游的服务端**。游戏世界里最多同时激活 **10 万个 NPC**（巡逻守卫、商人、怪物、任务 NPC），每个 NPC 对象约 256 字节，频繁创建/销毁——玩家走近时生成，走远时回收。

用 `new`/`delete` 直接管理这 10 万个对象，性能灾难：
- 频繁的系统调用导致 CPU 尖刺
- 内存碎片化严重，运行 30 分钟后 RSS 膨胀 40%
- 无法做批量调试检测（比如统一检查内存越界）

我需要一个**定长内存池**：预分配一大块内存，自行管理分配和回收。

但问题是我对 C++17 的 `alignas`、placement new、内存屏障这些细节不熟。所以我打开了 Claude。

---

## 对话记录

### 第一轮：提需求，得到一个「看起来不错」的方案

> **我：**
> ```
> 我需要一个 C++17 的定长内存池，需求如下：
> 1. 管理最多 10 万个固定大小（256 字节）的对象
> 2. O(1) 的分配和释放
> 3. 线程安全
> 4. 支持 placement new 构造对象
> 5. 内存连续，方便 cache 友好遍历
> 
> 请给出完整实现，包括头文件和 cpp 文件。
> ```

> **Claude（摘要）：**
> 
> Claude 给出了一个基于空闲链表（free list）的经典实现：
> - 用 `std::vector<char>` 作为底层存储
> - 空闲块用内部指针串联成链表
> - `allocate()` 从链表头取一块，`deallocate()` 放回链表头
> - 加了 `std::mutex` 做线程安全
> - 代码结构清晰，有注释，约 120 行
>
> 看起来很好——直到我注意到一个问题。

**我的修正：**

> **我：**
> ```
> 等等，你用的是 std::vector<char>，这在我的目标平台上有问题。
> 我们的服务器跑在 ARM64 移动设备上（类似树莓派级别的边缘服务器），
> vector 的内存对齐只有 alignof(char) == 1。
> 但 NPC 对象里有 int64_t 时间戳和 double 坐标，需要 8 字节对齐。
> 这个方案在 ARM 上会导致 unaligned access 故障。
> ```

这是第一次关键纠正。Claude 默认假设了 x86 桌面环境（x86 对非对齐访问容忍度很高，只影响性能不崩溃），但 ARM 上非对齐访问是硬件异常。这个约束必须我主动提。

---

### 第二轮：追问对齐，方案升级

> **我：**
> ```
> 请修正：底层存储必须保证 64 字节对齐（ARM cache line），
> 并且每个对象块也要 cache line 对齐（64 字节边界）。
> 用 C++17 的 alignas。另外确认：ARM64 上这个方案还有没有其他隐患？
> ```

> **Claude（摘要）：**
>
> Claude 这次做了两个重要修正：
>
> 1. 底层存储改用 `std::aligned_alloc`（或 `operator new` 带对齐参数）：
>    ```cpp
>    static constexpr size_t CACHE_LINE = 64;
>    alignas(CACHE_LINE) std::byte pool[POOL_SIZE];
>    // 或者动态分配：
>    void* raw = ::operator new(POOL_SIZE, std::align_val_t{CACHE_LINE});
>    ```
>
> 2. 对象块大小向上取整到 64 的倍数：
>    ```cpp
>    static constexpr size_t BLOCK_SIZE = ((sizeof(NPC) + CACHE_LINE - 1) / CACHE_LINE) * CACHE_LINE;
>    ```
>
> Claude 还主动提醒了 **false sharing** 问题：如果两个线程频繁访问相邻 cache line 上的 NPC，性能会退化。建议在高竞争场景下用 `alignas(CACHE_LINE)` 标注热字段。

这次修正后，代码在 ARM 上应该能跑了。但我转念一想——万一有越界写入呢？10 万个对象里出一个越界 bug，排查难度是噩梦级别的。我需要在调试模式下自动检测。

---

### 第三轮：加调试模式，Claude 给出 canary 方案

> **我：**
> ```
> 加一个调试模式（用编译宏 -DDEBUG_POOL 控制）：
> - 在每个对象块的头部和尾部插入 canary（魔数），释放时检查
> - 如果 canary 被破坏，立刻 assert 并打印是第几个块、哪一侧越界
> - 记录每个块的分配调用栈（用 __builtin_return_address），
>   方便定位谁分配的却没释放（类似简易内存泄漏检测）
> - 注意：调试模式会增大块大小，别破坏对齐
> ```

> **Claude（摘要）：**
>
> Claude 给出了一个设计精良的调试层：
>
> ```cpp
> struct BlockHeader {
>     static constexpr uint64_t CANARY = 0xDEADBEEFCAFEBABE;
>     uint64_t front_canary;
>     void* alloc_callstack[8];   // 分配调用栈
>     size_t block_index;
>     // ... 对象数据在这里 ...
>     // alignas(64) 保证对象起始地址对齐
> };
> 
> struct BlockFooter {
>     uint64_t rear_canary;
> };
> ```
>
> 布局变成：`[Header 64B | Object 256B | Footer 8B]`，总共 328 字节，但对齐到 64 字节后实际占用 384 字节。
>
> 释放时的检查逻辑：
> ```cpp
> void deallocate(void* ptr) {
>     auto* obj = static_cast<std::byte*>(ptr);
>     auto* header = reinterpret_cast<BlockHeader*>(obj - sizeof(BlockHeader));
>     auto* footer = reinterpret_cast<BlockFooter*>(obj + BLOCK_SIZE);
>     
>     assert(header->front_canary == BlockHeader::CANARY && 
>            "Front canary corrupted!");
>     assert(footer->rear_canary == BlockHeader::CANARY && 
>            "Rear canary corrupted!");
>     // ... 放回空闲链表 ...
> }
> ```

---

## 最终成果

（关键部分，完整代码约 200 行）

```cpp
// npc_memory_pool.h
#pragma once
#include <atomic>
#include <cstddef>
#include <new>

class NPCMemoryPool {
public:
    static constexpr size_t OBJECT_SIZE = 256;
    static constexpr size_t CACHE_LINE = 64;
    static constexpr size_t MAX_OBJECTS = 100'000;

#ifdef DEBUG_POOL
    // 调试模式：每个块 = Header + Object + Footer，总大小对齐到 cache line
    static constexpr size_t BLOCK_SIZE = 
        ((sizeof(BlockHeader) + OBJECT_SIZE + sizeof(BlockFooter) + CACHE_LINE - 1) 
         / CACHE_LINE) * CACHE_LINE;
#else
    static constexpr size_t BLOCK_SIZE = 
        ((OBJECT_SIZE + CACHE_LINE - 1) / CACHE_LINE) * CACHE_LINE;
#endif

    NPCMemoryPool();
    ~NPCMemoryPool();

    void* allocate();
    void  deallocate(void* ptr);

    // 调试：打印统计信息
    size_t allocated_count() const { return allocated_.load(); }

private:
    std::byte* pool_;                    // 对齐到 CACHE_LINE
    std::atomic<void*> free_list_;       // 空闲链表头
    std::atomic<size_t> allocated_;      // 已分配计数

    // ... 调试相关成员 ...
};
```

---

## 关键收获

### 1. 给 AI 提供平台约束，是得到正确方案的前提

第一轮我只给了功能需求，Claude 给了标准 x86 方案。第二轮我补充了「ARM64 + cache line 对齐」，方案立刻升级。**这不是 AI 能力不足，是上下文不足。**

类比：你告诉医生「头疼」，他开布洛芬；你补充「有胃溃疡史 + 正在吃抗凝药」，他开的药会完全不同。

### 2. AI 擅长「你明确要求的」，但不太会主动推测「你可能需要的」

调试模式（canary、调用栈）是我主动提出的，AI 执行得很好。但如果我不提，它不会自己说「你这个内存池要不要加越界检测？」——因为它在等我的需求信号。

### 3. 「追问」比「重写 prompt」更高效

我没有在发现对齐问题后重新写一大段需求，而是直接追问一个具体点。这样 AI 保留了上一轮的上下文，修正更精准。

### 我的评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 方案正确性 | ⭐⭐⭐⭐ | 三轮迭代后达到生产可用水平 |
| 平台适配 | ⭐⭐⭐⭐⭐ | 第二轮后处理了 ARM 对齐和 cache line |
| 可调试性 | ⭐⭐⭐⭐⭐ | canary + 调用栈记录，超越了我的预期 |
| 需要我修正 | 3 次 | 对齐、平台、调试模式，每次都是我主动提出 |

让我印象最深的是：**Claude 的第二轮回复里，主动提到了 false sharing 这个我没问的问题**。这说明一旦给了正确的上下文约束，AI 的推理质量会跃升。

---

## 图谱知识点映射

- [1.1.6 内存管理](../mds/1.基础能力/1.1.6.内存管理.md) — 内存对齐、内存池
- [1.1.2 C++语言](../mds/1.基础能力/1.1.2.C++语言.md) — placement new、alignas、智能指针对比
- [1.3.3 操作系统](../mds/1.基础能力/1.3.3.操作系统.md) — cache line、false sharing

> 🏥 相关实战案例：[SLG手游内存泄漏 — shared_ptr 循环引用](../cases/memory-leak-slg.md)
