***

title: 分布式缓存设计取舍与 JetCache 落地全景
tags: \[redis, jetcache, cache, architecture, 高并发, swr, singleflight]
date: 2026-09-02
source: ai-conversation
-----------------------

> 本文档系统性梳理分布式缓存的所有关键设计取舍、SWR/单飞等机制的利弊对比，
> 并完整对接 JetCache 框架的设计要素与配置点，最终给出电子元器件商城场景的落地决策。
>
> 背景：在「没有统一缓存框架」的项目中，面临自研 vs 用 JetCache 的选型，来回拷问过的所有设计点
> 均在此归档，目标是完全吃透分布式缓存的设计底层。

***

## 零、总纲：分布式缓存没有银弹，核心是 8 个维度的取舍矩阵

每一个缓存策略决策，都需要在下面 8 个维度上做 trade-off：

| 维度          | 含义                    | 衡量方式                |
| ----------- | --------------------- | ------------------- |
| **一致性窗口**   | 用户读到旧值的最长持续时间         | TTL、广播时延、锁超时        |
| **后端压力**    | 缓存失效瞬时打 DB/ES/下游的 QPS | 击穿次数、穿透次数           |
| **资源开销**    | 内存、线程、定时器、连接池成本       | 堆占用、活跃线程数、调度次数      |
| **代码复杂度**   | 团队能不能写对、后续谁维护         | 代码行数、bug 数/月、新人上手时间 |
| **故障爆炸半径**  | 出现 bug 时影响面有多大        | 影响的接口数、能否快速降级       |
| **可观测性**    | 出问题时排查速度              | 定位根因所需时间（分钟级？小时级？）  |
| **扩展性**     | 未来加 SWR/熔断/批量合并的成本    | 扩展策略的代码改动量          |
| **业务语义契合度** | 是不是符合具体数据的"允许旧值程度"    | 业务方是否投诉、资损金额        |

> 不存在 8 项全优的方案。所有的争论都是在为自己看重的 2\~3 项辩护。

***

## 一、缓存刷新机制的两种范式：定时器 vs SWR（访问驱动）

这是我们讨论的起点，也是最容易被误用的设计点。

### 1.1 两种范式的定义

| 项目        | 定时器模式（JetCache @CacheRefresh）     | SWR 模式（Stale-While-Revalidate）              |
| --------- | --------------------------------- | ------------------------------------------- |
| **核心思路**  | 为每个被访问过的 key 创建一个固定间隔的定时任务，到点重算   | **不创建任何常驻调度器**，仅在访问时判断剩余 TTL，达到阈值时异步刷新并返回旧值 |
| **刷新触发器** | 时间到了就刷，不管有没有人访问                   | **必须有访问才触发**，冷 key 永远零成本                    |
| **资源模型**  | O(Nkey) 个 ScheduledFuture，key 多必炸 | O(1) 常驻，只需要一个异步线程池                          |
| **用户阻塞**  | 无（后台线程刷）                          | 无（返回旧值 + 异步刷）                               |

### 1.2 定时器模式（@CacheRefresh）的取舍

#### ✅ 好处

1. **数据新鲜度最强**：只要 key 在刷新周期内，数据永远是最新的，用户不会感知过期
2. **缓存击穿防护完美**：热点 key 在过期前就被后台刷新，不存在瞬时冷启动
3. **分布式去重内置**：`BOTH`/`REMOTE` 下自动用分布式锁保证集群中只有一台机器刷新
4. **实现简单**：几行 scheduleAtFixedRate 代码

#### ❌ 坏处

1. **key 数量爆炸 = 死局**：每个 key 一个 ScheduledFuture + ThreadPool 调度，百万级 key 直接 OOM 或调度器瘫痪
2. **冷 key 浪费资源**：一个 key 只被访问一次，定时器仍会跑 N 次直到 `stopRefreshAfterLastAccess` 触发
3. **创建-停止抖动**：冷 key 反复被访问→停，反复创建和取消 `ScheduledFuture`

#### 🎯 适用场景判断（3 条全中才用）

- key 数量**极少且可枚举**（< 5000）

- 热点极度集中，**过期瞬间的击穿不可接受**

- 数据新鲜度要求高，不能接受 stale read

#### 🔧 JetCache 对接点

```java
@CacheRefresh(
    refresh = 1800,                          // 刷新间隔
    stopRefreshAfterLastAccess = 3600,       // 多久没访问就停刷（必设！不然一直跑）
    refreshLockTimeout = 60,                 // 分布式锁超时（方法慢就调大）
    timeUnit = TimeUnit.SECONDS
)
```

### 1.3 SWR 模式（访问驱动异步刷新）的取舍

#### ✅ 好处

1. **与 key 数量完全解耦**：1000w SKU 也只需一个 8\~32 线程的异步刷新池，零常驻 per-key 状态
2. **冷 key 零开销**：不访问 → 不判断 → 不消耗任何资源
3. **完美匹配长尾分布**：Pareto 分布天然适合——热点 key 反复触发刷新保持新鲜，冷 key 自然过期
4. **扩展性好**：CF 模式的 inflightMap 可以自然承接熔断、限流、批量合并等后续策略

#### ❌ 坏处

1. **新鲜度弱于定时器**：冷 key 过期后第一次访问必定 miss，直到下一个访问才触发刷新
2. **实现复杂度高**：要处理阈值判断、分布式刷新去重锁、刷新失败重试/退避、线程池隔离、超时取消
3. **需要补 JetCache 一层**：JetCache 原生不支持，需写 AOP 或 CacheDecorator，代码复杂度从 0 → 200\~500 行
4. **stale read 窗口**：刷新期间所有返回的都是旧值，如果刷新慢（DB 2s），2s 内所有用户都看到旧数据

#### 🎯 适用场景判断（满足 3 条以上就用）

- **key 多且长尾**（百万/千万级）

- 数据允许短时间旧值（几分钟\~几小时无业务损失）

- 热点相对集中，过期瞬间的击穿有体感

- 团队能 hold 住 inflightMap + 分布式锁的代码

#### 🔧 JetCache 对接点：需要补一层 SWR AOP 自定义注解

见附录 §B 完整代码。

***

## 二、单飞（Single-Flight / 防击穿）机制的取舍

> 单飞 = 同一个 key 并发未命中时，只允许 1 个线程回源，其余等待结果。

### 2.1 单飞到底要不要开？默认开启的 5 个坑

JetCache 的 `@CachePenetrationProtect` 默认**不开启**，原因是以下 5 个副作用：

| 坑                        | 不开时的表现              | 开了单飞后的表现                                                   |
| ------------------------ | ------------------- | ---------------------------------------------------------- |
| **每未命中多一层锁操作**           | miss → 直接查 DB（3 步）  | miss → inflightMap CAS + park/unpark（≥7 步）。**低命中率场景反而负收益** |
| **回源线程卡死 = 雪崩放大器**       | 100 线程各查各的，慢是慢但互不阻塞 | 1 个线程卡 DB，后面 99 个一起 hang，线程池迅速耗尽 → 服务假死                    |
| **嵌套调用 = 死锁**            | 递归 → 栈溢出（秒级重现，好排查）  | A 拿锁等内调用，内调用等 A 释放锁 → 死锁 hang 住，隐蔽难排                       |
| **监控指标失真**               | 回源次数 = 实际查库次数       | 回源次数=1，但 P99 延迟被等待拉高，打开监控看到一切正常却感觉慢                        |
| **语义与 @CacheRefresh 冲突** | 各管各的                | 后台刷新线程和请求线程都要回源，谁算"第一个"？逻辑变复杂                              |

**结论**：不要全局开，只在**确定的热点方法**上单独加 `@CachePenetrationProtect(timeout=3)`。

### 2.2 单飞的两种实现模式对比

| 维度               | synchronized / monitor wait 模式（JetCache 现在用的）        | ConcurrentHashMap + CompletableFuture 模式（标准 singleflight 实现）               |
| ---------------- | ---------------------------------------------------- | -------------------------------------------------------------------------- |
| **同步场景下的延迟/QPS** | 低并发下略优（JVM monitor 优化极强）                             | 基本持平，高并发大 key 时 CF 稳定性略好                                                   |
| **同步场景下线程模型**    | 等待线程 OS 级 wait                                       | `future.join()` 也 OS 级阻塞                                                   |
| **超时控制**         | 代码糙（`while + wait(deadline - now)` + 虚假唤醒处理，容易写错）    | 一行 `future.get(3, SECONDS)`，天然支持超时 + 取消                                    |
| **异常传播**         | 手动 `try/catch/notifyAll`，容易漏（漏了其他线程永远 wait）          | `completeExceptionally()` 统一广播，不会漏                                         |
| **可观测性（最关键差异）**  | jstack 看到 `WAITING on monitor <地址>`，**无法定位卡在哪个 key** | inflightMap 直接暴露为 JMX / actuator，一行看到「卡在 sku:10086 已 1282ms」               |
| **内存泄漏风险**       | 几乎为零（JVM 自动管理 monitor）                               | 回源线程抛 Error 没 catch → CF 永不 complete → inflightMap 条目永久驻留。必须配「最老条目 TTL 清理」 |
| **惊群效应**         | monitor 公平队列，逐个唤醒，CPU 平稳                             | 200 线程 `.get()` + complete 瞬间全部被唤醒，P99 抖动                                  |
| **SWR 扩展融合度**    | 需要另写一套去重机制（同步单飞用 monitor，异步刷新用 Redis 锁双份逻辑）          | **共享同一个 inflightMap**，单飞和刷新去重天然统一，后续策略随便拼                                  |
| **代码量 / 心智负担**   | 30\~50 行，快糙猛                                         | 100\~150 行，要处理清理、异常、超时、map 容量控制                                            |

### 2.3 选型三问（直接拍板）

| 问题                                      | 是 → CF 模式  | 否 → synchronized 模式 / 用 JetCache 自带 |
| --------------------------------------- | ---------- | ----------------------------------- |
| 线上出现过「大量 WAITING 线程不知道卡在哪」的事故？          | ✅ 可观测性救命   | ❌ 还没踩过坑就凑合用                         |
| 还要做 SWR / 熔断 / 批量合并等扩展策略？               | ✅ CF 是扩展地基 | ❌ 只要单飞功能 → 用现成的                     |
| 团队能 hold 住 inflightMap 的清理、异常传播、超时回滚细节？ | ✅ 写得对就上    | ⚠️ 写不对会内存泄漏 → 用 synchronized 更稳     |

***

## 三、多级缓存（L1 本地 + L2 Redis）一致性的取舍

> 缓存类型三选一：`LOCAL` / `REMOTE` / `BOTH`。`BOTH` 用了就必须处理 L1 一致性。

### 3.1 L1 本地缓存的四种段位策略

| 段位                                                                               | 做法                              | 一致性窗口                                     | 适用场景         | 实现成本   |
| -------------------------------------------------------------------------------- | ------------------------------- | ----------------------------------------- | ------------ | ------ |
| **段位 1：TTL 兜底**短 `localExpire`（5min），改数据只删 L2，L1 等过期                             | `localExpire` 分钟级 \~ 小时级        | 商品图文详情、评论数、参数表                            | 0            | <br /> |
| **段位 2：写时主动失效广播**JetCache `broadcastChannel`（Redis Pub/Sub），改数据时通知所有节点删 L1       | 正常即时（毫秒级广播），Pub/Sub 丢消息时退化到 TTL | 标题、店铺信息、分类字典**中厂最常用，90% 场景够**             | 1 行配置        | <br /> |
| **段位 3：版本号校验 + 广播双保险**读 L1 时先查 Redis 的 version key（`HGET sku_ver 123`），不一致就失效 L1 | 最多 1 次请求读到旧值，下一次必纠正             | 展示价、库存展示（非扣减）、优惠券可用数**价格敏感 + 不允许 5 分钟窗口** | 中等（缓存值封装版本号） | <br /> |
| **段位 4：强一致 L1**etcd/ZK watch + ACK 回执，失败重试                                       | 理论零窗口，但复杂度过高，写延迟数倍              | 全行业几乎不用，性价比极低                             | 极高，不推荐       | <br /> |

### 3.2 什么时候不该用 BOTH？（常被忽略）

1. **key 量级超过 JVM 承受力**（1000w SKU × 1KB = 10GB，塞不下） → 只用 `REMOTE`
2. **本地命中率 < 1%**（用户维度数据，key 分散） → BOTH 白加复杂度，没有收益
3. **强一致场景（库存、结算价、支付状态）** → L1 哪怕 30 秒的窗口都可能资损，强制 `REMOTE` + 超短 TTL（30s 内）+ 主动失效

### 3.3 本地缓存装不下的折中方案：Tiny L1 顶流缓存

不要 `localLimit=1000000`，改为：

```yaml
jetcache.local.default.limit: 2000   # 只放最热的 2000 条
```

- 容量小 → GC 压力小，哪怕 50 个 JVM 实例各存 2000 条，重复存储也才 10w 份（100MB 总开销）

- 命中的都是真正的顶流 SKU，收益比 BOTH 全放高得多

### 3.4 🔧 JetCache 对接点

| 设计点         | JetCache 配置 / 注解                                                 |
| ----------- | ---------------------------------------------------------------- |
| 二级缓存启用      | `cacheType = CacheType.BOTH`                                     |
| L1 大小上限     | `localLimit = 2000`（注解级）/ `jetcache.local.default.limit`（全局）     |
| L1 独立过期时间   | `localExpire = 300`（通常 < expire，减少不一致窗口）                         |
| 跨节点 L1 广播失效 | `jetcache.remote.default.broadcastChannel: xxx`                  |
| L1 类型       | `jetcache.local.default.type: caffeine`（推荐，默认 linkedhashmap 性能差） |
| L1 Key 转换器  | `jetcache.local.default.keyConvertor: fastjson2`                 |

***

## 四、穿透 / 击穿 / 雪崩三防体系

三种现象经常被混为一谈，解法完全不同：

| 现象       | 定义                                                  | 根因                    | 解法                                                            | JetCache 对应能力                              |
| -------- | --------------------------------------------------- | --------------------- | ------------------------------------------------------------- | ------------------------------------------ |
| **缓存穿透** | 查一个**不存在的 key**，每次都打到 DB                            | 恶意爬虫（不存在的 SKU ID）、脏数据 | ① Bloom Filter 先过滤② `cacheNullValue=true` 缓存空值                | `@Cached(cacheNullValue=true)` + 布隆过滤器（自加） |
| **缓存击穿** | **单个热点 key 过期瞬时**，大量并发打 DB                          | 热门商品详情、首页推荐           | ① 单飞（`@CachePenetrationProtect`）② SWR 提前异步刷新（过期无感）            | `@CachePenetrationProtect` + 自定义 SWR AOP   |
| **缓存雪崩** | **一批 key 同时过期**（比如 TTL 都是 1h，12 点整上线时一起写入，13 点全体失效） | 批量加载、相同 TTL 未加随机抖动    | TTL ± 10% 随机抖动（`expire = 3600 + random(0, 360)`）分层过期（本地短、远程长） | JetCache 不内置，需在自定义 CacheLoader 里补抖动        |

### 三种防护的组合拳（推荐默认组合）

```
请求路径：
  ├── Bloom Filter（如果 key 有不存在的风险）
  │     └── 不通过 → 直接返回空，不碰 Redis / DB
  ├── L1（Caffeine，tiny limit=2000）
  │     ├── 命中 → 返回（最快）
  │     └── miss → 查 L2
  ├── L2（Redis）
  │     ├── 命中 + 剩余TTL充足 → 返回
  │     ├── 命中 + 剩余TTL ≤ 20% → 返回旧值 + SWR异步刷新（分布式锁去重）
  │     └── miss → 单飞保护回源
  └── 单飞回源（@CachePenetrationProtect + timeout=3s）
        └── 查到 → 写回 L2 + L1 + 缓存null（如果是null）
```

***

## 五、电商场景（电子元器件商城）的逐场景决策表

> 电子元器件商城特征：1000w SKU、高长尾、型号同义替换、实时库存/阶梯价强一致、BOM 配单。

| 数据场景                            | Key 量级               | 热点集中度                    | 旧值容忍度        | 缓存类型                 | 刷新策略                                              | 单飞             | L1           | 关键备注                  |
| ------------------------------- | -------------------- | ------------------------ | ------------ | -------------------- | ------------------------------------------------- | -------------- | ------------ | --------------------- |
| **商品详情**（型号、参数、PDF、封装图）         | 1000w 级              | 极高（头部 1% 占 90% PV）       | ✅ 几小时        | BOTH（Tiny L1）        | **SWR：expire=24h，阈值=80%**                         | ✅ 加 timeout=3s | limit=2000   | 最需要 SWR 的场景           |
| **搜索聚合结果**（分类页、品牌筛选、参数筛选）       | 几十万组合                | 高（热门搜索词、热门分类）            | ✅ 几小时        | REMOTE               | **SWR：expire=30min，阈值=50%**                       | ✅              | ❌ 不开         | key 组合多，本地命中率极低       |
| **首页推荐 / 活动会场 / Banner**        | <1000                | 极端集中                     | ✅ 几分钟        | BOTH                 | **@CacheRefresh**（key 少不炸）refresh=30min + stop=1h | ✅              | ✅            | key 少，定时器完美           |
| **展示价（划线价 / 市场价）**              | \~SKU 数              | 高                        | ✅ 几分钟        | REMOTE               | **SWR：expire=10min，阈值=70%**                       | ✅              | ❌            | 只展示不结算                |
| **实时库存（可售数 + 在途）**              | SKU × 仓库≈ 5000w      | 极高                       | ❌ 0 容忍       | **REMOTE 强制**        | **无 SWR，无定时**TTL=30s + 扣减/入库时主动删                  | ✅ timeout=2s   | ❌ 严禁 L1（会超卖） | 结算前必须再查实时             |
| **结算阶梯价**（1~~9/10~~99/100+）     | SKU × 档级≈ 3000w      | 高                        | ❌ 0 容忍       | **REMOTE 强制**        | TTL=30s + 改价时主动删                                  | ✅ timeout=2s   | ❌ 严禁 L1      | 结算价必须实时               |
| **品牌 / 分类 / 参数字典**              | <1w                  | 中                        | ✅ 几小时        | LOCAL                | `@CacheRefresh refresh=1hstop=24h`                | ❌ 没必要          | ✅            | key 极少，直接本地+定时        |
| **型号同义词映射**（STM32F103 → 标准 SKU） | 百万级                  | 中高（热门型号）                 | ✅ 几天         | REMOTE               | \*\*SWR：expire=7d，阈值=90%\*\*或者 TTL 即可             | ❌              | ✅ 小容量        | 词典数据，更新极低频            |
| **用户订单 / 收藏 / 购物车**             | 用户量级百万级              | 极低（单用户一天几次）              | 订单 ❌其他 ✅ 几分钟 | REMOTE（订单短 TTL=5min） | TTL 纯过期，订单状态变更时主动删                                | ❌              | ❌ 不开         | 用户 key 分散，SWR/单飞全是负收益 |
| **BOM 配单整单报价**                  | 低频 key每次 BOM hash 不同 | 低（同一份 BOM 可能重复提交 3\~5 次） | ✅ 5min 内     | REMOTE               | TTL=5min，天然过期                                     | ❌              | ❌            | 没必要 SWR               |
| **会员等级 / 权益**                   | 用户量                  | 低                        | ❌ 充值后要立刻看到   | REMOTE               | TTL=5min + 变更时双删缓存                                | ❌              | ✅ 限容量        | 绝不能 stale 读           |
| **PDF / 规格书文件元信息**              | \~SKU 数              | 中                        | ✅ 几天         | BOTH                 | 超长 TTL=7d + 更新时 `@CacheInvalidate`                | ❌              | ✅            | 极少变动                  |

***

## 六、JetCache 设计要素全量对接与配置清单

### 6.1 注解能力总表（方法级 API）

| 注解                           | 作用             | 关键属性                                                                                                                                                                          | 重要默认值陷阱                                                                                                        |
| ---------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **@Cached**                  | 启用方法级缓存        | `name`（key前缀/统计名）、`key`（SpEL）、`expire`/`timeUnit`、`cacheType`、`localLimit`、`localExpire`、`serialPolicy`、`keyConvertor`、`cacheNullValue`、`condition`、`postCondition`、`enabled` | `cacheType=REMOTE`（容易误以为默认 BOTH）`expire` 未定义 + 全局也未定义 = 无穷大（约68年！务必显式设）`cacheNullValue=false`（不缓存 null = 穿透风险） |
| **@CacheUpdate**             | 更新缓存           | `name`（要和 @Cached 一致）、`key`、`value`（SpEL 指定新值）、`condition`                                                                                                                    | 远程操作可能失败（网络/Redis OOM），所以一定要配 expire 兜底                                                                        |
| **@CacheInvalidate**         | 删除缓存元素         | `name`、`key`、`multi`（批量删）、`condition`                                                                                                                                         | 同上，失败靠 expire 兜底                                                                                               |
| **@CacheRefresh**            | 定时器自动刷新        | `refresh`、`stopRefreshAfterLastAccess`、`refreshLockTimeout`、`timeUnit`                                                                                                        | **没有任何保护 key 数爆炸的机制**，滥用的唯一止损是 `stopRefreshAfterLastAccess`                                                    |
| **@CachePenetrationProtect** | 单飞防击穿          | `value`（默认 true）、`timeout`（等待上限，**默认无限等，必设**）、`timeUnit`                                                                                                                      | 只在单 JVM 生效，分布式下 N 台机器最坏 N 次回源                                                                                  |
| **@CreateCache**             | 声明式创建 Cache 实例 | 和 @Cached 类似，但是字段/API 级别                                                                                                                                                      | 需要 `@EnableCreateCacheAnnotation`                                                                              |

### 6.2 全局配置清单（yml / GlobalCacheConfig）

```yaml
jetcache:
  # ===== 基础行为 =====
  statIntervalMinutes: 15               # 统计输出间隔（0=关闭），必开，观察击穿/命中率靠这个
  areaInCacheName: false                # false = 远程key不拼area名，key更短（推荐）
  penetrationProtect: false             # 【强烈推荐】全局关，按方法单独开
  enableMethodCache: true               # 是否启用方法缓存（SpringBoot自动true）
  hiddenPackages: xxx.xxx               # AOP拦截时从堆栈里剥掉的包，让key生成更短，多包用逗号隔

  # ===== L1 本地缓存配置（支持多area） =====
  local:
    default:
      type: caffeine                    # caffeine / linkedhashmap。必须改 caffeine
      limit: 2000                       # L1 容量（elements），别设大
      keyConvertor: fastjson2           # fastjson2 / jackson / jackson3 / none
      expireAfterWriteInMillis: 3600000 # 全局默认本地过期，方法级可覆盖
      expireAfterAccessInMillis: 0      # 访问后延迟过期，0=关闭（一般不需要开）

  # ===== L2 远程缓存配置 =====
  remote:
    default:
      type: redis.lettuce               # redis（jedis）/ redis.lettuce（推荐，支持异步）
      keyConvertor: fastjson2
      valueEncoder: kryo                # java / kryo / kryo5（推荐kryo，小且快）
      valueDecoder: kryo
      expireAfterWriteInMillis: 86400000 # 全局默认远程过期，务必方法级覆盖
      broadcastChannel: ele-comp-cache-channel  # 【关键】L1跨节点失效广播，不配置BOTH更新有坑
      keyPrefix: 'ec:'                  # 可选：所有Redis key前缀，多项目共用一个Redis集群时必设

      # ===== Redis 连接池 =====
      poolConfig:
        minIdle: 10
        maxIdle: 50
        maxTotal: 100

      # ===== Redis 寻址（三选一） =====
      uri: redis://127.0.0.1:6379/      # 单机
      # uri: redis-sentinel://h1,h2,h3/?sentinelMasterId=mymaster  # 哨兵
      # mode: masterslave
      # cluster:                          # 集群
      #   - redis://c1:7000
      #   - redis://c2:7001
      #   - redis://c3:7002
```

**Java Config 对应（非 Spring Boot 场景）：**

```java
@Bean
public GlobalCacheConfig config() {
    GlobalCacheConfig cfg = new GlobalCacheConfig();

    Map<String, CacheBuilder> localBuilders = new HashMap<>();
    localBuilders.put("default", CaffeineCacheBuilder.createCaffeineCacheBuilder()
        .keyConvertor(Fastjson2KeyConvertor.INSTANCE)
        .limit(2000)
        .expireAfterWrite(1, TimeUnit.HOURS));

    Map<String, CacheBuilder> remoteBuilders = new HashMap<>();
    remoteBuilders.put("default", RedisLettuceCacheBuilder.createRedisLettuceCacheBuilder()
        .keyConvertor(Fastjson2KeyConvertor.INSTANCE)
        .valueEncoder(KryoValueEncoder.INSTANCE)
        .valueDecoder(KryoValueDecoder.INSTANCE)
        .redisUri("redis://127.0.0.1:6379/")
        .broadcastChannel("ele-comp-cache-channel")
        .expireAfterWrite(24, TimeUnit.HOURS));

    cfg.setLocalCacheBuilders(localBuilders);
    cfg.setRemoteCacheBuilders(remoteBuilders);
    cfg.setStatIntervalMinutes(15);
    cfg.setAreaInCacheName(false);
    cfg.setPenetrationProtect(false);   // 全局关
    return cfg;
}
```

### 6.3 JetCache 未内置、需要手动补的 5 个关键能力

| 能力                    | 为什么 JetCache 不做                    | 推荐补法                                                    | 预计代码量      |
| --------------------- | ---------------------------------- | ------------------------------------------------------- | ---------- |
| **SWR 访问驱动异步刷新**      | 注解框架不做具体策略判断，留给用户组合                | `@SwrRefresh` 自定义注解 + AOP + Redis SETNX 分布式刷新锁          | 200\~300 行 |
| **分布式单飞**             | `@CachePenetrationProtect` 只做单 JVM | 在 miss 时用 `SETNX lock:{key} NX EX 3` 包一层（SWR 代码里已经顺便做了） | 50 行       |
| **TTL 随机抖动防雪崩**       | 框架无法知道业务接受的抖动范围                    | 自定义 `CacheLoader` 装饰器，`load()` 返回时 TTL ± 10% 随机         | 30 行       |
| **Bloom Filter 防穿透**  | 和业务数据耦合（要知道全量合法 key 集合）            | Guava BloomFilter 启动加载，拦截器层先过滤                          | 100 行      |
| **缓存统计接入 Prometheus** | 只打日志不上报                            | 自定义 `CacheMonitor`，把 stats 转为 Micrometer gauges         | 100 行      |

***

## 七、自研 vs JetCache + 少量补充的决策终判

### 7.1 工作量与 ROI 对比

| 模块                        | 全自研（人日）             | JetCache（人日）       |
| ------------------------- | ------------------- | ------------------ |
| 方法级注解 AOP + SpEL key      | 15\~20              | 0                  |
| L1（Caffeine）+ L2（Redis）二级 | 10\~15              | 0                  |
| 跨节点 L1 Pub/Sub 失效广播       | 5\~8                | 0（一行配置）            |
| 多种序列化切换（Java/Kryo/JSON）   | 3\~5                | 0                  |
| 统计 + 监控埋点 + JMX           | 5\~8                | 0                  |
| Spring Boot Starter 自动装配  | 3\~5                | 0                  |
| 单飞防击穿                     | 3\~5                | 0                  |
| SWR 访问驱动刷新                | 8\~12               | **3\~5（需补一个 AOP）** |
| **合计**                    | **69~~96（3~~5 人月）** | **3\~5 人日**        |

### 7.2 应该全自研的唯一条件（3 条全中才考虑）

1. **公司 ≥2 个业务线**需要复用同一套缓存框架，后续还会扩展
2. **SWR 必须做到 Cache 实现层内嵌**（而不是 AOP 层多一次读 Redis 的开销）
3. **6 个月内**要做批量合并、预热队列、跨机房多活、灰度 TTL、熔断降级联动等高级策略

**不满足以上条件 → JetCache + 200 行 SWR 切面 = 最优解**。原因：

- 失败爆炸半径极小：SWR 代码写崩了，关掉注解就回落纯 TTL 模式，零事故

- 代码正确性由社区背书：注解 AOP、二级缓存、广播这些难写对的部分，JetCache 已经跑了 8 年

***

## 八、最终落地路线图（电子元器件商城）

```
Week 1: JetCache 基础接入
  ├─ 引入 starter + yml 配置（broadcastChannel 必开，L1 caffeine limit=2000）
  ├─ 字典表 / 分类 / 品牌 → @Cached + @CacheRefresh（key 极少，定时器稳）
  ├─ 库存 / 结算价 → @CacheInvalidate + TTL=30s + REMOTE 强制
  └─ 订单 / 用户数据 → 纯 TTL REMOTE，变更时主动删
  ✅ 覆盖率：60% 缓存场景，0 自研代码

Week 2: SWR AOP 接入（重点）
  ├─ 写 @SwrRefresh 注解 + Aspect（附录 B 代码可用）
  ├─ 商品详情 → @Cached(expire=24h, BOTH, localLimit=2000) + @SwrRefresh(0.8) + @CachePenetrationProtect(timeout=3)
  ├─ 搜索聚合结果 → @Cached(expire=30min, REMOTE) + @SwrRefresh(0.5) + 单飞
  └─ 展示价 → @Cached(expire=10min, REMOTE) + @SwrRefresh(0.7) + 单飞
  ✅ 核心高 QPS 场景全覆盖，增量代码 ~300 行

Week 3~4: 观测 + 微调
  ├─ JetCache stat 日志：看各 name 的 hitRate、loadCount
  ├─ APM（SkyWalking/Pinpoint）：看 DB QPS 降幅、P99 变化
  ├─ 如果 SWR 刷新队列老是满 → 调大 refreshPool
  ├─ 如果发现某 SKU 每次刷新都超时 → 排查慢查询
  └─ 可选：Prometheus Grafana Dashboard 接缓存统计

Month 2: 高级优化（按需）
  ├─ Bloom Filter 防不存在的 SKU ID 穿透（Guava BF，启动加载全量合法 SKU ID）
  ├─ 首页 / 会场 → 预热脚本（活动前把热点 key 全跑一遍）
  ├─ 缓存统计转 Micrometer，接入 Prometheus + Grafana 告警
  └─ TTL 随机抖动装饰器（防止活动页批量加载导致的集体过期）
```

***

## 附录 A：关键判断速查口诀

```
───── 缓存策略口诀 ─────
热点聚合走 SWR，
用户维度纯 TTL，
交易数据实时查，
静态配置定时器。

───── SWR 价值 4 问 ─────
热点集中吗？ key 多吗？
旧值能忍吗？ 回源贵吗？
≥3 个是 → 必开 SWR

───── 单飞实现选型 3 问 ─────
卡过不知道卡哪？
要做 SWR/熔断吗？
团队能写对 CF？
≥2 个是 → CF 模式
否则 → JetCache 自带 synchronizedLoad
```

***

## 附录 B：SWR 自定义注解 + AOP 完整参考代码

### B.1 @SwrRefresh 注解

```java
package com.xxx.ware.component.cache.swr;

import java.lang.annotation.*;

/**
 * SWR（Stale-While-Revalidate）访问驱动刷新注解。
 * <p>
 * 语义：当缓存剩余 TTL < expire * (1 - expireRatio) 时，
 * 立即返回旧值给调用方，同时异步提交刷新任务并通过分布式锁保证集群中仅一台刷新。
 * <p>
 * 必须和 @Cached 配合使用。@Cached 的 expire 必须在注解上显式设置（不能依赖全局默认）。
 *
 * @author cache-team
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface SwrRefresh {

    /** 触发刷新的剩余比例阈值。0.8 = 剩余 TTL ≤ expire × 20% 时触发异步刷新 */
    double expireRatio() default 0.8;

    /** 分布式刷新锁超时（秒）。如果刷新很慢，适当调大；防止刷新慢时锁被其他节点提前抢到 */
    int lockTimeoutSec() default 10;

    /** 同步读取（miss + 单飞）时的等待超时（秒）。防止回源卡死，所有等待线程陪葬 */
    int syncWaitTimeoutSec() default 3;
}
```

### B.2 SwrRefreshAspect 切面

```java
package com.xxx.ware.component.cache.swr;

import com.alicp.jetcache.Cache;
import com.alicp.jetcache.CacheGetResult;
import com.alicp.jetcache.anno.Cached;
import com.alicp.jetcache.anno.support.CacheContext;
import com.alicp.jetcache.anno.support.CacheInvokeConfig;
import com.alicp.jetcache.anno.support.ConfigProvider;
import com.google.common.util.concurrent.ThreadFactoryBuilder;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.expression.EvaluationContext;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;

import javax.annotation.PreDestroy;
import java.lang.reflect.Method;
import java.util.concurrent.*;

/**
 * SWR 切面。执行顺序：
 *   SwrRefreshAspect（先执行，做剩余TTL判断）
 *     → JetCacheInterceptor（执行 @Cached 查缓存/回源）
 *         → 实际业务方法
 *
 * 【关键注意】
 * 1. @Order 必须比 JetCacheInterceptor 更先（更小），或直接用 around 手动判断
 * 2. 刷新线程池使用 DiscardPolicy：队列满时直接丢弃任务，下次访问自然再触发，SWR 天然重试
 * 3. 分布式锁用 SETNX + 短 TTL，只有拿到锁的节点才真实刷新，杜绝 N 节点重复刷新
 * 4. inflightMap 只存"正在刷新的 key"，刷新完成立即移除，容量 ≈ refreshPool.coreSize
 */
@Aspect
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
public class SwrRefreshAspect {

    private final StringRedisTemplate redis;
    private final ConfigProvider configProvider;

    /** 正在刷新的任务 key → Future，用于本节点内去重（防止同一个节点提交多个刷新任务） */
    private final ConcurrentHashMap<String, CompletableFuture<Void>> inflight = new ConcurrentHashMap<>();

    /** 异步刷新专用线程池 —— 隔离，不占用业务 Tomcat / ForkJoin 线程 */
    private final ThreadPoolExecutor refreshPool = new ThreadPoolExecutor(
            8, 32,
            5L, TimeUnit.MINUTES,
            new ArrayBlockingQueue<>(5000),
            new ThreadFactoryBuilder().setNameFormat("swr-refresh-%d").setDaemon(true).build(),
            new ThreadPoolExecutor.DiscardPolicy()   // 关键：满了直接丢，下次访问再触发
    );

    private final ExpressionParser spelParser = new SpelExpressionParser();

    @Autowired
    public SwrRefreshAspect(StringRedisTemplate redis, ConfigProvider configProvider) {
        this.redis = redis;
        this.configProvider = configProvider;
    }

    @PreDestroy
    public void shutdown() {
        refreshPool.shutdownNow();
    }

    @Around("@annotation(swr)")
    public Object around(ProceedingJoinPoint pjp, SwrRefresh swr) throws Throwable {
        Method method = ((MethodSignature) pjp.getSignature()).getMethod();
        Cached cachedAnn = method.getAnnotation(Cached.class);
        if (cachedAnn == null) {
            // 没有 @Cached 配对，直接透传
            return pjp.proceed();
        }

        // 1. 从 JetCache 全局配置中取 Cache 实例
        String area = cachedAnn.area().isEmpty() ? "default" : cachedAnn.area();
        String cacheName = buildCacheName(cachedAnn, method);
        CacheInvokeConfig cic = CacheInvokeConfig.getByMethodInfo(cacheName, area);
        Cache<Object, Object> cache = configProvider.getCacheManager()
                .getCache(area, cic == null ? cacheName : cic.getCacheName());

        // 2. 生成 key（@Cached 注解的 SpEL，和 JetCache 用同一套解析）
        Object key = resolveKey(cachedAnn.key(), pjp.getArgs(), method);

        // 3. 正常读缓存（L1+L2 统一走 JetCache）
        CacheGetResult<Object> result = cache.GET(key);
        long expireSec = cachedAnn.expire();
        long thresholdSec = (long) (expireSec * (1 - swr.expireRatio()));

        if (!result.isSuccess()) {
            // 缓存完全 miss —— 交给 JetCache 自己的机制（含 @CachePenetrationProtect 单飞）
            return pjp.proceed();
        }

        // 4. 命中但剩余 TTL <= 阈值 → SWR 触发
        long remainingMs = result.getRemainingTtl();
        if (remainingMs > 0 && remainingMs / 1000 <= thresholdSec) {
            submitRefreshAsync(pjp, key, cacheName, swr);
        }

        // 5. 无论是否触发刷新，都立即返回当前值（SWR 核心：用户零阻塞）
        return result.getValue();
    }

    // ================== 私有辅助方法 ==================

    private void submitRefreshAsync(ProceedingJoinPoint pjp, Object key, String cacheName, SwrRefresh swr) {
        String lockKey = "swr:lock:" + cacheName + ":" + key;

        CompletableFuture<Void> existing = inflight.get(lockKey);
        if (existing != null && !existing.isDone()) {
            return; // 本节点已经有在刷的任务，去重
        }

        CompletableFuture<Void> newF = new CompletableFuture<>();
        existing = inflight.putIfAbsent(lockKey, newF);
        if (existing != null && !existing.isDone()) {
            return; // CAS 失败，另一个线程刚提交
        }

        // 先抢分布式锁（跨节点去重）—— NX + TTL
        Boolean locked = false;
        try {
            locked = redis.opsForValue().setIfAbsent(
                    lockKey, "1", swr.lockTimeoutSec(), TimeUnit.SECONDS);
        } catch (Exception e) {
            // Redis 连不上就算了，不阻塞主流程，最多就是本节点不刷这次
            inflight.remove(lockKey, newF);
            return;
        }

        if (!Boolean.TRUE.equals(locked)) {
            inflight.remove(lockKey, newF);  // 另一个节点在刷，本节点什么都不做
            return;
        }

        final CompletableFuture<Void> finalF = newF;
        refreshPool.submit(() -> {
            try {
                // 刷新 = 重新执行原方法（JetCache 会自动写回 L1 + L2）
                Object newValue = pjp.proceed();
                // 注：pjp.proceed() 本身会走 JetCacheInterceptor 从而触发 cache.PUT
                // 如果 key/方法执行逻辑正常，缓存已经被 JetCache 自动更新，无需手动 PUT
                finalF.complete(null);
            } catch (Throwable t) {
                finalF.completeExceptionally(t);
                // 异常不抛到主流程，刷新失败下次访问自然再触发
            } finally {
                inflight.remove(lockKey, finalF);
                try { redis.delete(lockKey); } catch (Exception ignore) { /* TTL 兜底 */ }
            }
        });
    }

    private String buildCacheName(Cached cached, Method method) {
        if (!cached.name().isEmpty()) {
            return cached.name();
        }
        // JetCache 默认自动生成策略：类名.方法名
        return method.getDeclaringClass().getSimpleName() + "." + method.getName();
    }

    private Object resolveKey(String keySpel, Object[] args, Method method) {
        if (keySpel == null || keySpel.isEmpty()) {
            // 未指定 key：JetCache 默认用所有参数拼接
            // 这里简化实现，实际生产建议复用 JetCache 内部的 KeyConvertor + ParameterInfo
            StringBuilder sb = new StringBuilder();
            for (Object arg : args) {
                if (sb.length() > 0) sb.append('_');
                sb.append(arg == null ? "null" : arg.toString());
            }
            return sb.toString();
        }
        EvaluationContext ctx = new StandardEvaluationContext();
        String[] paramNames = getParamNames(method);
        for (int i = 0; i < args.length; i++) {
            ctx.setVariable(paramNames[i], args[i]);
            ctx.setVariable("args[" + i + "]", args[i]);
        }
        return spelParser.parseExpression(keySpel).getValue(ctx);
    }

    private String[] getParamNames(Method method) {
        // 生产环境：使用 Spring 的 ParameterNameDiscoverer，
        // 或直接用 org.springframework.core.DefaultParameterNameDiscoverer（需 -parameters 编译参数）
        return new DefaultParameterNameDiscoverer().getParameterNames(method);
    }
}
```

### B.3 用法示例

```java
@Service
public class SkuServiceImpl implements SkuService {

    /**
     * 商品详情 —— SWR 标准用法：
     * - 24h 硬过期（兜底，极端冷门自然淘汰）
     * - 剩余 ≤ 4.8h（80% 阈值）触发异步刷新，用户无感
     * - miss 时单飞（3s 超时止损）
     * - Tiny L1（最多 2000 条）+ 跨节点广播
     */
    @Cached(
            name = "sku:",
            key = "#skuId",
            expire = 86400,
            cacheType = CacheType.BOTH,
            localExpire = 1800,
            localLimit = 2000,
            serialPolicy = SerialPolicy.KRYO,
            cacheNullValue = true
    )
    @CachePenetrationProtect(timeout = 3)
    @SwrRefresh(expireRatio = 0.8, lockTimeoutSec = 15)
    @Override
    public SkuDetail getSkuDetail(long skuId) {
        // 正常写业务：只有真正 miss 时才会走到这里
        return skuMapper.selectDetailById(skuId);
    }

    /**
     * 更新商品 —— 必须配套 @CacheInvalidate，否则靠 TTL 等 24h
     * JetCache 通过 broadcastChannel 广播给所有节点删 L1
     */
    @CacheInvalidate(name = "sku:", key = "#sku.skuId")
    @Override
    public void updateSku(SkuUpdateDTO sku) {
        skuMapper.updateById(sku);
    }
}
```

***

## 附录 C：常见故障排查速查

| 现象                                    | 第一排查点                                                       | 第二排查点                                        |
| ------------------------------------- | ----------------------------------------------------------- | -------------------------------------------- |
| P99 飙高，jstack 大量线程 WAITING on monitor | 加的是 `@CachePenetrationProtect` 没设 timeout → 某回源卡死           | 嵌套调用死锁（方法内部又调同 key 方法）                       |
| 改了数据，A 节点看到新值，B 节点还是旧值                | `broadcastChannel` 没配置 → B 节点 L1 没被通知                       | Redis Pub/Sub 网络抖动丢消息 → 等 `localExpire` 自然过期 |
| DB QPS 周期性飙升，和 expire 值一致             | 未启用 SWR，一批热点 key 同步过期 → 击穿                                  | 未加 TTL 随机抖动 → 所有 key 在同一时刻写入，整批同时过期          |
| 内存持续上涨，jmap 看到很多 `ScheduledFuture` 实例 | `@CacheRefresh` 滥用 → key 太多，没设 `stopRefreshAfterLastAccess` | 改用 SWR 模式替代定时器                               |
| JetCache 统计里 loadCount=1 实际还是很慢       | 单飞保护了 DB，但 P99 = 第一个线程查库 + 其他线程等待之和                         | 设 `timeout=3`，超时自己去跑，不要无限等                   |
| 改了 @CacheInvalidate Redis key 还在      | `name` 和 `area` 必须和 @Cached 完全一致（字符串严格相等）                   | 远程操作失败（Redis 超时/满）→ 靠 expire 兜底，监控要有 alert   |
| SWR 刷新队列经常 Discard                    | 热点 key 太多或 refresh 很慢 → 调大 pool 或优化 DB 查询                   | 正常现象：DiscardPolicy 不影响主流程，只是本次不刷，下次访问再触发     |
| 活动 0 点 DB 被打爆                         | 活动页缓存没预热 + SWR 阈值太靠近过期点                                     | 活动前预热脚本 + SWR 阈值调到 90%（更早开始刷）                |

***

## 参考资料

- [JetCache 官方文档 · 方法缓存](https://github.com/alibaba/jetcache/blob/master/docs/CN/MethodCache.md)

- [JetCache 官方文档 · 配置说明](https://github.com/alibaba/jetcache/blob/master/docs/CN/Config.md)

- Caffeine `refreshAfterWrite` 设计说明（SWR 本地模式参考）

- RFC 5861 HTTP Cache-Control: stale-while-revalidate（SWR 标准语义来源）

- Go `singleflight.Group` 源码（CF 模式标准实现参考）

