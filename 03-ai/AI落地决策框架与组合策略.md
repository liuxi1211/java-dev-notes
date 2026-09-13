---
title: AI 落地决策框架与组合策略（精炼版）
date: 2026-08-10
tags: [ai, architecture, 落地选型, 决策框架, agent, rag]
source: ai-conversation
related:
  - AI应用落地技术栈全景.md
  - TRAE与自研AI-Agent差异与选型.md
  - agent/Skill编排与多Agent协作机制.md
  - ../09-业务项目/stock-ai-system/股票AI问答系统设计.md
---

# AI 落地决策框架与组合策略（精炼版）

> 这是几轮"AI 落地选型"对话的**决策骨架提炼**，方便以后遇到任何 AI 落地场景时快速套用。详细论证见各关联笔记。

---

## 1. 一图看懂 AI 应用的本质

```
AI 应用 = 大模型（大脑）
        + 工具/MCP（手脚）
        + 知识库/RAG（记忆）
        + Agent 编排（管家）
```

四个组件，所有 AI 产品（客服、���索、资料查找、垂直 Agent）都是这四个的组合变化。

---

## 2. 语言选型：Python 还是 Java？

| 场景 | 选什么 |
|---|---|
| 自己训/微调模型 | **必须 Python**（算法工程师的活） |
| 部署开源大模型推理服务 | **必须 Python**（vLLM / TGI） |
| 用只有 Python 实现的算法库（rerank、切句、本地 embedding） | Python（独立服务） |
| **应用编排层**（对话管理、RAG 编排、权限、计费、业务对接） | **Java / Python / Go / Node 都行** |

**Java 团队的核心结论**：应用编排层是主战场，Spring AI Alibaba 完全能覆盖。Python 只在"模型层"必要，通过 HTTP/gRPC 调用即可。

---

## 3. 编排框架三流派

| 流派 | 代表 | 适用 |
|---|---|---|
| **低代码平台** | Dify / FastGPT / Coze | 快速 MVP、内部工具、非技术人员上手 |
| **Java 代码框架** | Spring AI Alibaba / LangChain4j | Java 团队主战场、深度定制、长期演进 |
| **Python 代码框架** | LangChain / LlamaIndex | 有 Python 能力、生态最全 |

---

## 4. TRAE vs 自研：核心决策树

```
这个 AI 系统，最终是谁在用？

├─ 是"我自己 / 几个同事"用
│   → 用 TRAE / Dify / Coze 足够
│   → 重点在调 Prompt、配 MCP、写 Skill
│
├─ 是"公司全员"用（内部平台）
│   → 自研为主（Spring AI Alibaba / Dify 自部署）
│   → TRAE 作为高级用户的增强工具
│   → 重点在权限、审计、多租户
│
└─ 是"对外客户"用（产品交付）
    → 必须自研
    → 重点在合规、商业化、SLA、用户体验
    → TRAE 不能作为运行时（只能作为研发工具）
```

详细差异矩阵见 [TRAE vs 自研 AI Agent 差异与选型](TRAE与自研AI-Agent差异与选型.md)。

---

## 5. 核心判断：TRAE 是工具，自研是产品

> **TRAE 和自研不是"谁做得更好"的对比，是"做出来给谁用"的对比。**

- **TRAE = 个人/团队的 AI 生产力工具**（"我"用得更爽）
- **自研 Agent 应用 = 对外交付的产品**（"我的客户"用得爽）

两者在 MCP 工具层、Agent 编排思想上同构可复用，但交付形态、用户对象、商业模式完全不同。

**关键事实**：TRAE 做出来的东西**只能在"装了 TRAE 并登录账号"的环境里运行**，不能像 App 一样打包交付给没有 TRAE 账号的最终用户。

---

## 6. 最优组合策略：���轨并行

```
[共享资产层]（两边都吃，不浪费）
 ├─ MCP Server（开放协议，TRAE 和自研都能调）
 ├─ Skill / SOP 知识库（Markdown，两边复用）
 └─ Prompt 库 / 评测集

       ┌──────────────────┐         ┌─────────────────────┐
       │  TRAE（自用）     │         │  自研 Agent 应用      │
       │  · 自己日常分析    │         │  · 对外产品           │
       │  · 快速原型验证    │         │  · 全员内部平台       │
       │  · Skill 试验田   │         │  · 商业化交付         │
       └──────────────────┘         └─────────────────────┘
```

**三个关键协同点**：
1. **MCP Server 是最大公约数**——TRAE 里验证好用的 MCP，直接搬进自研系统的工具层
2. **TRAE 作为"Skill 试验田"**——Prompt、SOP、Subagent 划分在 TRAE 里快速迭代（小时级反馈），成熟后翻译成自研代码
3. **不要把 TRAE 当后端运行时**——它是研发工具，不是交付产品

---

## 7. 落地三阶段（通用路线）

### 阶段 1：技术验证（1-2 周）

1. 用 **Dify 或 FastGPT** 搭 Demo，零代码，让老板看到效果
2. 同时用 **Spring AI Alibaba + 通义千问 + ES** 写最小 RAG，验证 Java 路线可行
3. 在 **TRAE** 里配好 MCP + 写 Skill，快速验证业务流程

### 阶段 2：选型决策

- 简单场景 / 快速上线 → Dify 自部署 + API 输出，Java 包一层业务
- 深度定制 / 长期演进 → Spring AI Alibaba 自研
- 个人/小团队内部用 → TRAE 直接搞定

### 阶段 3：团队能力

- 后端学 Spring AI（1-2 周上手）
- 不需要招算法工程师，**调好 API + 做好 RAG + 写好 Prompt** 覆盖 80% 场景
- 真要做模型微调，再单独引 Python 算法岗或外包

---

## 8. Java 团队推荐技术栈（开箱即用）

```
LLM：通义千问 / DeepSeek（性价比高，国内合规）
框架：Spring AI Alibaba 1.x
向量库：Elasticsearch 8.x（已会）或 Milvus
Embedding：bge-m3 或通义 text-embedding-v3
文档解析：Apache Tika + 自研切片 / MinerU
Rerank：bge-reranker-large
网关：Spring Cloud Gateway（已有）
监控：原有 Prometheus + 加 LLM token 计费统计
```

---

## 9. 一句话总结

> **AI 应用 80% 是应用工程，不是算法工程。**
> Java 团队用 Spring AI Alibaba + 通义千问 + ES 向量检索 + Dify 辅助，能做出市面上绝大多数 AI 产品。
> TRAE 是你自己的 AI 工具台（提效），自研是对外交付的产品（赚钱）。两者不冲突，组合使用最优。

---

## 关联笔记

- [AI 应用落地技术栈全景（总览）](AI应用落地技术栈全景.md) — 技术栈细节
- [TRAE vs 自研 AI Agent 差异与选型](TRAE与自研AI-Agent差异与选型.md) — 详细差异矩阵
- [Skill 编排本质与多 Agent 协作机制](agent/Skill编排与多Agent协作机制.md) — Agent 设计方法论
- [股票 AI 系统设计（实战案例）](../09-业务项目/stock-ai-system/股票AI问答系统设计.md) — 完整案例
- [Spring AI vs Spring AI Alibaba](SpringAI与SpringAIAlibaba选型.md) — 框架对比
