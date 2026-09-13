---
title: JetCache 源码解析
date: 2026-08-14
tags: [source-reading, java, cache, jetcache, spring, aop, concurrency]
source: source-reading
project: alibaba/jetcache
repo: https://github.com/alibaba/jetcache
version: 2.7.x (master 2.8.0.RC)
---

# JetCache 源码解析

> 这篇不是 API 文档的翻译，而是**从「我要写一个生产级缓存框架」的视角**倒推 JetCache 做对了什么。每一段都尽量落到一个结论：**这个设计手法我以后能不能复用、怎么复用**。
>
> 配套阅读：[JetCache 使用教程](JetCache使用教程.md)
>
> 源码版本基线：`2.7.x` 稳定线 + `master` 分支 `2.8.0.RC`（2026-05）。下文涉及行号以 master 为准，可能随 commit 漂移。

---

## 0. 先看模块切分：依赖链暴露了架构意图

来自 [AGENTS.md](https://github.com/alibaba/jetcache/blob/master/AGENTS.md) 的官方模块说明：

```
jetcache-anno-api/   仅注解和常量，零依赖
jetcache-core/       核心 Cache API、CacheManager、本地/远程缓存实现
jetcache-anno/       基于 Spring AOP 的注解处理（依赖 core）
jetcache-support/    Redis 驱动适配：jedis / lettuce / springdata / redisson
jetcache-starter/    Spring Boot 自动装配 + starter
```

依赖链：`anno-api ← core ← anno ← autoconfigure`，`core ← support/* ← starter/*`。

**这个切分的可借鉴之处**：

1. **`anno-api` 独立成包、零依赖** —— 注解可以被业务方引用而不被迫拖入整个 core。Spring Data、MyBatis 都是这个套路。**自己写框架时，注解包一定要瘦**。
2. **`support` 按驱动拆模块** —— 加一个新 Redis 客户端不影响老代码，符合开闭原则。代价是抽象层要够稳。
3. **core 不依赖 Spring** —— `Cache`/`CacheManager` 是纯 Java 接口，`jetcache-anno` 才把 Spring AOP 引进来。这意味着 core 可以脱离 Spring 用（虽然实际很少这么干）。**框架核心逻辑与容器解耦，是成熟框架的标志**。

---

## 1. 核心抽象：为什么是 `Cache` + `CacheResult` 两套 API

源码位置：[Cache.java](https://github.com/alibaba/jetcache/blob/master/jetcache-core/src/main/java/com/alicp/jetcache/Cache.java)

JetCache 的 `Cache` 接口同时提供**小写方法**和**大写方法**两套：

```java
// 小写：像 Map，简单
V get(K key);
void put(K key, V value);

// 大写：返回 CacheResult，信息完整
CacheGetResult<V> GET(K key);
CacheResult PUT(K key, V value);
```

官方文档原话：*"the methods in `Cache` never throws exception."*

**为什么这么设计？关键在于「缓存的失败是常态，不是异常」**。

- 小写 API：把"未命中""反序列化失败""网络异常"**全部归约为 null**，调用方代码最简单，适合 90% 场景；
- 大写 API：返回 `CacheGetResult(SUCCESS/FAIL/NOT_EXISTS/EXISTS_WITH_NULL)`，让少数需要精细控制的调用方拿到完整信息。

**可借鉴结论**：

> 任何涉及外部资源（缓存、RPC、DB）的抽象层，都该提供**「归约失败」的简单 API + 「暴露失败」的精确 API** 两套。强行只给一套，要么简单场景写起来啰嗦，要么复杂场景信息不足。Java 的 `Future.get()`（抛异常）vs `CompletableFuture`（结果对象）也是同一思路。

---

## 2. 读取主流程：`AbstractCache` 是整个框架的骨架

源码位置：[AbstractCache.java](https://github.com/alibaba/jetcache/blob/master/jetcache-core/src/main/java/com/alicp/jetcache/AbstractCache.java)

这是最值得逐行读的一个类。它用**模板方法**把"统计、日志、事件通知"等横切逻辑固定下来，把"具体怎么读写"留给子类。

### 2.1 `GET` 方法：模板方法的教科书

```java
@Override
public final CacheGetResult<V> GET(K key) {
    long t = System.currentTimeMillis();
    CacheGetResult<V> result;
    if (key == null) {
        result = new CacheGetResult<>(CacheResultCode.FAIL, MSG_ILLEGAL_ARGUMENT, null);
    } else {
        result = do_GET(key);          // ⭐ 模板方法，子类实现
    }
    CacheGetEvent event = new CacheGetEvent(this, System.currentTimeMillis() - t, key, result);
    notify(result, event);             // ⭐ 通知所有 CacheMonitor
    return result;
}

protected abstract CacheGetResult<V> do_GET(K key);
```

注意三个细节，每个都值得抄：

1. **`GET` 是 `final`** —— 横切逻辑（计时、事件、校验）不允许子类覆盖，**保证所有缓存实现都有一致的可观测性**。自己写框架时，凡是有横切关注点的方法，都用 `final` + 模板方法锁死。
2. **计时放在 try 外面**（这里直接用 `System.currentTimeMillis()` 前后差）—— 简单粗暴但有效，避免引入 StopWatch 这种重对象。
3. **`notify` 不是直接同步调用，而是看 future 是否完成**：

```java
private void notify(CacheResult r, CacheEvent e) {
    CompletionStage<?> f = r.future();
    if (f.toCompletableFuture().isDone()) {
        notify0(e);                    // 已完成 → 同步通知
    } else {
        f.thenRunAsync(() -> notify0(e), JetCacheExecutor.defaultExecutor());  // 异步
    }
}
```

**这是个被很多人忽略的设计**：远程缓存的 PUT 可能是异步的（Lettuce 天然异步），事件通知要等真正写完再发，否则 Monitor 看到的"PUT 成功"可能是假的。**异步操作的事件通知，必须挂在 future 完成回调上，不能在发起处假装完成**。

### 2.2 `logError`：异常归约的统一出口

```java
protected void logError(String oper, Object key, Throwable e) {
    StringBuilder sb = new StringBuilder(256);
    sb.append("jetcache(").append(this.getClass().getSimpleName())
      .append(") ").append(oper).append(" error.");
    if (!(key instanceof byte[])) {
        try {
            sb.append(" key=[").append(config().getKeyConvertor().apply((K) key)).append(']');
        } catch (Exception ex) { /* ignore */ }
    }
    SquashedLogger.getLogger(logger).error(sb, e);
}
```

两个手法：

1. **`SquashedLogger`（压缩日志器）** —— 反序列化失败时同一个 key 可能一秒打几千条日志，把磁盘打爆。SquashedLogger 会把"相同异常+相同 key"的日志在时间窗口内合并。**生产框架必须有日志降级机制**，否则一次故障变两次。
2. **key 转换用 try-catch 兜底** —— 打日志时连 key 序列化都可能失败，所以这里 `catch (Exception ex) { /* ignore */ }`。**日志代码绝不能因为自身失败把主流程搞挂**，这是铁律。

**可借鉴结论**：

> 子类的 `do_GET` 里反序列化失败时，标准写法是：`catch → logError(oper, key, e) → return CacheGetResult(FAIL)`。**异常不外抛，归约成失败结果**。这就是为什么前面教程里说"反序列化失败不会让接口 500"——根因在这里。

> Changelog 里也有佐证：*"Fix RedisLettuceCache not catch and log decode exception during GET/GET_ALL. (#688)"* —— 早期版本漏 catch 过，后来补上。说明这个 catch 不是可有可无，是踩过坑才加的。

---

## 3. 回源与防击穿：`computeIfAbsentImpl` + `synchronizedLoad`

源码位置：`AbstractCache.java` 第 148 行附近。这是整个框架**最精妙、最难写对**的一段，建议反复读。

### 3.1 为什么要单飞（singleflight）

缓存击穿的经典场景：热 key 过期的瞬间，1000 个并发请求同时发现"未命中"，于是 1000 个请求同时回源 DB，DB 被打挂。

解法叫**单飞**：1000 个请求里，只有 1 个真正回源，其余 999 个等它返回后共享结果。Go 的 `singleflight`、Guava Cache 的 `LoadingCache` 都是这个思路。

### 3.2 JetCache 怎么做的

```java
static <K, V> V computeIfAbsentImpl(K key, Function<K, V> loader,
                                    boolean cacheNullWhenLoaderReturnNull,
                                    long expireAfterWrite, TimeUnit timeUnit,
                                    Cache<K, V> cache) {
    CacheGetResult<V> r = cache.GET(key);
    if (r.isSuccess()) {
        return r.getValue();           // 命中，直接返回
    } else {
        // 未命中（含反序列化失败）→ 回源
        Consumer<V> cacheUpdater = (loadedValue) -> {
            if (needUpdate(loadedValue, cacheNullWhenLoaderReturnNull, loader)) {
                cache.PUT(key, loadedValue, expireAfterWrite, timeUnit).waitForResult();
            }
        };
        V loadedValue;
        if (cache.config().isCachePenetrationProtect()) {
            loadedValue = synchronizedLoad(...);    // ⭐ 单飞
        } else {
            loadedValue = newLoader.apply(key);      // 不保护，直接并发回源
            cacheUpdater.accept(loadedValue);
        }
        return loadedValue;
    }
}
```

重点看 `synchronizedLoad`（同文件内）：

```java
static <K, V> V synchronizedLoad(CacheConfig config, AbstractCache<K, V> abstractCache,
                                 K key, Function<K, V> newLoader, Consumer<V> cacheUpdater) {
    ConcurrentHashMap<Object, LoaderLock> loaderMap = abstractCache.initOrGetLoaderMap();
    Object lockKey = buildLoaderLockKey(abstractCache, key);
    while (true) {
        boolean create[] = new boolean[1];
        LoaderLock ll = loaderMap.computeIfAbsent(lockKey, (unusedKey) -> {
            create[0] = true;
            LoaderLock loaderLock = new LoaderLock();
            loaderLock.signal = new CountDownLatch(1);
            loaderLock.loaderThread = Thread.currentThread();
            return loaderLock;
        });
        if (create[0] || ll.loaderThread == Thread.currentThread()) {
            // 我是第一个（或重入）→ 真正回源
            try {
                CacheGetResult<V> getResult = abstractCache.GET(key);   // ⭐ double check
                if (getResult.isSuccess()) {
                    return getResult.getValue();        // 等锁期间别人已写好
                }
                V loadedValue = newLoader.apply(key);
                cacheUpdater.accept(loadedValue);       // 写回缓存
                return loadedValue;
            } finally {
                if (create[0]) {
                    ll.signal.countDown();              // 唤醒等待者
                    loaderMap.remove(lockKey);
                }
            }
        } else {
            // 已有人在回源 → 我等
            try {
                Duration timeout = config.getPenetrationProtectTimeout();
                if (timeout == null) ll.signal.await();
                else {
                    boolean ok = ll.signal.await(timeout.toMillis(), TimeUnit.MILLISECONDS);
                    if (!ok) return newLoader.apply(key);   // 等超时 → 自己回源（降级）
                }
            } catch (InterruptedException e) {
                return newLoader.apply(key);                // 被中断 → 自己回源
            }
            if (ll.success) return (V) ll.value;            // 共享结果
            else continue;                                   // 回源者失败 → 重试抢锁
        }
    }
}
```

这段代码信息密度极高，拆解出 7 个值得抄的手法：

#### 手法 1：`ConcurrentHashMap.computeIfAbsent` 当分布式锁用

用 `loaderMap.computeIfAbsent(lockKey, ...)` 的原子性来"抢锁"——第一个线程进入 lambda 并 `create[0]=true`，后续线程拿到已存在的 `LoaderLock` 且 `create[0]=false`。

**比 `synchronized(key)` 强在哪**：`computeIfAbsent` 是非阻塞的 CAS 语义，抢不到的线程立刻走等待分支，不会把整个 map 锁住。**自己实现 per-key 锁时，这是首选方案，不要用 `synchronized` 或 `ReentrantLock` 数组**。

#### 手法 2：`CountDownLatch` 做一次性信号

```java
loaderLock.signal = new CountDownLatch(1);
// 回源者 finally: ll.signal.countDown();
// 等待者: ll.signal.await();
```

**为什么不用 `Object.wait/notify`**：wait/notify 要求持锁、要求 while 循环防虚假唤醒、容易漏 notify。`CountDownLatch` 是一次性、不可复用、但语义清晰——正好匹配"回源只发生一次"的场景。**一次性事件用 CountDownLatch，周期性事件用 Condition 或 Phaser**。

#### 手法 3：Double Check（拿到锁后再查一次）

```java
if (create[0] ...) {
    CacheGetResult<V> getResult = abstractCache.GET(key);   // 再查一次！
    if (getResult.isSuccess()) return getResult.getValue();
    ...
}
```

抢到锁的线程在真正回源前**再 GET 一次**。为什么？因为在它"从 computeIfAbsent 出来"到"进入 try"之间，可能有别处的写入（比如别的节点通过广播失效后又重新填充）。

**这是双重检查锁的标准范式**：拿到锁后必须重新校验前置条件，否则会做重复工作。单例模式的 DCL 也是同理。

#### 手法 4：超时降级，不无限等

```java
boolean ok = ll.signal.await(timeout.toMillis(), TimeUnit.MILLISECONDS);
if (!ok) return newLoader.apply(key);   // 超时 → 自己回源
```

**不让等待者无限阻塞**。如果回源者卡死（DB 慢查询），等待者超时后自己回源，保证可用性。**任何"等待共享结果"的代码都必须有超时，否则一个慢请求能拖垮整个线程池**。

#### 手法 5：中断也降级

```java
catch (InterruptedException e) {
    return newLoader.apply(key);
}
```

被中断时不清空中断状态、���向上抛，而是自己回源。**缓存层的语义是"尽量帮忙，帮不上就自己干"，绝不能让单飞机制成为故障传播路径**。

#### 手法 6：回源失败 → 重试抢锁

```java
if (ll.success) return (V) ll.value;
else continue;   // 回源者失败，我重新进 while 抢锁
```

如果第一个回源者失败了（`ll.success` 仍为 false），等待者不会直接失败，而是 `continue` 重新抢锁、自己回源。**这是"尽力成功"的韧性设计**。

#### 手法 7：重入保护

```java
if (create[0] || ll.loaderThread == Thread.currentThread())
```

如果回源方法内部又调用了同一 key 的缓存（罕见但可能，比如递归），用 `loaderThread == currentThread` 识别重入，避免自己等自己死锁。**任何用锁的代码都要考虑重入**，否则就是个埋着的雷。

### 3.3 `buildLoaderLockKey`：本地锁 key 要独立于缓存 key

```java
private static Object buildLoaderLockKey(Cache c, Object key) {
    if (c instanceof AbstractEmbeddedCache) {
        return ((AbstractEmbeddedCache) c).buildKey(key);
    } else if (c instanceof AbstractExternalCache) {
        byte bytes[] = ((AbstractExternalCache) c).buildKey(key);
        return ByteBuffer.wrap(bytes);     // ⭐ 用 ByteBuffer 而非 String
    } else if (c instanceof MultiLevelCache) {
        c = ((MultiLevelCache) c).caches()[0];
        return buildLoaderLockKey(c, key);
    }
    ...
}
```

两个细节：

1. **多级缓存的锁 key 取第一级（本地）的 key** —— 因为单飞是 JVM 内的行为，用本地 key 即可，不需要把远程 key 的字节也纳入锁。
2. **`byte[]` 包成 `ByteBuffer`** —— 因为 `byte[]` 自身的 `hashCode/equals` 是引用语义，不能当 Map key。`ByteBuffer` 包装后按内容比较。**这是个 Java 语义陷阱，自己写代码踩过的人都懂**。

> ⚠️ **常见误解**：单飞（防击穿）和提前刷新（`@CacheRefresh`，见 §8）都"减少回源"，但**不能互相替代**——单飞只能降低回源次数，无法消除回源延迟。完整对比见 [§8.5 提前刷新 vs 单飞](#85-提前刷新-vs-单飞源码视角的对比必读)。

---

## 4. 多级缓存编排：`MultiLevelCache` 的极简哲学

源码位置：[MultiLevelCache.java](https://github.com/alibaba/jetcache/blob/master/jetcache-core/src/main/java/com/alicp/jetcache/MultiLevelCache.java)

多级缓存的读流程，核心就几行（伪代码还原）：

```java
// MultiLevelCache.do_GET
protected CacheGetResult<V> do_GET(K key) {
    for (Cache c : caches) {              // caches[0]=local, caches[1]=remote
        CacheGetResult<V> r = c.GET(key);
        if (r.isSuccess()) {
            // 命中高层（remote）时，回种低层（local）—— 但有条件
            return r;
        }
    }
    return CacheGetResult.NOT_EXISTS;
}
```

**值得抄的设计**：

1. **`caches` 是数组而非两层字段** —— `Cache[]` 而不是 `Cache local, Cache remote`。这样**理论上支持任意层**（虽然实际只用两层）。用数据结构表达"层级可扩展"，比用字段表达灵活得多。
2. **回种用 PUT，不走 loader** —— 高层命中后，把值 PUT 到低层。注意 JetCache 在某些版本里**有条件地回种**（比如 localExpire 已过半才回种），避免每次读都触发本地写。**回种不是无脑写，要考虑写放大**。
3. **锁用最后一层** —— `tryLock` 时 `MultiLevelCache use the last level to acquire the lock`（见 CacheAPI 文档）。分布式锁放 Redis 上，本地锁放 JVM 里，这是多级缓存的必然选择。

---

## 5. 一致性广播：`syncLocal` 怎么做的

这是 JetCache 区别于"手撸 Caffeine+Redis"最大的卖点。

配置开关：`broadcastChannel: projectA`（远程区配置）+ `QuickConfig.syncLocal(true)`。

### 5.1 机制

1. 每个 JVM 实例启动时，订阅 Redis 的一个 Pub/Sub channel（`broadcastChannel` 配置）；
2. 实例 A 执行 `PUT/REMOVE` 时，除了写自己的本地缓存和 Redis，**额外往 channel 发一条"我改了 key X"的消息**；
3. 实例 B、C 收到消息后，**清掉自己本地缓存的 key X**（注意是清，不是同步新值）。

### 5.2 为什么是"失效广播"而不是"更新广播"

这是最终一致性的标准做法，三个理由：

1. **减少消息体积** —— 失效消息只需带 key，更新消息要带 value（可能很大）；
2. **避免并发更新的乱序** —— 两个节点同时 PUT 同一 key，更新广播会让 key 在两份旧值间反复横跳；失效广播则无所谓顺序，最终都是清掉重新加载；
3. **本地缓存短 TTL 兜底** —— 即使广播丢失（Pub/Sub 不保证可靠），本地缓存也会在几十秒内自然过期，不会长期脏。

**可借鉴结论**：

> 多副本本地缓存的一致性，**永远用"失效广播 + 短 TTL 兜底"，不要用"更新广播"**。这是 Redis Cluster、Hazelcast、Ignite 都遵循的范式。Pub/Sub 不可靠没关系，只要 TTL 够短，最坏情况是"短暂不一致"而非"永久脏"。

### 5.3 源码位置

- `CacheNotifyMonitor`（jetcache-anno）—— 负责在写操作后发消息；
- 各 Redis 实现里的订阅逻辑 —— `RedisLettuceCache` / `RedissonCache` 等，监听 channel 并调用本地 `remove`。

2.7.7 Changelog: *"allow customize CacheNotifyMonitor"* —— 说明这块是可扩展的，你可以换成 Kafka/RocketMQ 广播。

---

## 6. 注解 AOP：`jetcache-anno` 怎么把注解变成缓存行为

源码位置：`jetcache-anno/` 模块，核心类 `CacheInterceptor`、`CacheInvokeConfig`、`ConfigProvider`。

### 6.1 切面挂载

`@EnableMethodCache(basePackages = "...")` 触发注册一个 `CacheInterceptor`（Spring AOP 的 `MethodInterceptor`），pointcut 匹配带 `@Cached`/`@CacheUpdate`/`@CacheInvalidate` 的方法。

**注意：JetCache 用的是 Spring AOP 而非 AspectJ**。这意味着：

- ✅ 享受 Spring 容器集成，配置简单；
- ❌ **内部调用不走代理**（`this.foo()` 不触发注解）—— 这是 Spring AOP 的固有限制，JetCache 文档 FAQ 里专门列了这条。

**可借鉴结论**：

> 选 Spring AOP 还是 AspectJ，是个明确的取舍。**99% 的业务框架选 Spring AOP 就够，但要主动在文档里写明"内部调用不生效"**，否则用户必踩坑。JetCache 的 FAQ 第一条就是这个。

### 6.2 SpEL key 解析

`@Cached(key = "SpEL.userId")` 需要编译时加 `-parameters`，否则只能用 `args[0]`。AGENTS.md 原话：*"The `-parameters` javac flag is required for SpEL annotation key expressions to work (also must be set in IDE settings, not just pom)."*

**这是个隐蔽的坑**：Maven 配了 `-parameters` 但 IDEA 没配，本地跑正常、CI 跑挂。**自己写注解框架用到方法参数名时，一定要在 README 里加粗强调 IDE 设置**。

### 6.3 注解元信息的缓存：`CachedAnnoConfig`

JetCache 不会每次调用都重新解析注解。启动时扫描所有带注解的方法，解析成 `CacheInvokeConfig`（包含 key 的 SpEL 表达式、TTL、cacheType 等），缓存起来。

**手法**：注解解析是反射 + SpEL，开销不小，**必须在启动期一次性解析并缓存**。运行期只查 Map。Spring MVC 的 `RequestMappingHandlerMapping` 也是这个套路。

---

## 7. 序列化：`ValueEncoder/ValueDecoder` 的身份号机制

源码位置：`jetcache-core/src/main/java/com/alicp/jetcache/support/`。

JetCache 的 value 在 Redis 里存的是 `[1字节类型标记][序列化字节]`。读取时先看类型标记，决定用哪个 Decoder。

```java
// 伪代码
public interface ValueDecoder extends Function<byte[], Object> {
    int identity();
    // ...
}
```

每种 Decoder 有个 `identity number`（如 Java=0、Kryo5=负数）。Encoder 写入时把 identity 写进字节流头部，Decoder 读取时按 identity 找对应实现。

**这个设计解决的问题**：同一个 Redis 实例里可能混存了 Java 序列化和 Kryo 序列化的数据（比如升级序列化器后老数据还在），Decoder 必须能识别"这条数据是哪种格式"。

**Issue #852** 就是个反面教材：早期 `DecoderMap` 的初始化有并发问题，导致启动瞬间读到未注册的 identity 而报 `no decoder for identity number:-153049664`。后来加了锁修复。**这说明身份号机制是对的，但初始化的线程安全要格外小心**。

**可借鉴结论**：

> 任何支持多种序列化的系统，都该在字节流里**写入格式标识**，而不是假设"全场都用同一种格式"。Protobuf 的 tag、Hessian2 的 type code、JSON 的 schema 版本都是同思路。**升级序列化器时，老数据能被识别、能被平滑迁移，靠的就是这个标识**。

### 7.1 2.8 的反序列化安全过滤器

2.8.0 新增 `decodeFilter`，默认开启。`java`/`kryo` 这类带类型信息的反序列化有 RCE 风险（经典 Jackson/FASTJSON 漏洞），过滤器用白名单 `decodeFilterAllowPatterns` 限制能反序列化的类。

**Changelog 原话**：*"Add decode filter for security (incompatible)"*。

**可借鉴结论**：

> 任何带类型信息的反序列化（默认 typing），**生产必须配白名单**。这是 2021 年 Log4Shell 之后的基本功。自己写框架如果默认开启 typing，等于给用户埋雷。

---

## 8. 自动刷新：`RefreshCache` 的后台线程模型

`@CacheRefresh(refresh = 240)` 让热 key 在过期前由后台线程主动刷新，避免"过期瞬间回源"。

源码位置：[RefreshCache.java](https://github.com/alibaba/jetcache/blob/master/jetcache-core/src/main/java/com/alicp/jetcache/RefreshCache.java)（271 行，继承 `LoadingCache`）。

### 8.1 任务的生命周期：访问驱动 + 续期

```java
// RefreshCache.java — 每次 GET 都会调用
@Override
public V get(K key) {
    if (config.getRefreshPolicy() != null && hasLoader()) {
        addOrUpdateRefreshTask(key, null);   // ⭐ 访问才注册任务
    }
    return super.get(key);
}

protected void addOrUpdateRefreshTask(K key, CacheLoader<K,V> loader) {
    RefreshPolicy refreshPolicy = config.getRefreshPolicy();
    long refreshMillis = refreshPolicy.getRefreshMillis();
    Object taskId = getTaskId(key);
    RefreshTask refreshTask = taskMap.computeIfAbsent(taskId, tid -> {
        RefreshTask task = new RefreshTask(taskId, key, loader);
        task.lastAccessTime = System.currentTimeMillis();
        ScheduledFuture<?> future = JetCacheExecutor.heavyIOExecutor()
                .scheduleWithFixedDelay(task, refreshMillis, refreshMillis, TimeUnit.MILLISECONDS);
        task.future = future;
        return task;
    });
    refreshTask.lastAccessTime = System.currentTimeMillis();   // ⭐ 续期 lastAccessTime
}
```

三个关键设计：

1. **访问驱动注册** —— 任务不是启动时全量创建，而是**第一次 GET 某 key 时才创建**。冷 key 永远不会被刷新，省 DB。
2. **`heavyIOExecutor` 独立线程池** —— 刷新任务跑在专用线程池，不污染业务线程池（默认 `JetCacheExecutor.defaultExecutor()` 是给事件通知用的）。**自己写后台任务时，重 IO 任务一定要和轻量任务分池，否则重任务会把轻任务饿死**。
3. **`lastAccessTime` 续期** —— 每次访问都更新这个字段，配合 `stopRefreshAfterLastAccess` 判断"这个 key 还有没有人在用"。

### 8.2 `RefreshTask.run`：刷新执行体

```java
@Override
public void run() {
    try {
        if (config.getRefreshPolicy() == null || (loader == null && !hasLoader())) {
            cancel();   // 配置变了或没 loader → 自我注销
            return;
        }
        long now = System.currentTimeMillis();
        long stopMillis = config.getRefreshPolicy().getStopRefreshAfterLastAccessMillis();
        if (stopMillis > 0 && lastAccessTime + stopMillis < now) {
            cancel();   // ⭐ 超过 stopRefreshAfterLastAccess 没人访问 → 注销任务
            return;
        }
        Cache concreteCache = concreteCache();
        if (concreteCache instanceof AbstractExternalCache) {
            externalLoad(concreteCache, now);   // 远程缓存：走分布式锁
        } else {
            load();                              // 本地缓存：直接刷新
        }
    } catch (Throwable e) {
        logger.error("refresh error: key=" + key, e);   // ⭐ 失败只记日志，老值保留
    }
}
```

**最值得抄的一行是 catch**：刷新失败时**不清缓存、不抛异常**，老值继续服务。**后台任务的失败绝对不能影响在线服务**，这是铁律。

### 8.3 `externalLoad`：分布式环境去重刷新（最精妙的部分）

多实例部署时，3 台机器都会为同一个热 key 创建 RefreshTask，如果不加控制，同一 key 会被刷新 3 次，DB 压力翻 3 倍。JetCache 的解法：

```java
private void externalLoad(final Cache concreteCache, final long currentTime) {
    byte[] newKey = ((AbstractExternalCache) concreteCache).buildKey(key);
    byte[] lockKey = combine(newKey, LOCK_KEY_SUFFIX);          // "_#RL#"
    byte[] timestampKey = combine(newKey, TIMESTAMP_KEY_SUFFIX); // "_#TS#"

    // 1. 查上次刷新时间戳
    CacheGetResult refreshTimeResult = concreteCache.GET(timestampKey);
    boolean shouldLoad = false;
    if (refreshTimeResult.isSuccess()) {
        shouldLoad = currentTime >= Long.parseLong(refreshTimeResult.getValue().toString()) + refreshMillis;
    } else if (refreshTimeResult.getResultCode() == CacheResultCode.NOT_EXISTS) {
        shouldLoad = true;
    }
    if (!shouldLoad) {
        if (multiLevelCache) refreshUpperCaches(key);   // 别人刷过了 → 只回种本地
        return;
    }

    // 2. 抢分布式锁，只让一个节点真正 load
    Runnable r = () -> {
        try {
            load();
            concreteCache.put(timestampKey, String.valueOf(System.currentTimeMillis()));
        } catch (Throwable e) { throw new CacheException("refresh error", e); }
    };
    boolean lockSuccess = concreteCache.tryLockAndRun(lockKey, loadTimeOut, TimeUnit.MILLISECONDS, r);

    // 3. 没抢到锁 + 多级缓存 → 延迟回种本地（让本地数据尽快更新）
    if (!lockSuccess && multiLevelCache) {
        JetCacheExecutor.heavyIOExecutor().schedule(
                () -> refreshUpperCaches(key), (long)(0.2 * refreshMillis), TimeUnit.MILLISECONDS);
    }
}
```

拆出 4 个手法：

#### 手法 1：双 key 辅助（`lockKey` + `timestampKey`）

JetCache 在 Redis 里为每个 key 额外建两个辅助 key：

- `{key}_#RL#` —— Refresh Lock，分布式锁标识；
- `{key}_#TS#` —— Timestamp，上次刷新时间。

**为什么需要 timestampKey**：`scheduleWithFixedDelay` 在每台机器上独立运行，3 台机器可能几乎同时触发刷新。光靠分布式锁会让 2 台机器空跑（抢锁失败白等一次 refreshMillis）。timestampKey 让**没抢到锁的节点先判断"别人是不是刚刷过"**，刷过就直接跳过 load。

**这是分布式定时任务的标准模式**：锁解决"同时只一个执行"，时间戳解决"不必每个周期都执行"。K8s CronJob、XXL-JOB 的调度去重都是同思路。

#### 手法 2：`tryLockAndRun` 而非 `lock + unlock`

```java
boolean lockSuccess = concreteCache.tryLockAndRun(lockKey, loadTimeOut, TimeUnit.MILLISECONDS, r);
```

用"尝试加锁→执行→自动释放"的封装，而不是手动 `tryLock`/`unlock`。**好处**：不会漏 unlock，不会因异常导致锁泄漏。**自己封装分布式锁时，优先提供 `tryLockAndRun` 这种模板方法 API，而不是裸 tryLock**。

#### 手法 3：抢锁失败也做有用

## 9. 监控：`CacheMonitor` 的观察者模式

源码位置：`CacheMonitor.java` + `DefaultCacheMonitor`。

```java
public interface CacheMonitor {
    void afterOperation(CacheEvent event);
}
```

每次 GET/PUT/REMOVE 后，`AbstractCache.notify` 把 `CacheEvent` 广播给所有 Monitor。`DefaultCacheMonitor` 累积统计（命中数、load 数、load 耗时），每 `statIntervalMinutes` 打印一次。

**手法**：

1. **Monitor 是 list，支持多个叠加** —— 一个缓存实例可以同时挂"统计 Monitor + 自定义告警 Monitor + 脏数据清理 Monitor"；
2. **事件携带耗时** —— `CacheGetEvent` 里有 `durationMillis`，统计不用自己再计时；
3. **异步通知**（见 §2.1）—— Monitor 慢不会拖累主流程。

**可借鉴结论**：

> 任何框架都要留**可插拔的观察点**。JetCache 的 `CacheMonitor` 是个范本：接口极简（一个方法）、事件自描述、支持多实例、异步通知。**自己写框架时，把"统计、日志、埋点"全做成 Monitor 插件，比硬编码在主流程里强 10 倍**。

这也是为什么前面教程建议"自定义 Monitor 主动清理反序列化失败的脏 key"——扩展点已经留好了，加个 Monitor 类就行。

---

## 10. 总结：JetCache 的设计哲学，以及我能偷师什么

把全文的设计手法浓缩成 11 条可复用的原则：

| # | 原则 | JetCache 的体现 |
|---|---|---|
| 1 | 注解包零依赖，核心与容器解耦 | `jetcache-anno-api` 独立 |
| 2 | 简单 API + 精确 API 双套 | 小写 `get` vs 大写 `GET` |
| 3 | 横切逻辑用 `final` + 模板方法锁死 | `AbstractCache.GET` |
| 4 | 失败归约，不抛异常 | `CacheResult(FAIL)` |
| 5 | 日志必须有降级（防刷屏） | `SquashedLogger` |
| 6 | 单飞防击穿：`ConcurrentHashMap` 当锁 + `CountDownLatch` 信号 + 超时降级 + double check + 重入保护 | `synchronizedLoad` |
| 7 | 多副本一致性：失效广播 + 短 TTL 兜底，别用更新广播 | `syncLocal` + broadcastChannel |
| 8 | 序列化字节流带格式标识，支持混存与平滑迁移 | `ValueDecoder.identity()` |
| 9 | 带类型反序列化必须配白名单 | 2.8 `decodeFilter` |
| 10 | 可插拔监控点，比硬编码统计强 | `CacheMonitor` 观察者 |
| 11 | 主动刷新要和单飞叠加：刷新保体验，单飞保命 | `RefreshCache` + `synchronizedLoad` 正交 |

**最后一句话**：

> JetCache 的代码量不算大（core 几千行），但每一处并发、异常、一致性、可观测性的处理都"刚刚好"。读它的源码，最大的收获不是学会用 JetCache，而是**学会怎么写一个"生产级"的基础设施库**——区别于 demo 的地方，全在那些 try-catch、超时、降级、日志降级、double check 这些"边角"里。

---

## 11. 延伸阅读与源码索引

- 模块结构与构建：[AGENTS.md](https://github.com/alibaba/jetcache/blob/master/AGENTS.md)
- 官方中文入门：[GettingStarted.md](https://github.com/alibaba/jetcache/blob/master/docs/CN/GettingStarted.md)
- 版本变更：[Changelog](https://github.com/alibaba/jetcache/wiki/Changelog)
- 关键源码文件（master 分支）：
  - [AbstractCache.java](https://github.com/alibaba/jetcache/blob/master/jetcache-core/src/main/java/com/alicp/jetcache/AbstractCache.java) — 模板方法骨架、`computeIfAbsentImpl`、`synchronizedLoad`
  - [Cache.java](https://github.com/alibaba/jetcache/blob/master/jetcache-core/src/main/java/com/alicp/jetcache/Cache.java) — 双套 API 定义
  - [MultiLevelCache.java](https://github.com/alibaba/jetcache/blob/master/jetcache-core/src/main/java/com/alicp/jetcache/MultiLevelCache.java) — 多级编排
  - [RefreshCache.java](https://github.com/alibaba/jetcache/blob/master/jetcache-core/src/main/java/com/alicp/jetcache/RefreshCache.java) — 自动刷新
  - `support/` 目录 — 各种 Encoder/Decoder、KeyConvertor、SquashedLogger

## 12. 相关笔记

- [JetCache 使用教程](JetCache使用教程.md) — 使用者视角的接入与踩坑
