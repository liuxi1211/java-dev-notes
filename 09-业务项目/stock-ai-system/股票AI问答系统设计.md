---
title: 股票 AI 问答系统设计：从需求到架构的完整实战
date: 2026-08-10
tags: [ai, agent, rag, mcp, spring-ai, 股票, 业务设计, architecture]
source: ai-conversation
project: stock-ai-system
related:
  - ../../03-ai/AI应用落地技术栈全景.md
  - ../../03-ai/agent/Skill编排与多Agent协作机制.md
  - ../../03-ai/TRAE与自研AI-Agent差异与选型.md
---

# 股票 AI 问答系统设计：从需求到架构的完整实战

> 这是一个**垂直领域 Agent 系统**的完整设计案例。背景：用户装了一堆金融数据 MCP 插件（iFinD、TDX、Wind、东方财富等），希望做一个完全基于 AI 的股票分析/场景问答系统，能力覆盖现有插件的全部共有能力。
>
> 本文回答两个核心问题：
> 1. 这算不算标准的 AI 企业落地？→ **算，且是教科书级的 Agent + RAG + MCP 混合架构**
> 2. 怎么实现？→ 四种实现路径 + 推荐架构 + 落地路线

---

## 1. 系统定性：这是什么类型的 AI 应用

装的这些插件（股票数据 MCP）本质是 **结构化数据查询能力**——查行情、查财务、查公告、查宏观。它们本身 **不是 AI**，只是数据接口。

要做的"股票 AI 问答系统"本质是：

> **大模型（大脑） + 一堆金融数据工具（手脚） + 行业知识库（记忆） + 一个会调度的 Agent（管家）**

AI 工程里标准名字：**Tool-augmented LLM / Agent 系统**，企业落地里最典型的形态之一。比纯 RAG 难，比通用 Agent 偏垂直。

| 维度 | 本系统 | 对应标准模式 |
|---|---|---|
| 能力来源 | 调插件拿实时数据 | **Function Calling / Tool Use** |
| 知识补充 | 行业研报、财报、规则 | **RAG（向量检索）** |
| 任务编排 | "分析这只股票"要拆成多步 | **Agent / 多步规划** |
| 输出形态 | 报告、图表、结论 | **结构化输出 + 引用溯源** |
| 数据时效 | 实时行情 + 历史数据 | **工具调用 + 缓存** |

---

## 2. 市面同类产品的真实架构（同花顺问财、东方财富AI、雪球AI）

都不是"一个 LLM 直接回答"，而是：

```
用户："宁德时代现在能买吗？"
        │
        ▼
┌──────────────────────────────────────────────┐
│  1. 意图理解 / Query 改写                      │
│     - 是查行情？查基本面？要买卖建议？          │
│     - 实体识别：宁德时代 → 300750              │
└────────────────┬─────────────────────────────┘
                 ▼
┌──────────────────────────────────────────────┐
│  2. Agent 规划：决定调哪些工具                  │
│     - get_stock_summary(300750)  基本面        │
│     - get_stock_performance(300750) 行情       │
│     - get_stock_financials(300750) 财报       │
│     - search_news("宁德时代")    资讯          │
│     - RAG: 检索"电池行业研报"                  │
└────────────────┬─────────────────────────────┘
                 ▼ (并行调用多个 MCP 工具)
┌──────────────────────────────────────────────┐
│  3. 工具执行（MCP 插件在这里发挥作用）          │
│     iFinD / TDX / Wind MCP 返回结构化数据     │
└────────────────┬─────────────────────────────┘
                 ▼
┌──────────────────────────────────────────────┐
│  4. 结果整合 + LLM 推理                        │
│     - 技术面：处于什么趋势位                   │
│     - 基本面：PE/PB/业绩增速                   │
│     - 资金面：北向、龙虎榜                     │
│     - 综合判断 + 风险提示                      │
└────────────────┬─────────────────────────────┘
                 ▼
┌──────────────────────────────────────────────┐
│  5. 输出：带图表 + 数据引用 + 免责声明的报告    │
└──────────────────────────────────────────────┘
```

**关键洞察**：装在 TRAE 里的这些插件 = MCP 工具，本系统 = 把这些工具交给一个 Agent 调度。TRAE 本身就是这么干的——TRAE 是 Agent 宿主，把 iFinD、TDX 等 MCP 服务器的工具暴露给大模型调用。

---

## 3. 四种实现路径（从简单到完整）

### 路径 1：基于现成 Agent 平台（最快，1 天出 Demo）

**用 Dify / Coze / FastGPT + MCP 集成**

- Dify 新版支持 MCP 工具接入
- 把 iFinD / TDX MCP 配置进去
- 写好 System Prompt（角色 = 股票分析师，调用规范，风控规则）
- 加一个知识库（研报、行业模板）

**优点**：快，能直接看到效果，非技术人员也能调
**缺点**：复杂分析、多步推理受限；不好做深度风控和合规审计

### 路径 2：Spring AI Alibaba 自研（企业级，Java 团队主路线）

**核心方案**。架构：

```java
// 核心代码骨架（Spring AI Alibaba）

@RestController
public class StockAgentController {

    private final ChatClient chatClient;

    public StockAgentController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("你是一位专业的A股分析师，必须基于工具返回的真实数据回答，"
                + "禁止编造数据，所有结论必须引用数据来源。涉及买卖建议必须附风险提示。")
            .defaultTools(
                new StockSummaryTool(),      // 包装 get_stock_summary
                new StockFinancialTool(),    // 包装 get_stock_financials
                new StockPerformanceTool(),  // 包装 get_stock_performance
                new StockNewsTool(),         // 包装 search_news
                new IndexTool(),             // 包装 index_data
                new MacroTool(),             // 包装 get_edb_data
                new RagSearchTool()          // 自研：检索研报/财报知识库
            )
            .build();
    }

    @PostMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chat(@RequestBody String userQuery) {
        return chatClient.prompt()
            .user(userQuery)
            .stream()
            .content();
    }
}

// 工具声明示例
@Component
public class StockFinancialTool {
    private final IfindStockClient ifind;  // 封装 iFinD MCP 的 HTTP 调用

    @Tool(description = "查询A股上市公司的财务报表，包括资产负债表、利润表、现金流量表")
    public String getFinancials(String stockCode) {
        return ifind.call("get_stock_financials", Map.of("stock_code", stockCode));
    }
}
```

**关键设计**：

1. **MCP 工具适配层**：iFinD / TDX MCP 都是 MCP 协议，要么用支持 MCP 的框架（Spring AI 1.x 支持 MCP Client），要么用 HTTP/Java 客户端封装成 `@Tool`
2. **RAG Tool 是自研的**：行业研报、历史财报、监管规则这类文本知识，走向量检索
3. **System Prompt 即风控**：金融场景 Prompt 必须硬约束（不准预测、必须加免责、必须引用数据）

### 路径 3：多 Agent 协作（**按需使用，不是默认选择**）

> **重要修正**：经过实际盘点股票插件（详见 [Skill vs Subagent 区分](../../03-ai/agent/Skill编排与多Agent协作机制.md)），市面上股票插件**主要用 Skill 而非 Subagent**。日常 90% 的股票问答，**单 Agent + 多 Skill 就够了**。多 Agent（独立上下文）只在以下场景才用：
> - 单次任务会塞爆上下文（如生成 50 页深度研报）
> - 需要不同模型/工具权限（如风控 Agent 用更保守的模型）
> - 多维度并行分析（Map-Reduce）

复杂任务（如"生成一份宁德时代深度研究报告"）拆成多个专业 Agent：

```
规划Agent → 分发任务
  ├─ 基本面Agent（专调财务工具，出基本面分析）
  ├─ 技术面Agent（专调行情工具，出技术分析）
  ├─ 资金面Agent（专调北向/龙虎榜，出资金分析）
  ├─ 行业Agent（RAG 检索研报，出行业对比）
  └─ 风控Agent（审核所有结论，加免责）
汇总Agent → 整合成最终报告
```

**关键认知**：
- "拆分任务"和"隔离上下文"是两个独立诉求，可以单独实现
- 独立上下文不一定要 Subagent，自研时用独立 ChatClient 即可（5 种实现方式详见 [Skill 编排笔记 §6](../../03-ai/agent/Skill编排与多Agent协作机制.md#6-独立上下文不用-subagent-也能实现的-5-种方式重点)）
- **编排策略**：90% 场景应该静态写死，只有入口层用动态意图分类（详见 [Skill 编排笔记 §7](../../03-ai/agent/Skill编排与多Agent协作机制.md#7-静态编排-vs-动态编排90-场景应该写死重点)）

Spring AI Alibaba 的 Graph 能力 / LangGraph4j 支持这种编排。详细拆分原则和实现见 [Skill 编排与多 Agent 协作机制](../../03-ai/agent/Skill编排与多Agent协作机制.md)。

### 路径 4：模型微调（垂直能力深化，可选）

当 Prompt + RAG + 工具用尽后，想让模型更像"真正的分析师"，可用研报/分析师点评数据做 SFT。**初期不要碰，后期再说。**

---

## 4. 与"标准企业 AI 落地"的对应关系

| 标准企业 AI 落地的要素 | 本系统是否覆盖 |
|---|---|
| Tool Calling / Function Calling | ✅ MCP 插件就是工具 |
| RAG 知识库 | ✅ 研报/财报/规则 |
| Agent 多步规划 | ✅ 分析一只股票要拆多步 |
| 流式输出 | ✅ SSE |
| 引用溯源 | ✅ 每个数据点要能回到源 |
| 风控/合规 | ✅ 金融场景的硬需求 |
| 权限/审计 | ✅ 企业落地标配 |
| 结构化输出 | ✅ 报告/图表/JSON |

**金融场景的特殊价值**：
- 数据高度结构化（行情、财报都是表）→ LLM 出错率低
- 用户需求清晰（查数、分析、对比）→ 容易定义工具集
- 失败可追溯（数据全靠工具，模型不编造）→ 合规好做

**唯一的特殊性**：金融合规。涉及买卖建议在国内有牌照要求，企业内部做"分析辅助系统"完全可以，对外做"荐股产品"要走牌照。**边界要提前明确**。

---

## 5. MCP 插件能力映射（针对实际安装的插件）

| MCP 来源 | 数量 | 在系统中的角色 |
|---|---|---|
| **iFinD 系列**（stock/fund/bond/index/news/edb/global） | 7 套 | **主力数据源**：A股、基金、债券、指数、宏观、新闻、港美股 |
| **TDX（通达信）** | 1 套 | **行情补充**：K线、分时、技术指标、板块、龙虎榜 |
| **Wind / mx-ds / full-link 系列** | 多套 | **能力叠加**：研报查询、宏观、AI盯盘等增值能力 |
| **MySQL** | 1 套 | **业务存储**：用户、自选股、问诊历史、计费 |
| **trends-hub / github-trending** | 2 套 | 舆情/热点补充（可选） |

**系统能力清单（覆盖插件的所有共有能力）**：

- 实时行情速览（quote 类工具）
- 个股深度分析（summary + financials + performance + shareholders）
- 技术面研判（K线、技术指标）
- 财报分析（financials + 业绩预告/快报）
- 行业对比（sector + 同业对比）
- 资金面（北向、龙虎榜）
- 资讯/公告/舆情（news + notice）
- 宏观环境（edb 宏观指标）
- 基金/债券分析
- 港美股分析
- 报告生成（多 Agent 产出深度报告）

---

## 6. 推荐实施路线

### 阶段 1：能力封装（1-2 周）

- 把装的 MCP 工具梳理成一份 **能力清单 + 参数 + 返回值** 的文档
- 用 Spring AI Alibaba 的 MCP Client 直接接入，或封装成 `@Tool`

### 阶段 2：单轮问答 MVP（1 周）

- ChatClient + 几个核心工具（行情、财务、资讯）
- 写好 System Prompt（分析师角色 + 风控约束）
- SSE 流式输出，跑通"问一只股票 → 拉数据 → 回答"

### 阶段 3：Agent 化（2-3 周，**渐进式**，不要一上来就多 Agent）

> **编排原则**：先单 Agent + 多 Skill（动态 Prompt 模板）覆盖 90% 场景，按需引入独立上下文（独立 ChatClient / Map-Reduce），最后才考虑完整多 Agent。详见 [Skill 编排笔记 §7-§8](../../03-ai/agent/Skill编排与多Agent协作机制.md)。

- **3.1 单 Agent + 多 Skill**：用 Advisor / 动态 Prompt 模板实现"按需加载分析方法"
- **3.2 入口层意图分类**：用小模型（qwen-turbo）判断用户意图，路由到对应的静态流程
- **3.3 按需引入独立上下文**：
  - 风控审核 → 必须独立 ChatClient（防污染）
  - 深度报告 → Map-Reduce（并行 + 隔离）
- **3.4 加 RAG**：研报知识库，加引用溯源

### 阶段 4：产品化（2-4 周）

- 前端（React + ECharts，把 LLM 生成的数据画成图）
- 自选股、问诊历史、用户体系
- 监控（token 成本、调用成功率、用户满意度）

---

## 7. 两个关键判断

1. **现在每天用 TRAE 问股票的体验，就是想做的产品的早期形态**。要做的是：把这种体验从"个人用 TRAE"变成"公司产品化部署给别人用"。技术内核一样，区别在工程化和产品化。详见 [TRAE vs 自研差异](../../03-ai/TRAE与自研AI-Agent差异与选型.md)。

2. **股票场景是从"AI 笔记/客服"跨到"AI 专业分析"的最佳跳板**。因为它强依赖结构化数据 + 工具调用，正好让 Java 后端的优势（系统集成、数据治理、工程化）能充分发挥，不会跟纯算法团队正面对抗。

---

## 关联笔记

- [AI 应用落地技术栈全景（总览）](../../03-ai/AI应用落地技术栈全景.md)
- [Skill 编排本质与多 Agent 协作机制](../../03-ai/agent/Skill编排与多Agent协作机制.md)
- [TRAE vs 自研 AI Agent 差异与选型](../../03-ai/TRAE与自研AI-Agent差异与选型.md)
- [AI 落地决策框架与组合策略](../../03-ai/AI落地决策框架与组合策略.md)
