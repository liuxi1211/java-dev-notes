---
tags: [llm, agent]
date: 2026-08-25
source: ai-conversation
---

# Agent与Loop的本质

> Agent、Loop、SubAgent 三个核心概念的定义、关系与工程要点。相关笔记：[MCP-CLI-Skill三种接入模式对比](MCP-CLI-Skill三种接入模式对比.md)、[SubAgent工程实践](SubAgent上下文隔离与蒸馏实践.md)

## 1. Agent 是什么

**一句话定义：Agent = LLM + 工具调用 + 自主决策循环**

| 维度 | 普通 LLM（Chat） | Agent |
|---|---|---|
| 输出 | 只能生成文本 | 能执行动作（API、文件、代码） |
| 流程 | 一问一答 | 循环：思考→行动→观察→再思考 |
| 自主性 | 无 | 有目标，自主拆解任务 |
| 状态 | 无状态 | 有记忆/状态 |

**类比**：LLM 是被绑在椅子上的大脑；Agent 是解开绳子、给它工具，说"帮我订机票"，它自己完成查日期→搜航班→比价→下单。

### Agent 四块拼图

1. **大脑（LLM）**：推理决策
2. **工具（Tools）**：函数、API、MCP、代码执行器
3. **规划（Planning）**：任务拆解、反思、重试（ReAct、Plan-and-Execute）
4. **记忆（Memory）**：短期（上下文）+ 长期（向量库）

### Agent 爆发的原因（2023-2025）

- Function Calling 原生化：模型能可靠输出结构化工具调用 JSON
- 推理能力提升：撑住多步规划不跑偏
- 上下文窗口变长：装得下多轮工具返回
- MCP 等协议标准化：工具接入成本骤降

## 2. Loop（循环）是什么

Agent 圈说的 loops = **Agent Loop**，即 Agent 的发动机：

```
while (未完成) {
    LLM 思考 → 调用工具 → 观察结果 → 继续思考...
}
```

业界共识（源自 Anthropic《Building effective agents》）：**不需要复杂框架，一个足够强的模型 + 一个简单循环 = 强大的 Agent**，即 "It's just loops"。

### Loop vs Agent 的区别

- **Agent** 是"是什么"（系统/产品，名词）
- **Loop** 是"怎么跑"（驱动机制，如发动机工作循环）

关系：`Agent ≈ LLM + 工具 + Loop（+ 记忆/规划）`

趋势：从复杂多 Agent 编排回归**单循环 + 强模型**——每步可观测、可干预、好排查。

### 最小 Agent Loop 伪代码

```python
messages = [{"role": "user", "content": task}]

while True:
    resp = llm(messages, tools=tool_schemas)   # 思考
    if resp.tool_calls:                        # 行动
        for call in resp.tool_calls:
            result = execute(call)
            messages.append(observation(result))  # 观察
    else:
        return resp.text                       # 退出：最终答案
```
