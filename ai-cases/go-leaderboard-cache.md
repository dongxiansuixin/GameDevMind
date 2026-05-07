# 用 ChatGPT 设计排行榜缓存方案

> **一句话总结：** 排行榜查询从 5 秒超时优化到 8 毫秒，ChatGPT 帮忙对比了 4 种缓存策略，但最终选择「本地缓存 + Redis 二级方案」是我基于业务数据特征做的判断——AI 能铺开选项，取舍要靠人。

---

## 背景

我们游戏的全服排行榜（战力榜、竞技场排名、公会战积分）跑在 Go 服务上，MySQL 5.7 单表 200 万行。随玩家量增长，问题爆发：

- 排行榜查询平均耗时 **5.2 秒**，接口超时率 15%
- MySQL CPU 飙到 **95%**，高峰期数据库连接池耗尽
- 慢查询日志里 `/api/rank/top100` 长期霸榜

`EXPLAIN` 一看：**全表扫描**，没走索引。`ORDER BY score DESC LIMIT 100` 对 200 万行排序，神仙也扛不住。

我把症状丢给了 ChatGPT。

---

## 对话记录

### 第一轮：描述症状，验证方向

> **我：**
> ```
> Go 服务 + MySQL 5.7，排行榜表 200 万行，查询 top100 耗时 5 秒。
> SQL: SELECT user_id, score, rank FROM leaderboard ORDER BY score DESC LIMIT 100
> EXPLAIN 显示 type=ALL, rows=2000000, Using filesort。
> MySQL CPU 95%。怎么优化？
> ```

> **ChatGPT（摘要）：**
>
> 给了清晰的优先级排序：
> 1. **立即止血**：`CREATE INDEX idx_score ON leaderboard(score DESC)` — 把 filesort 变成 index scan
> 2. **加 Redis 缓存**：top100 用 Redis Sorted Set（`ZADD` / `ZREVRANGE`），查询走 Redis，MySQL 只做持久化
> 3. **考虑分页**：top100 不够的话，后续用 `ZREVRANGE ... WITHSCORES`，O(log N)
> 4. **读写分离**：如果还要支持实时写入，考虑 Redis 做写入缓冲 + 异步刷 MySQL
>
> 方向完全正确。我加了索引后查询降到 800ms，加上 Redis Sorted Set 降到 50ms。

**我的感受：** ChatGPT 的方案很标准，像读了一篇「MySQL 排行榜优化」的最佳实践文章。但它没提到一个关键问题：**缓存一致性**。如果玩家积分更新了，Redis 和 MySQL 怎么同步？

---

### 第二轮：追问一致性，ChatGPT 铺开方案矩阵

> **我：**
> ```
> Redis Sorted Set 方案有个问题：玩家积分实时变化（PVP 结算），
> 如果直接用 ZADD 更新 Redis，MySQL 异步写的话，
> 缓存和数据库不一致怎么办？对比一下 Write-Through 和 Cache-Aside 哪种更适合。
> ```

> **ChatGPT（摘要）：**
>
> 给了一个很详细的对比表：

| 策略 | 写入路径 | 读取路径 | 一致性 | 复杂度 |
|------|----------|----------|--------|--------|
| **Cache-Aside** | 更新 DB → 删缓存 | 查缓存 → miss 查 DB → 回填 | 最终一致（有窗口） | ⭐⭐ |
| **Write-Through** | 同时写缓存+DB | 只读缓存 | 强一致（事务内） | ⭐⭐⭐ |
| **Write-Behind** | 写缓存 → 异步刷 DB | 只读缓存 | 弱一致（可能丢数据） | ⭐⭐⭐⭐ |
| **Read-Through** | 写 DB → 缓存自动加载 | 缓存 miss 自动查 DB | 最终一致 | ⭐⭐⭐ |

> ChatGPT 推荐 **Write-Through**（同时写 Redis + MySQL），理由是排行榜对实时性要求高，玩家打完 PVP 立刻要看排名变化，不能忍受 Cache-Aside 的删除-回填窗口。

**我的修正：** 这个分析很好，但我意识到一个问题：我们的排行榜查询 QPS 高达 5000/s，如果每次查 top100 都穿透到 Redis，Redis 的带宽会成为新瓶颈。我需要更进一步。

---

### 第三轮：提出二级缓存方案，ChatGPT 生成框架

> **我：**
> ```
> 我们排行榜查询 QPS 5000+，全走 Redis 的话网络 IO 太高了。
> 我想要一个「本地缓存 + Redis」的二级方案：
> 1. 查询先走本地 in-memory cache（sync.Map），命中直接返回
> 2. 本地 miss 走 Redis
> 3. Redis miss 走 MySQL（理论上不应该发生）
> 4. 本地缓存每 5 秒从 Redis 刷新一次（top100 允许 5 秒延迟）
> 5. 写入用 Write-Through：更新 Redis ZADD + 发 PubSub 通知所有实例刷新本地缓存
> 
> 给我 Go 的代码框架。用 sync.Map + goroutine + Redis PubSub。
> ```

> **ChatGPT（摘要）：**
>
> 生成了约 150 行的代码框架，结构清晰：

```go
type LeaderboardCache struct {
    local      sync.Map           // key: boardType, value: []RankEntry
    redis      *redis.Client
    refreshCh  chan string        // PubSub 通知刷新
    ttl        time.Duration
}

func (c *LeaderboardCache) GetTop100(boardType string) ([]RankEntry, error) {
    // 1. 查本地缓存
    if cached, ok := c.local.Load(boardType); ok {
        return cached.([]RankEntry), nil
    }
    // 2. 本地 miss → Redis
    entries, err := c.redis.ZRevRangeWithScores(ctx, boardType, 0, 99).Result()
    if err != nil {
        return nil, err
    }
    // 3. 回填本地缓存
    c.local.Store(boardType, entries)
    return entries, nil
}

func (c *LeaderboardCache) UpdateScore(boardType, userID string, score float64) {
    // Write-Through: Redis + 通知所有实例
    c.redis.ZAdd(ctx, boardType, &redis.Z{Score: score, Member: userID})
    c.redis.Publish(ctx, "leaderboard:refresh", boardType)
}

func (c *LeaderboardCache) startRefresher() {
    // 每 5 秒全量刷新本地缓存
    // 同时监听 PubSub 做实时失效
}
```

> 还贴心地加了：`sync.RWMutex` 保护写入、优雅关闭 `context.Context`、`singleflight` 防止缓存击穿。

---

## 最终成果

基于 ChatGPT 的框架，我做了以下人工调整后上线：

| 我的调整 | 原因 |
|----------|------|
| `sync.Map` → 自己封装的 `map + sync.RWMutex` | sync.Map 适合读多写极少，我们有定期刷新写入 |
| PubSub 通知改为 `boardType` 级别 | 原方案全量刷新太粗暴，只刷变更的榜单 |
| 加 `singleflight.Group` | 防止缓存 miss 时大量请求同时穿透到 Redis |
| 本地缓存 TTL 改为 3 秒 | 5 秒延迟实测玩家能感知排名滞后 |

上线效果：

| 指标 | 优化前 | 优化后 | 改善 |
|------|--------|--------|------|
| 排行榜查询耗时 | 5,200ms | **8ms** | **-99.8%** |
| MySQL CPU | 95% | 12% | ✅ |
| Redis 网络 IO | — | 可控（本地命中率 97%） | ✅ |
| 接口超时率 | 15% | 0% | ✅ |
| PVP 结算到排名更新延迟 | — | < 3 秒 | ✅ |

---

## 关键收获

### 1. AI 能对比多种方案，但「取舍」是人做的

ChatGPT 第二轮给出的 Write-Through vs Cache-Aside 对比很全面，但它不会主动说「你们的 QPS 高到需要二级缓存」。这个判断来自我对系统负载的理解。

**AI 铺开选项 → 人做架构取舍。** 这是目前为止 AI 协作最清晰的边界。

### 2. 给 AI 的约束越具体，代码越可用

第一轮我只说「排行榜慢」，AI 给了通用方案。第三轮我精确到「QPS 5000+、二级缓存、PubSub 通知、允许 3-5 秒延迟」——AI 生成的代码基本可以直接用，我只做了微调。

### 3. AI 生成的代码细节需要人工复核

ChatGPT 用 `sync.Map` 作为本地缓存是对的，但它没考虑我们「定期全量刷新」的写入模式。`sync.Map` 在写多读少的场景下不如 `RWMutex + map`。这类性能细节，AI 目前还把握不好。

---

## 图谱知识点映射

- [2.2.2 数据库](../mds/2.技术能力/2.2.2.数据库.md) — 索引、Redis Sorted Set、缓存策略
- [1.2.2 数据结构](../mds/1.基础能力/1.2.2.数据结构.md) — Skip List（Redis ZSet 底层）、哈希表
- [1.3.5 计算机网络](../mds/1.基础能力/1.3.5.计算机网络.md) — PubSub 模式、网络 IO 优化
- [1.2.4 代码重构](../mds/1.基础能力/1.2.4.代码重构.md) — 缓存层抽象

> 🏥 相关实战案例：[排行榜查询超时 — 全表扫描 5s 到 8ms 的优化之路](../cases/leaderboard-optimization.md)
