# 实战案例：ABBA 死锁——MMO 服务器不定期卡死之谜

> _"进程还活着，但它已经死了。" —— 运维值班笔记，2024.11.03 凌晨 3:07_

## 背景

2024 年秋天，我们负责的 MMO 手游刚挺过百万 DAU 大关。团队还没从增长曲线带来的兴奋中缓过神，线上就开始闹鬼了。

事情要从运维群里一个颤抖的截图说起。凌晨 3 点，三区服务器进程的 Grafana 面板三条线全部归零：CPU 0%、网络流量 0%、日志吞吐 0%。进程还在，PID 没变，但它已经是一个僵尸——不接收连接、不处理请求、不打印一行日志。`kill -9` 是唯一能让它解脱的方式。

这套服务器的架构是典型的三线程模型：

| 组件 | 职责 | 保护对象 |
|------|------|----------|
| 网络线程 | epoll 事件循环，接收/发送玩家包 | 连接表（自管理） |
| 逻辑线程 | 处理游戏逻辑，修改玩家与公会数据 | `Player::m_mutex`、`Guild::g_mutex` |
| 定时器线程 | 定时任务调度，排行榜刷新，公会战结算 | 定时器队列（自管理） |

技术栈 C++17，Linux 环境，自研网络框架基于 epoll。`Player` 对象有自己的 `std::mutex m_mutex`，公会 `Guild` 对象有自己的 `std::mutex g_mutex`。两个锁都用于保护跨线程访问的共享数据。

卡死没有任何预警，也没有任何规律——有时一天两次，有时一周一次。运维最初认为是机器故障，迁移了三次物理机，问题照旧。我们给进程加了 `--restart=always` 的守护脚本，祈祷重启够快就不会被玩家骂上 TapTap。

直到有一次，玩家在公会战打到一半时全员掉线，TapTap 评分一夜掉了 0.3。

老板说："查，不查到根因别下班。"

## 症状

我把所有的卡死记录拉了出来，试图找规律：

| 时间 | 区服 | 在线玩家 | 运行时长 | 备注 |
|------|------|----------|----------|------|
| 10/03 15:42 | 三区 | 18,342 | 36小时 | 无异常日志 |
| 10/07 22:18 | 一区 | 21,005 | 5小时 | 刚更新版本 2.3.1 |
| 10/11 03:07 | 三区 | 6,221 | 73小时 | 低峰期，排除负载问题 |
| 10/14 19:55 | 二区 | 25,331 | 19小时 | 高峰时段 |
| 10/20 16:30 | 一区 | 17,892 | 48小时 | — |

规律？没有规律。时间随机，负载随机，区服随机。唯一确定的是：**一旦卡死，所有症状完全一致**——

1. **玩家全部掉线**：网络线程不再 accept 新连接，已有连接超时断开
2. **CPU 降为零**：不是 100% 的死循环，是真正的 0%——线程全部阻塞
3. **无任何日志**：逻辑线程不处理任何 tick，日志系统收不到写入请求
4. **进程不退出**：不是 crash，不是 OOM，就是一个完美的、沉默的僵尸

运维给了我一条救命稻草：某次卡死时他们抢在重启前做了 `gdb attach`，发来了三张 backtrace 截图：

```
# 网络线程
Thread 1 (LWP 21453):
#0  __lll_lock_wait () at ../sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1  __pthread_mutex_lock_full () at ../nptl/pthread_mutex_lock.c:312
#2  Guild::BroadcastToMembers() at guild.cpp:247
#3  NetThread::ProcessGuildMessage() at net_thread.cpp:189

# 逻辑线程
Thread 2 (LWP 21454):
#0  __lll_lock_wait () at ../sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1  __pthread_mutex_lock_full () at ../nptl/pthread_mutex_lock.c:312
#2  Player::JoinGuildBattle() at player.cpp:512
#3  LogicThread::ProcessGuildBattleStart() at logic_thread.cpp:334

# 定时器线程
Thread 3 (LWP 21455):
#0  __lll_lock_wait () at ../sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1  __pthread_mutex_lock_full () at ../nptl/pthread_mutex_lock.c:312
#2  Guild::UpdateRanking() at guild.cpp:389
#3  TimerThread::OnTimerTick() at timer_thread.cpp:156
```

三个线程，全部卡在 `pthread_mutex_lock`。没有循环，没有忙等，全部在等待某把锁。

我盯着这三张截图，后背开始发凉。

> 这不是 crash，不是 race condition，不是死循环。这是并发编程里最恶心的那个词——**死锁**。

## 排查过程

### 第一步：排除死循环，锁定死锁

先做最简单的排除法。死循环会导致 CPU 100%，但我们看到的是 0%。gdb 的 backtrace 也证实了：没有无界循环的调用栈，三个线程都在系统调用层等待。

不过 gdb attach 只能给我们一个快照——我们需要确认**每次**卡死都是同样的情况。于是我在三个线程的关键路径上插入了心跳日志：

```cpp
// 每个线程的主循环
while (running_) {
    // 每 5 秒打印一次心跳
    if (++tick_count % 5000 == 0) {
        LOG_INFO("Thread {} heartbeat: tick={}", thread_name, tick_count);
    }
    ProcessOneTick();
}
```

下一次卡死时翻日志，三条心跳线在同一秒内全部停止。也就是说——**三个线程几乎同时进入阻塞**。

死锁无疑。

### 第二步：加死锁检测探针

知道是死锁还不够，必须知道**谁跟谁锁住了什么**。但死锁发生后进程已经僵死，常规日志打不出来。我需要一个能在僵死状态下"喊救命"的机制。

方案：给 `std::mutex` 加一层薄封装，用 `try_lock` + 超时检测：

```cpp
class DeadlockAwareMutex {
public:
    template<typename Duration>
    void lock_with_timeout(Duration timeout) {
        auto deadline = std::chrono::steady_clock::now() + timeout;
        while (!mtx_.try_lock()) {
            if (std::chrono::steady_clock::now() > deadline) {
                // 超时了！打印诊断信息并告警
                LOG_CRITICAL(
                    "DEADLOCK DETECTED! Thread={} waited for mutex={} over {}ms. "
                    "Owner thread={}, lock acquired at:\n{}",
                    GetCurrentThreadName(),
                    mutex_name_,
                    std::chrono::duration_cast<std::chrono::milliseconds>(timeout).count(),
                    owner_thread_name_,
                    acquire_stacktrace_
                );
                // 发告警到监控系统
                AlertManager::Send("deadlock_detected", mutex_name_);
                // 然后继续阻塞等待（最终还是会死锁，但告警已经发出去了）
                mtx_.lock();
                return;
            }
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
        // 成功获取锁，记录持有者信息
        owner_thread_name_ = GetCurrentThreadName();
        acquire_stacktrace_ = GetStackTrace();
    }
    // ... unlock 清空持有者信息
};
```

部署后不到一周，告警响了。日志里的信息让我瞬间锁定了关键线索：

```
DEADLOCK DETECTED! Thread=Logic waited for mutex=Guild.g_mutex over 5000ms.
Owner thread=Net, lock acquired at:
  net_thread.cpp:184 Guild::BroadcastToMembers()

DEADLOCK DETECTED! Thread=Net waited for mutex=Player[uid=1004237].m_mutex over 5000ms.
Owner thread=Logic, lock acquired at:
  logic_thread.cpp:329 Player::JoinGuildBattle()
```

两条日志互为因果：**网络线程持有了 Guild 的锁，等待 Player 的锁；逻辑线程持有了同一个 Player 的锁，等待 Guild 的锁。** 经典的死锁环。

而且注意——定时器线程也在等 Guild 的锁（`Guild::UpdateRanking`），它被网络线程挡在了外面，是间接受害者。

### 第三步：分析锁获取顺序

有了"谁持有谁等待"的数据，剩下就是翻代码找调用路径。

**逻辑线程**的调用链（公会战开始）：

```
LogicThread::ProcessGuildBattleStart()
  └→ for each member in guild:
       └→ Player::JoinGuildBattle()
            └→ lock(m_mutex)       // ① 先锁 Player
                 └→ Guild::AddBattleParticipant()
                      └→ lock(g_mutex)  // ② 后锁 Guild
```

**网络线程**的调用链（公会消息广播）：

```
NetThread::ProcessGuildMessage()
  └→ Guild::BroadcastToMembers()
       └→ lock(g_mutex)              // ① 先锁 Guild
            └→ for each member:
                 └→ Player::SendPacket()
                      └→ lock(m_mutex)  // ② 后锁 Player
```

两条调用链的锁顺序**完全相反**：

- 逻辑线程：先 `Player.m_mutex` → 后 `Guild.g_mutex`
- 网络线程：先 `Guild.g_mutex` → 后 `Player.m_mutex`

这就是 ABBA 死锁的教科书模板。

### 第四步：找到触发场景

既然锁序不一致，为什么不是每次必死？答案在于**并发窗口极小**：

- 逻辑线程遍历公会成员时，必须在锁住某个 Player 之后、锁 Guild 之前，恰好网络线程已经拿到了 Guild 锁并开始广播到同一个 Player
- 这个窗口只有几微秒，必须在两个线程执行到恰好的代码行时同时发生

这解释了一切：

- **为什么低频**：公会战每周才开启一次，每次触发死锁的概率不到 1%
- **为什么无规律**：取决于两个线程的执行步调是否恰好撞进那个微秒级窗口
- **为什么定时器也死**：`Guild::UpdateRanking()` 也需要 `g_mutex`，网络线程持有它不放，定时器线程就成了陪葬

至此，根因完全清晰。

## 根因分析

### 经典 ABBA 死锁

```
         逻辑线程                          网络线程
            │                                │
            │ lock(Player_A.m_mutex) ✓        │
            │                                │ lock(Guild.g_mutex) ✓
            │ want lock(Guild.g_mutex)       │
            │   ↓ 等待网络线程释放...         │ want lock(Player_A.m_mutex)
            │                                │   ↓ 等待逻辑线程释放...
            │                                │
            ╚══════════ 死锁环 ══════════════╝
```

**时序分解：**

| 时刻 | 逻辑线程 (T1) | 网络线程 (T2) | 状态 |
|------|--------------|--------------|------|
| t₁ | `lock(Player_A.m_mutex)` ✓ | — | T1 持有 Player_A |
| t₂ | 准备锁 Guild… | `lock(Guild.g_mutex)` ✓ | T1 持有 Player_A，T2 持有 Guild |
| t₃ | `lock(Guild.g_mutex)` ⏳ 等待 T2 | `lock(Player_A.m_mutex)` ⏳ 等待 T1 | **死锁形成** |

### 为什么之前没发现

1. **极窄的竞争窗口**：两个线程各自持有第一把锁后，才去获取第二把锁。这个"各自持有第一把锁"的状态只持续微秒级，两个线程必须在这个窗口内同时试图获取对方的锁，概率极低。

2. **低频触发场景**：公会战是一周一次的事件。日常的玩家登录、打怪、交易都不会同时触及这两条调用链。只有公会战开启的瞬间——逻辑线程遍历成员 + 网络线程广播公会消息——才构成必要条件。

3. **测试环境无法复现**：QA 环境单线程跑脚本、或者并发强度不够，永远撞不进这个窗口。本地压测工具模拟 500 人公会战跑了 10000 次都没触发，因为压测工具的网络包和逻辑 tick 是串行的。

### 定时器线程的"误杀"

定时器线程本身没有问题——`Guild::UpdateRanking()` 只锁 `g_mutex`，不会再去锁 `Player.m_mutex`。但它的锁请求被卡在网络线程持有的 `g_mutex` 后面，属于无辜陪葬。三个线程的死锁图实际上是：

```
      逻辑线程 ──持有──→ Player_A.m_mutex
         │                    ↑
       等待                  等待
         ↓                    │
      Guild.g_mutex ←──持有── 网络线程
         ↑
       等待
         │
      定时器线程
```

网络线程是整张图的关键节点，它同时阻塞了逻辑线程和定时器线程。

## 解决方案

方案分四层，从代码到流程全覆盖。

### 1. 统一锁顺序规则（治本）

团队定了一条铁律，写入编码规范文档：

> **任何时候需要同时持有多个锁，必须按「先 Guild 后 Player」的顺序获取。严禁先锁 Player 再锁 Guild。**

改动集中在逻辑线程的 `ProcessGuildBattleStart()`：

```cpp
// ❌ 改前 —— 先 Player 后 Guild，与网络线程锁序相反
void LogicThread::ProcessGuildBattleStart(Guild& guild) {
    for (auto& member : guild.GetMembers()) {
        std::lock_guard<std::mutex> player_lock(member->m_mutex);  // ① Player
        // ... 处理玩家数据 ...
        std::lock_guard<std::mutex> guild_lock(guild.g_mutex);     // ② Guild ⚠️ 顺序反了！
        guild.AddBattleParticipant(member);
    }
}

// ✅ 改后 —— 先 Guild 后 Player，与网络线程锁序一致
void LogicThread::ProcessGuildBattleStart(Guild& guild) {
    // 先锁公会（短临界区，只收集成员列表）
    std::vector<std::shared_ptr<Player>> members;
    {
        std::lock_guard<std::mutex> guild_lock(guild.g_mutex);
        members = guild.GetMembersSnapshot();  // 拷贝成员列表
    }  // Guild 锁释放

    // 再逐个处理玩家
    for (auto& member : members) {
        std::lock_guard<std::mutex> player_lock(member->m_mutex);
        // ... 处理玩家数据 ...
        {
            std::lock_guard<std::mutex> guild_lock(guild.g_mutex);
            guild.AddBattleParticipant(member);
        }
    }
}
```

核心思路：**把锁范围拆小**，避免同时持有两把锁的时间窗口。即使需要同时持有，也保证全局一致的顺序。

### 2. 使用 `std::lock` 原子获取多锁（防御层）

对于确实需要同时持有两把锁的场景，改用 `std::lock` 原子获取：

```cpp
// 同时获取多把锁，std::lock 内部使用 try-and-back-off 算法，避免死锁
std::scoped_lock dual_lock(player.m_mutex, guild.g_mutex);
// 要么同时拿到两把，要么都不拿——永远不会有"持一把等另一把"的中间状态
```

`std::lock` 是 C++11 标准库提供的死锁避免算法（类似 `std::lock` 在内部用 `try_lock` + 回退），保证原子性。而 `std::scoped_lock`（C++17）更是语法糖，直接 RAII 管理多把锁，推荐全项目统一使用。

### 3. CI 接入 ThreadSanitizer（预防层）

在 CI 流水线中加入了 ThreadSanitizer 编译和测试：

```cmake
# CMakeLists.txt —— 新增 TSan 构建类型
if(CMAKE_BUILD_TYPE STREQUAL "Tsan")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=thread -fno-omit-frame-pointer -g")
    set(CMAKE_LINKER_FLAGS "${CMAKE_LINKER_FLAGS} -fsanitize=thread")
endif()
```

CI 流程：
```
代码提交 → 常规编译+单元测试 → TSan 编译+公会战场景压测(30min) → 通过才允许合入
```

TSan 对死锁的检测不那么直接（它更擅长数据竞争），但对锁顺序违规（lock order inversion）有专门的检测器，配合 `TSAN_OPTIONS="report_bugs=1"` 跑公会战场景，能抓到大部分锁序问题。

### 4. 超时告警（兜底层）

保留排查阶段写的 `DeadlockAwareMutex`，但阈值从 5 秒收紧到 **2 秒**，并直接对接到 PagerDuty：

```cpp
// 生产环境配置
constexpr auto LOCK_TIMEOUT = std::chrono::seconds(2);

// 超时触发：打印双端调用栈 + 发送 PagerDuty 告警 + dump 线程状态到文件
mutex_.lock_with_timeout(LOCK_TIMEOUT);
```

即使最坏情况下死锁再次发生，我们也能在 2 秒内知道，而不是等到玩家骂上论坛。

## 效果

改动的上线窗口选在周二凌晨 2 点的例行维护。那个深夜，我守在监控面板前，看着灰度流量从 1% 逐步加到 100%。

| 指标 | 改前（一个月） | 改后（六个月） |
|------|---------------|---------------|
| 死锁次数 | 4-8 次/月 | **0** |
| 因卡死导致的掉线 | 约 80,000 玩家·次/月 | **0** |
| 运维凌晨唤醒次数 | 4-8 次/月 | **0** |
| TapTap 评分 | 3.8 → 3.2（最低） | 回升至 4.1 |

六个自然月的监控数据显示：**零死锁**。

额外收获：ThreadSanitizer 在 CI 中抓住了一个隐蔽的数据竞争——`Player::m_lastLoginTime` 在定时器线程（排行榜计算读取）和逻辑线程（玩家登录写入）之间没有同步保护。在它引发"排行榜分数乱跳"的线上 bug 之前，我们就修掉了。某种意义上，这次死锁排查的投入回报率远超预期。

## 经验教训

### 1. 多锁场景必须定锁序——这是铁律，不是建议

这件事之后，团队编码规范加了一条硬性规定：**任何涉及两把及以上锁的函数，必须在注释中写明锁获取顺序，Code Review 时验证所有调用路径的锁序一致性。** 不再是"大家注意一下"，而是"不写锁序注释的 PR 直接打回"。

### 2. `std::lock` / `std::scoped_lock` 可以原子获取多把锁

C++17 的 `std::scoped_lock` 是避免 ABBA 死锁最轻量的武器。它接受多个 mutex，内部使用死锁避免算法（try-and-back-off）原子获取。团队已将其设为默认推荐：能用 `scoped_lock` 的地方不允许手动逐把 lock。

### 3. 低频并发 bug 最难查——加超时检测和 ThreadSanitizer

这个死锁如果在测试环境复现，可能一天就能查到。偏偏它在本地 10000 次压测下安然无恙，只在线上以 1% 的概率触发。面对这种 bug，**被动等复现是等不起的**。必须主动布防：
- **生产环境**：锁超时检测 + 调用栈打印
- **CI 环境**：ThreadSanitizer / Helgrind 持续扫描

### 4. 代码审查必须检查锁顺序

锁序问题肉眼很难发现——尤其当调用链跨了三个函数和两个类的时候。我们后来在 Code Review Checklist 中加了固定条目：

> - [ ] 本次改动是否引入了新的锁获取路径？
> - [ ] 新路径的锁获取顺序是否与已有路径一致？
> - [ ] 是否存在**先锁 A 后锁 B** 与 **先锁 B 后锁 A** 共存的调用路径？

这三个问题救了我们不止一次。

---

> 后记：那个因为公会战掉线而怒打一星的 TapTap 用户，在问题修复半年后把评分改回了五星，附言只有四个字："终于不卡了。"

---

## 图谱知识点映射

- [操作系统](../mds/1.基础能力/1.3.3.操作系统.md) — 多线程 / 死锁的四个必要条件、ABBA 死锁模型、锁序规则
- [服务端优化](../mds/3.研发能力/3.2.3.服务端优化.md) — 多线程架构下的性能与稳定性权衡、锁粒度设计
- [代码质量管理](../mds/5.管理能力/5.2.1.代码质量管理.md) — Code Review 锁序检查清单、静态分析工具（TSan）接入 CI、编码规范落地
