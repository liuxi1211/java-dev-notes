---
title: Chat API 多轮对话机制与上下文管理
date: 2026-08-14
tags: [llm, context-engineering, kv-cache, api]
source: ai-conversation
---

# Chat API 多轮对话机制与上下文管理

> 起因：读火山方舟 Chat API 文档时对 `messages` 参数产生的疑问，逐步延伸到多轮上下文管理、Responses API、KV Cache 前缀缓存的完整知识链。结论适用于所有 OpenAI 兼容风格 Chat API（方舟、OpenAI、DeepSeek、通义等）。

## 一、核心心智模型：LLM API 是无状态的

每次调用 `chat/completions` 都是一次完全独立的推理，模型不记得上一次调用了什么。多轮对话的上下文必须由**调用方维护**，每次请求把完整历史按时间顺序重放一遍：

```json
{
  "model": "doubao-seed-2-1-pro-260628",
  "messages": [
    { "role": "user",      "content": "我叫张三" },
    { "role": "assistant", "content": "你好，张三！" },
    { "role": "user",      "content": "我叫什么？" }
  ]
}
```

`messages` 数组 = 一份按时间排序的对话剧本，模型的任务只有一个：**给剧本接下一条 `assistant` 台词**。网页端"连续对话"的体验，底层就是每轮把历史全量拼进 `messages` 再发一次。

由此推出的工程结论：

- token 消耗随轮次**线性增长**，最终撞上下文窗口上限
- 超长后需要自己做历史裁剪或摘要
- 所谓"长期记忆"功能，本质是把外部存储的记忆注入 `messages`

## 二、role 的含义与底层原理

| role | 谁产生的 | 作用 |
|---|---|---|
| `system` | 开发者写死 | 人设、行为约束、输出格式等元指令，优先级最高 |
| `user` | 终端用户 | 用户提问/指令，触发模型响应 |
| `assistant` | 模型上一轮回复（由调用方回填） | 维持多轮上下文一致性 |
| `tool` | 程序 | 配合 `tools` 参数回传工具执行结果 |

**role 不只是 API 语法糖，它直接影响送入模型的 token**。底层会把每条消息序列化成训练时用的特殊标记（ChatML 格式，类似 `<|system|>...<|user|>...<|assistant|>...`），模型用这种对话格式训练，对不同角色的话有不同强度的响应：

- `system` 权重最高 → 安全约束、格式要求（"必须输出 JSON"）放这里最可靠，放 `user` 里效果打折
- `assistant` 历史回填影响后续风格，甚至可故意构造假 assistant 回复做 few-shot 示范
- 模型主要在回应最后一条 `user` 消息

响应里的 `choices[0].message` 就是"追加到数组末尾的下一条消息"，role 恒为 `assistant`。完整多轮循环：

```
messages = [system?, user]
→ 调 API → 拿到 assistant 回复 → append 回 messages
→ 用户再输入 user → 再调 API → 循环
```

## 三、上下文压缩：应用层的职责

Chat API **绝不会自动压缩**��史——只收数组、算 token、收钱。压缩由应用层做，三条路线：

| 策略 | 做法 | 成本 | 适用 |
|---|---|---|---|
| 滑窗截断 | 保留 system + 最近 N 轮，老的直接丢 | 零 | 大部分客服/助手场景 |
| LLM 摘要（rolling summary） | 超阈值时调模型把老对话压成要点 | 每次压缩一次调用 | 长对话且老信息有保留价值 |
| 检索式记忆（RAG） | 老对话 embedding 入库，每轮检索相关片段注入 | 检索成本 | 超长周期���话 |

摘要压缩的两种实现：

```java
// 全量重摘：简单，但每次压缩成本高
if (tokenOf(history) > THRESHOLD) {
    summary = llm.call("将以下对话压缩为要点，保留关键事实、决策和未完结话题：\n" + oldTurns);
    memory.replaceWith(conversationId, summary);
}

// 增量摘要：每次只消化一轮，成本摊薄（生产更常用���
// summary_new = llm.call("已有摘要：" + summary_old + "\n新对话：" + latestTurn + "\n合并更新摘要")
```

工程要点：

- **成本权衡**：压缩省的是后续每一轮都要重发的 N 轮 token，对话越长越划算；三五轮短对话不值得压
- **时机**：别在请求链路上同步压缩（加延迟），异步压缩、下轮生效
- **结构化抽取**（mem0 思路）：把对话抽取成结构化事实（`{姓名: 张三, 偏好: 简短回答}`）存库，每轮注入少量事实 token，信息密度最高

生产主流组合拳（三层各司其职）：

```
messages = [system]
         + [历史摘要]           ← 远期记忆，几百 token
         + [最近 5-8 轮原文]     ← 近期细节保真
         + [检索到的相关老片段]   ← 按需召回
         + [本轮 user 输入]
```

框架封装：Spring AI `ChatMemory`（`MessageWindowChatMemory` 即滑窗）、LangChain `ConversationSummaryMemory` / `ConversationBufferWindowMemory`。

## 四、Responses API：服务端保存对话状态

火山方舟已上线 Responses API（250615 及之后版本模型默认支持），OpenAI 侧 2025-03 推出。核心机制：

- 返回带 ID 的 Response 对象，**默认存储在服务端**（OpenAI 侧 `store=true` 默认保留 30 天）
- 下一轮只传**新输入 + `previous_response_id`**，服务端自动拼接链上历史：

```python
r1 = client.responses.create(model="doubao-seed-2-1-pro-260628", input="我叫张三")
r2 = client.responses.create(
    model="doubao-seed-2-1-pro-260628",
    previous_response_id=r1.id,   # 只传增量，不重发历史
    input=[{"role": "user", "content": "我叫什么？"}],
)
```

方舟还叠加显式上下文缓存（Session 缓存 `caching={"type": "enabled"}`），链式调用命中缓存的输入 token 有计费折扣。

**必须��除的三个误解**：

1. **不是免费记忆**：链上所有历史 token 仍按 input 计费（OpenAI 明确 "all previous input tokens are billed as input tokens"），只是命中部分打折。服务端保存 ≠ 省掉 token
2. **不是无限上下文**：模型窗口上限照旧，链太长照样超窗，裁剪/摘要还是得自己做——"存"外包了，"压缩"没外包
3. **不是你的数据库**：OpenAI 默认只留 30 天；方舟 Responses API 也不支持精调模型在线推理、TPM 保障包等场景。审计、合规、跨设备恢复仍需自存对话记录

**工程结论**：单会话省事用 Responses API；产品要长期演进，对话历史仍落自己的库（DB/ES），否则状态锁在厂商服务端，换模型迁移成本极高。

## 五、缓存命中的真正含义：Prefill 阶段的 KV Cache 前缀复用

`cached_tokens` 的命中与**网络传输无关**，是 Transformer 推理阶段（Prefill）的 KV Cache 复用。

自回归生成两阶段：

```
阶段一 Prefill：整个 prompt 所有 token 并行计算
              → 每层产出每个 token 的 K/V 矩阵（注意力键值）
              → 计算密集，是首 token 延迟（TTFT）的主要来源

阶段二 Decode：逐 token 生成
              → 新 token 的 Q 与所有历史 token 的 K/V 做注意力
              → 历史 K/V 直接从缓存读，不重算
```

关键观察：第 N 轮 prompt 和第 N+1 轮 prompt 的开头一大段完全相同（system + 前面轮次）。上次已算出这段的 K/V，服务端存下来下次复用：

- **命中** = 这段前缀的 K/V 张量直接复用，跳过重复 prefill 计算
- 收益 = TTFT 明显下降 + 省掉的算力折成计费折扣（所以命中部分打折计费而非免费）
- 观测 = `usage.prompt_tokens_details.cached_tokens`

两个重要推论：

1. **为什么必须前缀匹配**：K/V 和 token 绝对位置绑定（注意力按位置叠加），从第一个 token 逐块比对，一旦某处不同其后全部失效。这是官方最佳实践"静态内容放最前、动态内容放最后"的根本原因——system prompt 放最前才能稳定命中，每轮变动的用户输入放最后
2. **为什么存储计费/会淘汰**：KV Cache 按 token × 层数 × 隐藏维度存张量，开销远大于存文本（方舟显式缓存存储计费、隐式缓存不保证命中的原因——容量有限会被淘汰）

**概念区分**：跨请求的**前缀缓存**（多轮对话省算力的优化）≠ 单次请求内 decode 阶段的 **KV Cache**（自回归生成的基本机制，任何推理都有）。另外即使命中缓存，这些 token 依然占上下文窗口、依然计费——**缓存省的是算力，不是空间**。

## 参考

- [火山方舟 Chat API 文档](../火山引擎%20API%20文档.md)（本地副本）
- [迁移至 Responses API - 火山引擎](https://docs.volcengine.com/docs/82379/1585128)
- [上下文缓存原理及选型 - 火山引擎](https://www.volcengine.com/docs/82379/1398933)
- [Conversation state - OpenAI](https://developers.openai.com/api/docs/guides/conversation-state)
- [Data controls - OpenAI](https://developers.openai.com/api/docs/guides/your-data/)
