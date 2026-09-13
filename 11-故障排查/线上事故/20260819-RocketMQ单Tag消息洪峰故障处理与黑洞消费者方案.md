---
tags: [rocketmq, troubleshooting, architecture, middleware]
date: 2026-08-19
source: ai-conversation
---

# RocketMQ 单 Tag 消息洪峰故障处理与黑洞消费者方案

## 背景

上游因 bug 向某 topic 的一个 tag 发送了几千万条消息，下游消费组处理不了，形成大规模堆积。且该消费组在同一 topic 上还订阅了其他 tag（必须继续正常消费），无法直接整组跳过堆积。

## 核心认知：RocketMQ 的隔离粒度

这是本次所有方案推导的基石：

- **offset 是 queue 级、按 consumerGroup 独立维护的**
- tag 不是隔离边界，queue 才是；同 queue 内不同 tag 的消息会互相阻塞
- 因此"只跳过某一个 tag"无法通过 resetOffset 实现（它跳的是整组所有 tag）
- 独立 consumerGroup 消费了消息，只推进自己的位点，原组的 lag 原地不动

## 紧急处理决策树

```
该 topic 该 group 是否只消费这一个 tag（或整 topic 都可丢）？
├─ 是 → resetOffset / Dashboard 跳过堆积（秒级，零代码）
├─ 否，但可以等 → 停消费等 commitLog 过期（默认72h，确认磁盘余量，防触发磁盘水位激进清理）
└─ 否，其他 tag 必须正常消费 → 黑洞消费者 / 消费端 tag 跳过（配置化）
```

关键命令：

```bash
# 查看堆积分布
sh mqadmin consumerProgress -n namesrv -g consumerGroup
# 按时间重置位点（高危，需审批，建议先 dump 再跳）
sh mqadmin resetOffsetByTimestamp -n namesrv -g consumerGroup -t topic -s "2026-08-19 10:00:00"
```

零代码备选：停掉消费者等 commitLog 物理过期（72h）。缺点是占磁盘，若触碰磁盘水位（85%）会触发提前激进清理，可能影响同 Broker 其他 topic。

## 黑洞消费者方案（独立 consumerGroup）

单独起一个 consumerGroup 只订阅问题 tag，策略化处理消息。

标准组合拳：

```
1. 修复上游 bug，停止脏消息继续产生
2. 黑洞组把问题 tag 消息全部消费掉（dump 到 MySQL 兜底，或直接 drop）
3. 对原消费组 resetOffset，把位点跳到 bug 修复之后的时间点
4. 原组恢复正常消费，业务无感
```

关键逻辑：第 3 步敢跳是因为第 2 步已把消息存到 MySQL——数据没丢只是绕过了 MQ，将来有价值可从库重放。故障处理中最贵的是不可逆决策，dump 给了后悔药。

代码骨架（Spring Boot + rocketmq-spring）：

```java
public interface BlackholeStrategy {
    String name();
    void handle(List<MessageExt> msgs);
}

@Component
public class DropStrategy implements BlackholeStrategy {
    public String name() { return "drop"; }
    public void handle(List<MessageExt> msgs) {
        log.info("dropped {} msgs", msgs.size());
    }
}

@Component
public class MysqlDumpStrategy implements BlackholeStrategy {
    public String name() { return "mysql"; }
    public void handle(List<MessageExt> msgs) {
        jdbcTemplate.batchUpdate(
            "INSERT INTO blackhole_dump (msg_id, tag, keys, body, born_time, dump_time) VALUES (?,?,?,?,?,NOW())",
            msgs, msgs.size(),
            (ps, m) -> {
                ps.setString(1, m.getMsgId());
                ps.setString(2, m.getTags());
                ps.setString(3, m.getKeys());
                ps.setBytes(4, m.getBody());
                ps.setLong(5, m.getBornTimestamp());
            });
    }
}
```

dump 表 DDL：

```sql
CREATE TABLE blackhole_dump_202608 (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    msg_id VARCHAR(64),
    tag VARCHAR(64),
    keys VARCHAR(128),
    body MEDIUMBLOB,
    born_time BIGINT,
    dump_time DATETIME,
    KEY idx_tag_time (tag, dump_time)
);
```

## 最终演进：消费端通用拦截器（长期方案）

不部署独立项目，直接在本项目所有消费者前加 AOP 拦截器 + Apollo 动态开关，故障时零发版零部署：

```java
@Aspect
@Component
public class BlackholeInterceptor {

    @Around("execution(* org.apache.rocketmq.spring.core.RocketMQListener+.onMessage(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        MessageExt msg = resolveMessageExt(pjp.getArgs()[0]); // 兼容 byte[]/MessageExt/POJO
        if (msg != null && blackhole.shouldBlock(msg.getTopic(), msg.getTags())) {
            blackhole.apply(msg);   // drop / mysql / log
            return null;            // 直接 ACK，不进业务
        }
        return pjp.proceed();
    }
}
```

Apollo 规则数组化，支持多 topic/tag 同时拦截：

```properties
blackhole.rules = [{"topic":"ORDER_NOTIFY","tag":"BAD_TAG","strategy":"mysql","enabled":true}]
```

四个工程要点（决定它是应急神器还是掉链子）：

1. **拦截点能拿到 tag**：POJO 模式下拿不到 tag，需统一用 MessageExt 或 header 透传，提前踩坑验证
2. **规则数组热更新**：Apollo ConfigChangeListener 生效，不重启
3. **dump 异步限速**：拦截器在消费线程池里，同步插库会拖慢正常 tag 消费；独立线程池攒批落库；几千万条要按天分表 + dump 速率上限（超限自动降级 drop 并告警）
4. **拦截计数监控**：按 topic/tag 暴露 metrics，否则开了拦截是盲飞

分工定位：拦截器是长期疫苗（拦未来流量）；独立黑洞组只剩一个场景有价值——业务服务整体被打挂时拦截器也随之失效，此时独立小工程可救场。

## 为什么优先独立组/拦截器而不是直接改原消费者

- 改业务服务代码要 CR/测试/审批/发版，期间堆积还在涨；拦截器/独立小工程与业务解耦，改配置即生效
- 切面在原组 ACK 消息是"过完即丢"没有后悔药；dump 是先存后丢，可纠错误判
- 注意：AOP 切面无法改变 consumerGroup，它只是代码插入，真正的隔离必须靠独立实例 + 独立 group

## 业界方案对比

| 方案 | 代表 | 特点 |
|---|---|---|
| 消费端拦截/熔断 | 本文方案 | 主流做法，配置中心动态开关 |
| 按 schema 版本拦截 | Confluent Schema Registry | 消息带版本，不认识的进 DLQ，比按 tag 更通用 |
| 服务端过滤 | Pulsar EntryFilter (4.6+) | Broker 端丢弃，不占推送带宽，终极形态 |
| DLQ + 一键重放 | AWS SQS redrive policy | 重放是控制台操作 |
| 消息治理平台 | 字节/美团/快手自建 | 发送配额准入 + 消费侧在线降级 + 全链路对账 |

升级方向（按投入产出排序）：

1. **发送侧配额**（治本）：发送封装层 per-tag 限流，超阈值熔断+告警，洪峰在源头被掐死
2. **拦截器 starter 化**：做成公司级 `mq-guard-starter`，规则统一配置中心管理
3. **重放工具化**：dump 只是半程，"按条件查询 + 限速重放回 MQ"形成拦→存→放闭环

## 预防清单

- 上游：所有发送加开关+限流；循环/重试类代码加发送量上限告警（洪峰 bug 几乎都来自无上限循环或重试风暴）
- 架构：不同业务拆 topic 或至少拆 consumerGroup，tag 不是隔离边界
- 监控：堆积量分级告警（>10万预警，>100万电话）；发送 QPS 突增告警；DLQ 消息数 >0 即告警
- 应急：订阅表达式外置配置中心；resetOffset 操作演练并记录注意事项（需审批）
- 消费端：限流+快速失败，重试超限直接落库人工处理，避免打挂存储

## Tags

- rocketmq
- troubleshooting
- architecture
- middleware
