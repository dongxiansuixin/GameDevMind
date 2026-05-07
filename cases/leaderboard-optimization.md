# 排行榜数据库查询优化实战：从 5.2s 到 8ms 的性能逆袭

> **一句话总结：** 卡牌手游运营活动期间，30 万玩家同时在线导致排行榜接口超时率 15%。通过加索引、Redis 缓存、本地二级缓存、游标分页和连接池调优五管齐下，将查询延迟从 5200ms 降到 8ms，MySQL CPU 从 95% 降到 18%。

---

## 背景

我们维护的是一款**卡牌手游**，后端采用 **Go 游戏服务器 + MySQL 8.0**，数据库部署在**阿里云 RDS**（16 核 64G 内存，SSD 存储）。

排行榜系统包含三个核心功能：

| 功能 | 说明 | 数据量级 |
|------|------|----------|
| 全服战力排名 | 所有服务器玩家按战力降序排列 | ~300 万行 |
| 本服排名 | 单服内玩家排名 | ~3 万行/服 |
| 好友排名 | 好友列表内的排名对比 | ~200 行/请求 |

### 问题触发

某次运营活动——**"战力冲刺赛"**，规则很简单：活动结束时（晚 21:00）根据全服战力排名发放限定称号和 SSR 卡牌。活动吸引了大量玩家冲榜，同时在线人数飙升至 **30 万**。

活动当晚 20:55 分开始，运维告警报接连响起：

> *"排行榜接口超时率 15%。"*
> *"RDS CPU 使用率 95%。"*
> *"玩家大量投诉：排行榜打不开、转圈 10 秒不出来。"*

这是一个典型的**读密集型雪崩**——运营活动制造了瞬时并发高峰，而底层数据库没有做好应对准备。

---

## 症状

### 1. 监控面板数据

| 指标 | 正常时段 | 活动高峰 | 恶化倍数 |
|------|----------|----------|----------|
| 全服排行榜 P50 延迟 | 120ms | 5200ms | 43x |
| 全服排行榜 P99 延迟 | 350ms | 8700ms | 25x |
| MySQL CPU 使用率 | 22% | 95% | 4.3x |
| 接口超时率（3s 阈值） | 0.05% | 15% | 300x |
| 排行榜 QPS | 200 | 3500 | 17.5x |

### 2. 慢查询日志（阿里云 RDS 慢查询 top 5）

```sql
# Query_time: 8.7s  Lock_time: 0.0s  Rows_sent: 100  Rows_examined: 3120000
SELECT uid, nickname, power, level, vip_level, guild_name
FROM player_rank
ORDER BY power DESC
LIMIT 100;
```

慢查询日志里全是同一条 SQL——`SELECT * FROM player_rank ORDER BY power DESC LIMIT 100`。每条需要扫描**全表 312 万行**才能返回 100 行结果。

### 3. 业务表现

- 玩家点击"全服排行榜"后白屏 5-8 秒
- 运营活动发奖截止前 5 分钟，高峰期 15% 的请求直接超时
- 大量超时请求堆积导致 Go 协程暴增，内存持续上涨
- 玩家社区出现大量吐槽帖："榜单卡到怀疑人生"

---

## 排查过程

### 第 1 步：EXPLAIN 分析（5分钟定位）

直接在 RDS 控制台执行 EXPLAIN：

```sql
EXPLAIN SELECT uid, nickname, power, level, vip_level, guild_name
FROM player_rank
ORDER BY power DESC
LIMIT 100;
```

结果：

| id | select_type | table | type | possible_keys | key | rows | Extra |
|----|-------------|-------|------|---------------|-----|------|-------|
| 1 | SIMPLE | player_rank | **ALL** | NULL | NULL | **3,120,537** | **Using filesort** |

关键信息：

- **type=ALL**：全表扫描。这已经是最高级别的红色警报，意味着 MySQL 在逐行读取 300 万行数据。
- **rows=3,120,537**：预估扫描行数。实际 `Rows_examined` 完全一致。
- **Using filesort**：因为 `ORDER BY power`，但 power 字段上没有索引，MySQL 只能把所有数据读入内存后排序。当数据量超过 `sort_buffer_size` 时，还需要在磁盘上做归并排序（外部排序），IO 开销极大。

**结论**：执行计划本身就是灾难，先看索引。

### 第 2 步：检查索引（关键发现）

```sql
SHOW INDEX FROM player_rank;
```

结果：

| Table | Key_name | Column_name |
|-------|----------|-------------|
| player_rank | PRIMARY | uid |
| player_rank | idx_level | level |
| player_rank | idx_server | server_id |

**power（战力）字段上没有索引！**

经询问 DBA 确认：`player_rank` 表是项目初期由后端开发直接建的，DBA 后来审阅时只加了 `level` 和 `server_id` 索引（用于本服排行和等级过滤），但**遗漏了全服排行榜最核心的 `power` 字段索引**。

这是一次**流程漏洞**——建表语句未经 DBA final review 就上线了。平常 QPS 低时 MySQL 硬扛还能撑住，但运营活动一来就原型毕露。

### 第 3 步：加索引后的效果（不够理想）

紧急添加降序索引：

```sql
CREATE INDEX idx_power_desc ON player_rank(power DESC);
```

再次 EXPLAIN：

| id | type | possible_keys | key | rows | Extra |
|----|------|---------------|-----|------|-------|
| 1 | index | idx_power_desc | idx_power_desc | 100 | **Using index** |

- type 从 `ALL` 变成 `index`
- rows 从 312 万变成 100（直接用索引顺序读取，无需扫描全表）
- Extra 从 `Using filesort` 变成 `Using index`（覆盖索引，连回表都省了）

**查询延迟从 5.2s → 0.8s**，降了 6.5 倍。但 **0.8 秒仍然远超目标（要求 P99 < 100ms）**。

为什么呢？因为：
- 虽然走了索引，但每次请求仍需查询 MySQL（磁盘 IO + 网络往返）
- 3500 QPS 的并发打在同一个索引上，B+ 树的根节点和中间节点成为热点
- RDS 的 Buffer Pool 虽然能缓存大部分数据，但高并发下的锁竞争仍在消耗 CPU

**索引解决了"查询本身慢"的问题，但没有解决"请求太多"的问题。**

### 第 4 步：分析访问模式（关键洞察）

进一步分析业务日志发现：

```sql
-- 统计每小时 Top100 的人员变动
SELECT COUNT(*) FROM (
    SELECT uid FROM player_rank ORDER BY power DESC LIMIT 100
) AS top100_current
WHERE uid NOT IN (
    SELECT uid FROM rank_snapshot WHERE snapshot_time = DATE_SUB(NOW(), INTERVAL 1 HOUR)
);
```

结果：**每小时 Top100 中只有 10-20 人的排名发生变化，大多数时间 Top 10 完全不动。**

这是因为卡牌手游的战力增长比较缓慢——顶级玩家已经接近天花板，每次更新带来的战力提升最多几十点，远不足以撼动头部排名。

**这意味着 99% 的排行榜查询在返回完全相同的结果。每次查询都从 MySQL 全量排序再取 Top100，是巨大的浪费。**

**核心洞察：排行榜是一个"读多写极少"的典型场景，缓存收益极高。**

---

## 根因分析

### 鱼骨图分析

```
                    无索引（full scan）          无缓存层
                         │                        │
                    ┌────▼────────────────────────▼──────┐
                    │                                    │
                    │    ┌──────────────────────────┐    │
  导火索 ──────────►│    │  排行榜接口超时率 15%     │◄───┘
                    │    │  P99 = 8.7s              │
  运营活动          │    └──────────────────────────┘
  瞬时并发 3500QPS  │                                    │
                    └────────────────────────────────────┘
```

### 三层根因

| 层级 | 根因 | 占比 | 说明 |
|------|------|------|------|
| **主因** | power 字段无索引，全表扫描 | ~80% | 5.2s 的瓶颈核心都在这。加上索引后降到 0.8s，但还不够。 |
| **放大器** | 无缓存层，每个请求直接打到 DB | ~15% | 3500 QPS × 312 万行扫描 = 灾难。即使有索引，DB 仍是瓶颈。 |
| **导火索** | 运营活动造成瞬时并发高峰 | ~5% | 活动设计（集中发奖）制造了流量尖刺，暴露了架构短板。 |

### 为什么之前没暴露？

1. **日常 QPS 只有 200-300**，全表扫描虽然慢但请求少，平均延迟还能勉强维持在 120ms
2. **测试环境玩家数量只有几百人**，`EXPLAIN` 显示 `rows` 也只有几百行，掩盖了全表扫描的本质
3. **没有做运营活动压测**——这是最大的流程缺陷。运营活动上线前只测了功能正确性，没测性能极限

---

## 解决方案

### 方案总览

采用**多层缓存 + 索引优化 + 连接池调优**的组合策略，从外到内形成**本地缓存 → Redis → MySQL**的三级读体系：

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│ 本地内存缓存 │────►│ Redis 缓存  │────►│ MySQL (RDS)  │
│ (1s 过期)   │     │ (10s 过期)  │     │ (持久层)     │
└─────────────┘     └─────────────┘     └──────────────┘
   命中率: 85%+        命中率: 99%+        命中率: 100%
   延迟: <1ms          延迟: <3ms           延迟: ~1ms
                                            (走索引后)
```

### 1. 加索引（治本——消除全表扫描）

```sql
-- 降序索引，覆盖 ORDER BY power DESC 查询
CREATE INDEX idx_power_desc ON player_rank(power DESC);

-- 必要时扩展为覆盖索引，避免回表
-- CREATE INDEX idx_power_desc_cover ON player_rank(power DESC, uid, nickname, level, vip_level, guild_name);
```

**效果**：从 5.2s → 0.8s，降幅 85%。

### 2. Redis 缓存（核心优化——拦截 99% 请求）

在 Go 服务中引入 Redis 作为排行榜缓存层：

```go
// Redis 中存储 Top200 排行榜（比请求多的 100 多缓存一倍，避免边界问题）
// 使用 Sorted Set：member=uid，score=power

// 排行榜刷新定时器（每 10 秒从 MySQL 刷新一次）
func RefreshLeaderboardCache(ctx context.Context) error {
    rows, err := db.QueryContext(ctx,
        "SELECT uid, power FROM player_rank ORDER BY power DESC LIMIT 200")
    if err != nil {
        return err
    }
    defer rows.Close()

    pipe := redis.Pipeline()
    pipe.Del(ctx, "leaderboard:top200")
    for rows.Next() {
        var uid string
        var power int64
        rows.Scan(&uid, &power)
        pipe.ZAdd(ctx, "leaderboard:top200", redis.Z{Score: float64(power), Member: uid})
    }
    pipe.Expire(ctx, "leaderboard:top200", 15*time.Second) // 过期时间略长于刷新间隔
    _, err = pipe.Exec(ctx)
    return err
}

// 查询排行榜（直接从 Redis 取）
func GetTopPlayers(ctx context.Context, limit int) ([]PlayerRank, error) {
    // ZREVRANGE 按 score 降序取 Top N
    results, err := redis.ZRevRangeWithScores(ctx, "leaderboard:top200", 0, int64(limit-1)).Result()
    if err != nil {
        return nil, err
    }
    // 转换结果...
}
```

**设计要点**：
- **缓存 Top200 而非 Top100**：留有余量，防止分页请求超出缓存范围
- **10 秒刷新**：平衡数据新鲜度和 DB 压力。排行榜数据变动慢，10 秒延迟完全可接受
- **主动刷新而非被动失效**：避免缓存击穿（缓存过期瞬间的大量并发打到 DB）

### 3. 本地内存缓存（减少 Redis 网络往返）

在 Go 进程内存中再加一层极短过期的本地缓存：

```go
// 使用 sync.Map + 时间戳实现简单本地缓存
type LocalRankCache struct {
    mu       sync.RWMutex
    data     []PlayerRank
    expireAt time.Time
}

func (c *LocalRankCache) Get() ([]PlayerRank, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    if time.Now().Before(c.expireAt) && c.data != nil {
        return c.data, true
    }
    return nil, false
}

func (c *LocalRankCache) Set(ranks []PlayerRank) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data = ranks
    c.expireAt = time.Now().Add(1 * time.Second) // 1 秒过期
}
```

**1 秒过期带来的效果**：
- 同一秒内的重复请求**完全不走网络**
- 在 3500 QPS 场景下，假设请求均匀分布，本地缓存命中率可达 **85%+**
- 大大减轻了 Redis 的压力（Redis QPS 从 3500 降到 ~500）

### 4. 分页优化：游标分页替代 OFFSET

原实现使用 LIMIT + OFFSET 做分页：

```sql
-- 第 2 页：OFFSET 100，MySQL 仍需扫描并跳过前 100 行
SELECT * FROM player_rank ORDER BY power DESC LIMIT 100 OFFSET 100;
-- OFFSET 10000 时：MySQL 需要扫描并丢弃 10100 行！
```

改为**游标分页（Keyset Pagination）**：

```sql
-- 前端传入上一页最后一条的 power 值
SELECT * FROM player_rank
WHERE power < @last_power   -- 从上一页末尾继续
ORDER BY power DESC
LIMIT 100;
```

配合索引 `idx_power_desc`，每次查询只扫描 100 行，O(1) 复杂度。即使翻到第 100 页也毫秒级返回。

### 5. 连接池调优

```go
// Go sql.DB 连接池配置
db.SetMaxOpenConns(200)       // 从 50 扩到 200
db.SetMaxIdleConns(50)        // 从 10 扩到 50
db.SetConnMaxLifetime(30 * time.Minute)
db.SetConnMaxIdleTime(5 * time.Minute)
```

**为什么扩连接池？**
- 索引优化后单次查询变快，连接周转率提高，但瞬时并发请求仍需排队
- 原来 50 个连接在 3500 QPS 下，每个请求平均占连接 14ms——如果 P99 延迟达到 800ms，连接池会在高并发下耗尽
- 扩到 200 个连接，即使部分请求变慢也有足够的缓冲

---

## 效果

### 上线后对比

| 指标 | 改前 | 改后 | 提升 |
|------|------|------|------|
| 排行榜查询延迟 (avg) | 5200ms | **8ms** | **650x** |
| 排行榜查询延迟 (P99) | 8700ms | **22ms** | **395x** |
| MySQL CPU 使用率 | 95% | **18%** | ↓ 81% |
| 接口超时率 | 15% | **0.01%** | ↓ 1500x |
| 并发支持 (QPS) | 3000 | **50000+** | **16.7x** |
| Redis QPS | 0（未使用） | ~500 | — |
| 本地缓存命中率 | 0（未使用） | **87%** | — |

### 架构变更前后对比

```
改前:
  玩家 ──► Go Server ──► MySQL (每请求都全表扫描 312 万行)
  3500 QPS × 全表扫描 = 灾难

改后:
  玩家 ──► Go Server ──► 本地内存 (命中 87%, <1ms)
                │
                └──────► Redis (命中 99%+, <3ms)
                              │
                              └──► MySQL (命中 <1%, ~1ms, 走索引)
  三级缓存逐层拦截，MySQL 只承受 ~35 QPS
```

---

## 经验教训

### 1. 排行榜是典型的"读多写少"场景，缓存收益极高

排行榜数据变化频率低（前 100 名每小时只变 10-20 人），但读请求量巨大。这种场景下，缓存是**性价比最高的优化手段**。10 秒的缓存延迟对玩家体验几乎无影响，但换来了 650 倍的性能提升。

> **设计原则**：遇到读多写少的场景，第一反应就应该是"加缓存"，而不是"优化 SQL"。

### 2. EXPLAIN 是排查第一步——type=ALL 直接报警

如果 DBA 或开发在最初就执行了 `EXPLAIN`，`type=ALL` 和 `rows=312 万` 会立即暴露问题。事实上，这个问题**本可以在上线前就被发现**。

> **流程教训**：建表语句必须做 EXPLAIN 验证；代码审查应包含执行计划检查。

### 3. 多层缓存比单层更稳

很多人一上来就只加 Redis，但在高并发场景下，Redis 本身也可能成为瓶颈（单实例 Redis 10 万 QPS 也扛不住极端流量）。

三层设计的好处：
- **本地内存**：零网络开销，极限性能
- **Redis**：跨进程共享，多实例一致
- **MySQL**：兜底保底，数据永远正确

每一层都是上一层的"安全气囊"。

### 4. 运营活动前必须做压测——血的教训

本次活动上线前的测试缺陷：

| 缺失项 | 后果 |
|--------|------|
| 未做排行榜接口压测 | 不知道 3000 QPS 下会雪崩 |
| 测试环境数据量太小（几百条） | EXPLAIN 显示 rows=几百，掩盖全表扫描 |
| 未模拟瞬时并发尖刺 | 缓存穿透、连接池耗尽等问题全部隐藏 |
| DBA 未参与性能评审 | power 字段漏建索引未被发现 |

> **强制流程**：任何运营活动上线前，必须有运维/后端/DBA 三方签字确认的**性能压测报告**。

### 5. 分页用游标而非 OFFSET

`OFFSET` 在大偏移量下效率极低——MySQL 必须扫描并丢弃前面所有行才能定位到起始位置。游标分页（Keyset Pagination）始终保持 O(1) 复杂度。

不过游标分页的代价是**不能跳页**，这在排行榜场景中其实不是问题——很少有玩家会跳到第 500 页看排名。

### 6. 其他补充

- **监控先行**：慢查询日志和 RDS 性能洞察是这次快速定位问题的关键。如果没有这些，可能还在抓瞎。
- **覆盖索引锦上添花**：如果查询的字段都能从索引中拿到（覆盖索引），MySQL 连回表都省了，延迟还能再往下压。
- **降级策略**：建议在极端情况下（Redis 挂了），自动返回"排行榜暂不可用，请稍后再试"，而不是把流量直接打到 MySQL。

---

## 图谱知识点映射

- [数据库](../mds/2.技术能力/2.2.2.数据库.md) — MySQL 索引优化、EXPLAIN 分析、慢查询排查
- [数据存储](../mds/6.运营能力/6.1.2.数据存储.md) — Redis 缓存策略、多层缓存架构
- [数据统计分析](../mds/6.运营能力/6.2.2.数据统计分析.md) — 排行榜数据特征分析、访问模式挖掘
