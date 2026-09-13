---
title: JetCache 使用教程
date: 2026-08-14
tags: [java, spring, springboot, cache, jetcache, redis, caffeine]
source: knowledge
---

# JetCache 使用教程

> 面向使用者的落地手册。读者画像：Java/Spring Cloud 后端工程师，要把 JetCache 接进一个**高并发搜索类接口**项目（详情页、商品资料、替代料、推荐料等），TTL 在 2~10 分钟量级。
>
> 读完后你应该能：独立完成接入、选对序列化器与缓存类型、避开 90% 的生产坑、知道什么时候该自研补位。

---

## 1. 它是什么，和 Spring Cache 比有什么不同

JetCache 是阿里开源的 **Java 缓存访问框架**。注意定语：**框架**，不是缓存实现。它本身不存数据，底层还是 Caffeine / Redis / Tair，它的价值在「抽象层」。

与 Spring Cache 的关键差异：

| 维度 | Spring Cache | JetCache |
|---|---|---|
| 注解 | `@Cacheable` / `@CacheEvict` / `@CachePut` | `@Cached` / `@CacheInvalidate` / `@CacheUpdate` / `@CacheRefresh` |
| TTL 支持 | ❌ 注解层无原生 TTL，依赖 CacheManager 配置 | ✅ 注解上直接写 `expire` |
| 多级缓存 | ❌ 需自己组合 `CompositeCacheManager`，一致性自己搞 | ✅ `cacheType=BOTH` 开箱即用 |
| 集群本地缓存一致性 | ❌ 无 | ✅ `syncLocal=true`，基于 Redis Pub/Sub 广播失效 |
| 防击穿（单飞） | ❌ 无 | ✅ `@CachePenetrationProtect` |
| 自动刷新 | ❌ 无 | ✅ `@CacheRefresh` |
| 缓存统计 | 弱 | ✅ 内置 `CacheMonitor`，可对接 Actuator |

一句话：**Spring Cache 是「能用」，JetCache 是「好用且生产级」。** 搜索接口这种高 QPS、强一致敏感的场景，JetCache 明显更合适。

> GitHub：https://github.com/alibaba/jetcache （截至 2026-08 主线版本 2.7.x 稳定，2.8 要求 JDK17+）

---

## 2. 接入：Spring Boot Starter 三步走

### 2.1 选 Starter

按你用的 Redis 客户端选（**强烈建议 Lettuce**，Spring Boot 默认就是它，且 JetCache 对它的批量命令优化做得最好）：

```xml
<!-- 推荐：Lettuce 版 -->
<dependency>
    <groupId>com.alicp.jetcache</groupId>
    <artifactId>jetcache-starter-redis-lettuce</artifactId>
    <version>2.7.5</version>
</dependency>

<!-- 备选：Jedis 版 -->
<dependency>
    <groupId>com.alicp.jetcache</groupId>
    <artifactId>jetcache-starter-redis</artifactId>
    <version>2.7.5</version>
</dependency>
```

> Caffeine 是本地缓存实现，starter 已传递依赖，无需单独引。Spring Boot 3.x 用 2.7.4+；Spring Boot 2.x 用 2.6.x。

### 2.2 开启注解 + 基础配置

启动类：

```java
@SpringBootApplication
@EnableMethodCache(basePackages = "com.yourcompany")   // 开启 @Cached 等方法注解
@EnableCreateCacheAnnotation                            // 开启 @CreateCache（编程式 API，可选）
public class App { }
```

`application.yml`（最小可用配置）：

```yaml
jetcache:
  statIntervalMinutes: 15          # 每 15 分钟打印一次缓存统计
  areaInCacheName: false           # key 里是否带 area 前缀，2.7 默认 false

  local:
    default:                       # 默认本地缓存区
      type: caffeine
      keyConvertor: fastjson2      # 复杂对象 key 转字符串
      limit: 100                   # 每个 cacheName 本地最多 100 条
      expireAfterWriteInMillis: 30000   # 本地默认 30s 过期

  remote:
    default:                       # 默认远程缓存区
      type: redis
      keyConvertor: fastjson2
      valueEncoder: java           # ⚠️ 见 §3，建议换 jackson
      valueDecoder: java
      expireAfterWriteInMillis: 600000  # 远程默认 10 分钟
      redisHost: 127.0.0.1
      redisPort: 6379
      # redisPassword: xxx
      # redisSentinels: ...        # 哨兵
      # redisClusterNodes: ...     # 集群
```

### 2.3 第一段代码

```java
@Service
public class ProductDetailService {

    @Cached(name = "product:detail:", key = "SpEL.productId",
            expire = 600, cacheType = CacheType.BOTH,
            localLimit = 200, localExpire = 30)
    public ProductDetail getDetail(Long productId) {
        return productRepo.findById(productId);  // 缓存未命中时走这里
    }

    @CacheInvalidate(name = "product:detail:", key = "SpEL.productId")
    public void evict(Long productId) { }
}
```

跑起来，第一次走 DB，第二次直接命中。接入完成。

---

## 3. 序列化器：决定你能不能安稳睡觉

这是 JetCache 最容易被忽略、却最影响生产稳定性的配置。L2（Redis）存的是字节，**反序列化失败 = 缓存变废品**。

### 3.1 四种内置序列化器对比

| 序列化器 | 类型信息 | 删字段 | 加字段 | 改字段类型 | 改包名/删类 | 性能 |
|---|---|---|---|---|---|---|
| `java` (JDK) | 强（类全名） | ✅ | ✅ | ❌ 炸 | ❌ ClassNotFound | 慢、体积大 |
| `kryo5` | 弱 | ✅ | ✅ | ❌ 炸 | ❌ | 快、紧凑 |
| `fastjson2` | 无（默认） | ✅ 忽略 | ✅ | ❌ 炸 | ✅ 不依赖类名 | 快 |
| `jackson` | 无 | ✅ 忽略 | ✅ | ❌ 炸 | ✅ 不依赖类名 | 中 |

**生产首选 `jackson` 或 `fastjson2`**，原因：

1. 不在字节里写类全限定名 → **改包路径、删类不会 ClassNotFound**；
2. 未知字段忽略 → **删字段、加字段向后兼容**；
3. 唯一炸点：**改字段类型**（如 `String→Long`），但这种需求本来就该用新 cacheName 版本号规避。

### 3.2 切换为 Jackson

```yaml
jetcache:
  remote:
    default:
      valueEncoder: com.alicp.jetcache.support.JacksonValueEncoder
      valueDecoder: com.alicp.jetcache.support.JacksonValueDecoder
```

> 2.7.0+ 才内置 Jackson 编解码器，且默认不注册（要显式指定类名）。2.7.0 以下需自己实现 `ValueEncoder`/`ValueDecoder`。

### 3.3 反序列化失败时会发生什么

**重要：JetCache 会吞掉异常，不会让接口 500。** 流程：

```
GET → 反序列化抛异常 → catch → 记 ERROR 日志 → 返回 CacheGetResult(FAIL)
     ↓
用 @Cached / computeIfAbsent → FAIL 被当作"未命中" → 走 loader 回源 → PUT 覆盖脏数据
     ↓
下一次请求：拿到的是新值 ✅ 自愈
```

**但有两个边角要警惕**（详见第 5 节踩坑）：

- **回源也失败**时（DB 异常），脏数据不会被清理，持续打 DB；
- **批量结构变更**时，N 条老数据各自"自愈一次"，瞬间打爆 DB（自愈型雪崩）。

---

## 4. 四种典型用法

### 4.1 注解式：`@Cached`（最常用）

```java
@Cached(name = "rec:", expire = 300, cacheType = CacheType.REMOTE)
@CachePenetrationProtect                        // 单飞，防击穿
@CacheRefresh(refresh = 240, stopRefreshAfterLastAccess = 600)  // 后台异步刷新
public List<Sku> recommend(Long userId) { ... }
```

| 注解属性 | 含义 |
|---|---|
| `name` | cacheName，作为 key 前缀，**改它 = 换一套缓存**（版本号技巧） |
| `key` | SpEL 表达式，省略则用全部参数拼 |
| `expire` / `localExpire` | 远程 / 本地 TTL（秒），`localExpire` 通常 < `expire` |
| `cacheType` | `LOCAL` / `REMOTE` / `BOTH` |
| `localLimit` | 本地缓存最大条数（LRU 淘汰） |
| `cacheNullValue` | 是否缓存 null（防穿透），默认 false |
| `condition` | SpEL，返回 false 时**不读不写缓存**（如只缓存特定品类） |

### 4.2 失效与更新

```java
@CacheInvalidate(name = "product:detail:", key = "SpEL.productId")
public void update(Long productId) { ... }

@CacheUpdate(name = "product:detail:", key = "SpEL.product.id", value = "SpEL.product")
public ProductDetail updateAndReturn(ProductDetail product) { ... }
```

> `@CacheUpdate` 要求方法**返回更新后的对象**，JetCache 会把返回值写回缓存。若方法返回 void，则用 `value` 指定的 SpEL 取参。

### 4.3 编程式 API：`@CreateCache`（2.7 已标 `@Deprecated`，仍可用）

```java
@CreateCache(name = "user:", expire = 600, cacheType = CacheType.BOTH, localExpire = 60)
private Cache<Long, User> userCache;

public User get(Long id) {
    return userCache.computeIfAbsent(id, this::loadFromDb);
}
```

### 4.4 推荐：`CacheManager.getOrCreateCache`（2.7+ 官方推荐）

```java
@Configuration
public class CacheConfig {
    @Bean
    public Cache<Long, ProductDetail> productDetailCache(CacheManager cm) {
        QuickConfig cfg = QuickConfig.newBuilder("product:detail:")
                .expire(Duration.ofMinutes(10))
                .cacheType(CacheType.BOTH)
                .syncLocal(true)                 // ⭐ 多实例本地缓存一致性广播
                .localExpire(Duration.ofSeconds(30))
                .localLimit(500)
                .penetrationProtect()            // ⭐ 单飞防击穿
                .build();
        return cm.getOrCreateCache(cfg);
    }
}
```

注入后直接用：`productDetailCache.computeIfAbsent(id, loader)`。

**强烈建议用这种方式**：声明式、可复用、配置集中、便于挂 Monitor。

---

### 4.5 提前刷新 vs 单飞：别混淆（高频误区）

这两个特性都"减少回源"，但**解决的是完全不同的问题，不能互相替代**。这是 JetCache 最容易被误解的点。

#### 一句话区分

| 特性 | 触发时机 | 谁承担延迟 | 解决的问题 |
|---|---|---|---|
| **单飞** `@CachePenetrationProtect` | key **已过期后**，并发请求来时 | **用户请求**（同步等 loader） | 防 DB 被打挂（N 并发只回源 1 次） |
| **提前刷新** `@CacheRefresh` | key **还没过期**时，后台主动 | **后台线程**（用户无感） | 消除过期瞬间的延迟毛刺 |

> **单飞是"出了事少死点人"，提前刷新是"根本不出事"。**

#### 用场景说明为什么单飞替代不了刷新

假设详情接口：TTL=10 分钟，DB 查询 200ms，单机 QPS 2000。

**只开单飞，key 过期瞬间：**

```
T=10:00:00  key 过期，1000 个并发请求同时 miss
            → 单飞：1 个请求查 DB（200ms），其余 999 个 await 等待
T=10:00:00.2  DB 返回 → 1000 个请求同时放行
```

**结果**：DB 没被打挂（单飞功劳），但这 1000 个请求**每个都等了 200ms**，监控曲线上是一个 P99 尖峰。

**开了提前刷新（refresh=8 分钟）：**

```
T=08:00:00  key 写��，TTL=10min
T=08:08:00  后台线程主动刷新 → 覆盖并重置 TTL，用户全程无感
T=08:18:00  再次后台刷新 ...
            key 永远不会"过期到需要用户同步等"
```

**结果**：用户永远拿到热数据，P99 平如直线。代价是后台周期性多查几次 DB。

#### 决策表：到底要不要开提前刷新

| 你的场景 | 单飞够吗 | 提前刷新 |
|---|---|---|
| 详情页（核心交易路径，要求 P99 < 50ms） | 不够 | ✅ 强烈建议开 |
| 替代料、推荐料（边缘功能，可容忍偶尔慢） | ✅ 够 | 可不开 |
| 商品资料（变更低频、TTL 长） | ✅ 够 | 可不开 |
| 实时性要求高（库存、价格） | — | ❌ 别用缓存刷新，改用主动失效/消息驱动 |

#### 推荐配置（搜索接口项目）

```java
// 详情页：核心路径，提前刷新 + 单飞（双保险）
@Cached(name = "product:detail:", expire = 600, cacheType = CacheType.BOTH, localExpire = 30)
@CacheRefresh(refresh = 480, stopRefreshAfterLastAccess = 1800)  // 8 分钟后台刷新，30 分钟没人访问就停
@CachePenetrationProtect                                          // 兜底：刷新失败/网络抖动时防击穿
public ProductDetail getDetail(Long id) { ... }

// 推荐料：非核心，只开单飞
@Cached(name = "recommend:", expire = 300, cacheType = CacheType.BOTH)
@CachePenetrationProtect
public List<Sku> recommend(Long userId) { ... }
```

**配置原则**：
- **单飞是底线**，所有缓存都该开（防 DB 雪崩，几乎零成本）；
- **提前刷新是奢侈品**，只给"对延迟敏感的核心接口"开（它会周期性多打 DB）；
- **两者叠加**才是生产级标准——提前刷新保证日常无毛刺，单飞兜住刷新失败的极端情况。

#### 提前刷新的两个关键参数

| 参数 | 含义 | 建议 |
|---|---|---|
| `refresh` | 刷新间隔 | 设为 `expire` 的 60%~80%（留出缓冲，避免刷新延迟导致过期） |
| `stopRefreshAfterLastAccess` | 多久没人访问就停止刷新 | **必设**！不设会一直刷新没人用的冷 key，浪费 DB |

> 多实例部署下，提前刷新**只有一个节点真正执行**（基于 Redis 分布式锁 + timestampKey 去重），不会放大 DB 压力。源码细节见 [JetCache 源码解析 §8](JetCache源码解析.md#8-自动刷新refreshcache-的后台线程模型)。

---

## 5. 生产踩坑清单（按血泪程度排序）

### 5.1 🔴 自愈型缓存雪崩（结构变更）

场景：上线改了 VO 字段类型，Redis 里躺着 10 万条老数据。

- 每条第一次被访问 → 反序列化失败 → 回源 DB → 覆盖；
- 10 万条 = 10 万次 DB 查询瞬间涌入。

**解法**：结构变更时**改 `name` 版本号**（`product:detail:` → `product:detail:v2:`），让老 key 自然过期，新 key 全新开始。或上线前用 SCAN + DEL 主动清。

### 5.2 🔴 多实例本地缓存不一致

场景：3 台机器，A 更新了 product:1，B、C 的 Caffeine 还在返回旧值。

**解法**：`syncLocal(true)`。JetCache 会在 A 更新时往 Redis 发一条 Pub/Sub 消息，B、C 收到后**清掉本地对应 key**（注意是清，不是同步新值——保持 L1 短 TTL + 广播失效是最终一致性的标准做法）。

> 只在 `cacheType=BOTH` 且多实例部署时才需要开。单机或纯 REMOTE 无所谓。

### 5.3 🟠 注解失效：内部调用

```java
public void outer() {
    this.inner();   // ❌ 不走 AOP 代理，@Cached 不生效
}

@Cached(name = "x:")
public String inner() { ... }
```

**解法**：把 `inner` 挪到另一个 Bean，或注入自身代理（`@Autowired private XxxService self;` 后调 `self.inner()`）。

### 5.4 🟠 null 值语义与缓存穿透

查一个不存在的 key，每次都打到 DB。**解法**：

```java
@Cached(name = "x:", cacheNullValue = true)   // 缓存 null
public X find(Long id) { ... }
```

注意 `cacheNullValue` 的 TTL 最好**短于正常 TTL**（JetCache 不直接支持，需自定义或在 value 里包一层带过期标记的对象）。

### 5.5 🟡 Key 设计：前缀 + 业务可读

JetCache 的最终 Redis key 形如：`${area}:${cacheName}${key}`，如 `default:product:detail:12345`。

约定：cacheName 用 `业务域:子域:` 格式（带末尾冒号），便于运维 SCAN 清理和告警分组。

### 5.6 🟡 裸用 `cache.get(key)` 的坑

```java
V v = cache.get(key);   // 反序列化失败时返回 null，但不会清脏数据、不会回源
if (v == null) { v = load(); cache.put(key, v); }   // 自己写的回源
```

这种用法下，反序列化失败 → get 返回 null → 你回源 → put 覆盖。看起来也对，**但如果回源失败，脏数据永不清理**。团队约定：**统一用 `computeIfAbsent` 或 `@Cached`**，禁止裸 `get` + 手写回源。

### 5.7 🟡 `areaInCacheName` 的版本差异

2.7 起默认 `false`（key 里不带 area 名）。从 2.6 升 2.7 时，**Redis 里已有数据的 key 会变**，等于一次性全量失效。升级前评估。

### 5.8 🟢 序列化大对象

搜索结果 List 可能很大（几十 KB）。Jackson/Kryo 都会比 JDK 序列化小 30%~60%，**网络和内存都省**。这也是选 Jackson 的理由之一。

---

## 6. 监控与运维

### 6.1 开统计

```yaml
jetcache:
  statIntervalMinutes: 15
```

每 15 分钟在日志里打印每个 cacheName 的：get 次数、命中率、load 次数、load 耗时。

### 6.2 接 Actuator / Prometheus

JetCache 的统计通过 `CacheMonitor` 暴露。可以自定义 Monitor 把指标送到 Micrometer：

```java
public class MicrometerMonitor implements CacheMonitor {
    private final MeterRegistry registry;
    // afterOperation 里把 CacheEvent 转成指标
}
```

关键指标：`cache.hit.rate`、`cache.load.count`、`cache.load.duration`、`cache.get.fail.count`（反序列化失败）。

### 6.3 运维接口：按 cacheName 清缓存

JetCache 不提供"按前缀批量删"。生产建议封一个 HTTP 接口（带鉴权），内部用 Redis SCAN + DEL，上线结构变更时调用：

```java
@PostMapping("/cache/evict")
public String evict(@RequestParam String cacheName) {
    // RedisTemplate.scan("default:" + cacheName + "*") → DEL
}
```

---

## 7. 什么时候 JetCache 不够，需要补位

| 场景 | JetCache 是否够 | 补位方案 |
|---|---|---|
| 标准读缓存（详情、推荐、替代料） | ✅ 够 | 直接用 |
| 多实例本地缓存一致性 | ✅ 够（syncLocal） | 直接用 |
| 反序列化失败自动回源 | ✅ 主流程够 | 加 Monitor 兜底回源也失败的情况 |
| 主动删脏数据 | ⚠️ 不主动删 | 自定义 `CacheMonitor` 里 REMOVE |
| 缓存预热 | ❌ 无 | 应用启动后主动 PUT 热点 key |
| 多级缓存 + 自定义降级策略 | ⚠️ 灵活度有限 | 自研补位或扩展 Monitor |

---

## 8. 速查清单（贴墙版）

```
✅ 序列化器用 jackson / fastjson2，别用 java
✅ cacheType=BOTH + syncLocal=true 解决多实例一致性
✅ 结构变更改 name 版本号，别依赖运行时自愈
✅ 防击穿开 penetrationProtect
✅ 统一用 @Cached / computeIfAbsent，禁裸 get
✅ cacheName 用 业务域:子域: 格式，带冒号
✅ 监控命中率 + 反序列化失败计数
✅ 上线前清缓存或换版本号
```

---

## 9. 相关笔记

- [JetCache 源码解析](JetCache源码解析.md) — 想知道上面这些行为底层怎么实现的，看这篇
