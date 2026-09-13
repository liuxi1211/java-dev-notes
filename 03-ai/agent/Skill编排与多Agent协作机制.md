---
title: Skill 编排本质、多 Agent 协作与上下文隔离机制（含自研对照）
date: 2026-08-10
tags: [ai, agent, skill, subagent, mcp, spring-ai, prompt-engineering, 多agent, 上下文隔离]
source: ai-conversation
related:
  - ../AI应用落地技术栈全景.md
  - ../TRAE与自研AI-Agent差异与选型.md
  - ../AI落地决策框架与组合策略.md
  - Wind金融Skill能力清单.md
  - ../../09-业务项目/stock-ai-system/股票AI问答系统设计.md
---

# Skill 编排本质、多 Agent 协作与上下文隔离机制（含自研对照）

> 这篇笔记是 Agent 设计方法论的完整版，回答以下几个层层递进的核心问题：
> 1. TRAE 里 Skill 编排到底在做什么？"不深入理解就出不来好效果"怎么解决？
> 2. 多 Agent 协作到底怎么实现？怎么知道拆分哪些？
> 3. **Skill 和 Subagent 有什么区别？什么时候用哪个？**
> 4. **独立上下文是不是不用 Subagent 也能实现？**
> 5. **自己实现时，编排该写死还是动态？**

核心论点：
- **Skill 和 Subagent 是两个独立的机制，不要混淆**
- **"拆分任务"和"隔离上下文"是两个独立诉求，可以单独实现**
- **90% 的企业场景应该用静态编排（写死），只有开放式场景才动态**

---

## 1. Skill 编排在做什么：是 Prompt，不是代码

### 1.1 关键澄清

一个常见的认知误区：

> ❌ "Skill 是决定调用哪个 MCP 的代码"

正确理解：

> ✅ "Skill 是写给 LLM 看的 SOP 文档。真正决定调什么的，是 LLM 自己。"

TRAE 里的 Skill（比如"全链路个股分析"）本质上是一个 **Markdown 文档**：

```markdown
---
name: full-link-stock-analysis
description: 全链路个股分析。覆盖实时盘面、盯盘择时、事件归因到深度研究...
---

# 全链路个股分析助手

## 触发词
个股分析、深度研究、看好还是看空、估值贵不贵...

## 工作流程
1. 用户输入股票后，先调 get_stock_summary 拿基本面
2. 然后调 get_stock_performance 看技术面
3. 如果用户问估值，调 get_stock_financials + 同业对比
4. 输出必须包含：基本面/技术面/资金面/风险提示
5. 涉及买卖建议必须加免责声明
```

### 1.2 TRAE 处理 Skill 的三步机制

```
用户输入 "宁德时代现在能买吗"
        │
        ▼
[1] TRAE 把所有 Skill 的 description 拼成一段话塞给 LLM
    "你有这些能力：..."
    LLM 选择匹配的 Skill
        │
        ▼
[2] 加载这个 Skill 的完整 Markdown 进上下文
        │
        ▼
[3] LLM 按这份 SOP 行动，触发 Function Call → 执行 MCP → 拿数据 → 继续
```

**核心认知**：Skill 是 **Prompt 的一种特殊形态**——按需加载的、结���化的 Prompt。它不是程序逻辑，是给 LLM 看的"工作手册"。LLM 读懂手册后，自己决定调哪个工具。

这种"按需加载"机制的好处是 **节省上下文窗口**：不把所有 Skill 都塞进去，只加载匹配到的那一份。

---

## 2. "不深入理解就出不来好效果"的真正解法

裸 LLM 直接回答用户一句"宁德时代能买吗"，效果一定差。解决这个问题的 **不是 Skill 本身，而是 Skill 里写的"追问/澄清/拆解"逻辑**。

### 2.1 好 Skill vs 差 Skill

**差的 Skill（只列工具，LLM 乱调一气）**：

```markdown
# 工作流程
调用 get_stock_summary、get_stock_financials、search_news...
```

**好的 Skill（定义了完整的思考流程）**：

```markdown
# 工作流程

## Step 0：意图澄清（必须执行）
当用户提问模糊时（比如"X 能买吗""X 怎么样"），不要直接答，先做：
1. 实体标准化：宁德时代 → 300750
2. 时间维度判断：问的是当下买卖？中长期持有？还是看财务？
3. 如果意图不清晰，反问用户

## Step 1：信息收集（按需调工具）
- 查基本面：get_stock_summary
- 查行情：get_stock_performance  
- 查财报：get_stock_financials
- 查资讯：search_news
（每个工具的调用时机说清楚）

## Step 2：分析框架
按 基本面/技术面/资金面/消息面 四个维度组织分析

## Step 3：输出规范
- 必须引用数据来源
- 买卖建议必须附���险提示
- 不要预测具体价位
```

### 2.2 "深入理解"的三个实现支柱

| 支柱 | 作用 | 在 Skill 中对应 |
|---|---|---|
| **System Prompt / SOP** | 把"应该怎么思考"写清楚 | Skill 正文 |
| **意图分类与澄清** | 先把用户意图标准化 | Step 0 |
| **工具调用时机约束** | 什么情况下调什么工具 | Step 1 |

---

## 3. Skill ��自研系统中的对应物

在 Spring AI Alibaba 里，Skill 等价物是 **Advisor（拦截器）+ System Prompt + 动态 Prompt 模板**。

```java
@Configuration
public class StockAgentConfig {

    @Bean
    public ChatClient stockAgent(ChatClient.Builder builder) {
        return builder
            .defaultSystem(stockAnalysisSop())  // 等价于 Skill 的正文
            .defaultTools(allMcpTools())        // 等价于 MCP 工具集
            .defaultAdvisors(
                new IntentClarifyAdvisor(),     // 等价于 Step 0：意图澄清
                new RiskGuardAdvisor()          // 等价于 Step 3：风控
            )
            .build();
    }

    private String stockAnalysisSop() {
        return """
            你是专业A股分析师。流程：
            1. 先标准化股票代码
            2. 意图不明时反问
            3. 按 基本面/技术面/资金面/消息面 收集
            4. 输出附数据来源和风险提示
            """;
    }
}
```

---

## 4. Skill vs Subagent：两个完全不同的机制（重点）

很多人把这两个混了，但它们在 TRAE 里是**两个独立的机制**：

| 维度 | **Skill** | **Subagent** |
|---|---|---|
| **本质** | 一段 SOP/工作指南（给当前 Agent 看的 Prompt） | 一个独立的"小 Agent"（独立上下文窗口） |
| **执行方式** | 在**当前**对话上下文里执行 | 在**独立**的上下文窗口里执行 |
| **谁来调度** | 当前 Agent（任何对话都能用） | 只有内置 "Agent" 智能体能调 |
| **上下文污染** | 会（Skill 读的数据进入主上下文） | 不会（Subagent 的中间过程隔离） |
| **适合场景** | 流程性指导、分析方法、输出格式 | 大量检索/分析、需要上下文隔离的专业任务 |
| **文件位置** | `~/.trae-cn/skills/` 或插件内 `skills/` | `~/.trae-cn/agents/` |
| **文件格式** | `SKILL.md`（YAML + Markdown） | `{name}.md`（YAML + Markdown） |

### 用一句话区分

> **Skill 是"教科书"，Subagent 是"另一个人"。**
> - Skill：当前这个 AI 临时学了一本《如何做基本面分析》的教材，自己干
> - Subagent：当前这个 AI 叫来一个"基本面分析师同事"，让同事干完把结果交回来

### 重要澄清：Subagent 是预先写好的，不是动态生成

官方文档原文（[子智能体 Subagent](https://docs.trae.cn/ide/subagents)）：

> "子智能体（Subagent）是一种通过 **Markdown 文件定义**的专用智能体。"

Subagent 是预先写好的 `.md` 文件，放在固定目录里。**不是每次对话临时生成的**。动态的是"主 Agent 这次要不要调它"（靠 description 语义匹配），被调用的 Subagent 本身是静态写好的。

| 事项 | 是不是动态 |
|---|---|
| Subagent 的定义文件（角色、工具、Prompt） | ❌ 静态，预先写好 |
| 主 Agent 本次要不要调它 | ✅ 动态，按对话内容匹配 description |
| Subagent 内部怎么执行任务 | ✅ 动态，LLM 自己推理 |

### 容易混淆：TRAE 还有"Task 子代理"（真动态）

| 机制 | 动态吗 | 说明 |
|---|---|---|
| Subagent | ❌ 静态文件 | 预先写好，主 Agent 按需调 |
| Skill | ❌ 静态文件 | 同上 |
| Task 工具派的子代理 | ✅ **真动态** | 主 Agent 用 Task 工具临时起一个子代理，任务描述当场写 |

**区别**：Subagent 是可复用的专业角色（永久存在），Task 子代理是一次性的临时工（用完即弃）。

---

## 5. 多 Agent 协作机制：拆分原则

### 5.1 拆分四原则

**原则一：按"专业领域"拆，不按"工具"拆**

❌ 错误拆法（按工具）：行情Agent、财报Agent、新闻Agent...
✅ 正确拆法（按专业视角）：基本面Agent、技术面Agent、资金面Agent、风控Agent

每个 Agent 是"一个完整视角的分析师"，不是"一个工具的操作员"。

**原则二：按"上下文是否需要隔离"拆**

如果两个子任务的中间过程会互相污染，就拆出去。如果只是调个简单工具拿个数字，不拆。

**原则三：按"能否并行"拆**

基本面、技术面、资金面分析彼此独立 → 并行执行，大幅提速。

**原则四：按"输出形态不同"拆**

写报告、生成图表、做风险审核，输出格式差异大的任务适合拆。

### 5.2 股票系统的推荐拆法

```
主Agent（Orchestrator）
  职责：理解需求、拆任务、汇总
  
├─ 信息收集（可选，简单场景可省略，单 Agent 即可）
├─ 基本面Agent：财务健康度、估值、护城河
├─ 技术面Agent：趋势、支撑压力、指标信号
├─ 资金面Agent：主力动向、筹码结构
├─ 消息面Agent：近期催化与风险事件
└─ 风控审核Agent（最后一步，强制执行，必须独立上下文）
```

---

## 6. 独立上下文：不用 Subagent 也能实现的 5 种方式（重点）

> **关键认知：Subagent 把"独立上下文+独立Prompt+工具白名单+模型选择+结果回传"打包在一起，但"独立上下文"本身可以单独实现。**

自研时实现"独立上下文"的 5 种方式：

### 方式 1：新建独立的 ChatClient（最直接，等价于 Subagent）

```java
@Component
public class FundamentalAnalysisService {
    private final ChatClient fundamentalClient;
    
    public FundamentalAnalysisService(ChatClient.Builder builder) {
        this.fundamentalClient = builder
            .defaultSystem("你是基本面分析师，专注财报和估值...")
            .defaultTools(financialTools())
            .build();
    }
    
    public String analyze(String stockCode) {
        // 调用结束后，上下文不残留到主对话
        return fundamentalClient.prompt()
            .user("分析 " + stockCode + " 的基本面")
            .call()
            .content();
    }
}
```

**特点**：完全等���于 Subagent，最简单可控。适用：**需要复用的专业角色**。

### 方式 2：无状态函数调用（最轻量，不进任何上下文）

```java
public String queryFinancials(String stockCode) {
    return client.prompt()
        .system("从以下数据中提取关键财务指标，返回 JSON")
        .user(mcpService.call("get_stock_financials", stockCode))
        .call()
        .content();  // 只取结果字符串
}
```

**特点**：上下文最干净。适用：**工具结果的格式化、清洗**。

### 方式 3：多轮对话的"会话隔离"（按用户/任务隔离）

```java
@PostMapping("/chat")
public String chat(@RequestBody ChatRequest req) {
    return chatClient.prompt()
        .user(req.getMessage())
        .advisors(new MessageChatMemoryAdvisor(chatMemory, req.getSessionId()))
        .call()
        .content();
}
```

**特点**：解决"用户 A 的对话污染用户 B"。适用：**多用户系统的会话管理**。

### 方式 4：Advisor 拦截器做上下文裁剪（不隔离，但清理）

```java
public class ContextFilterAdvisor implements BaseAdvisor {
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest req, CallAroundAdvisorChain chain) {
        AdvisedRequest cleaned = req.mutate()
            .user(filterNoise(req.userText()))
            .build();
        return chain.nextAroundCall(cleaned);
    }
}
```

**特点**：不是真隔离，是"打扫"。适用：**单 Agent 内部的小幅上下文治理**。

### 方式 5：Map-Reduce 模式（并行独立 + 汇总）

```java
public String deepAnalysis(String stockCode) {
    CompletableFuture<String> fundamental = supplyAsync(() -> 
        fundamentalService.analyze(stockCode));
    CompletableFuture<String> technical = supplyAsync(() -> 
        technicalService.analyze(stockCode));
    CompletableFuture<String> capital = supplyAsync(() -> 
        capitalService.analyze(stockCode));
    
    CompletableFuture.allOf(fundamental, technical, capital).join();
    
    // 主上下文只接收三个结论字符串，不接收中间数据
    return aggregatorClient.prompt()
        .user(String.format("基本面：%s\n技术面：%s\n资金面：%s\n请汇总",
            fundamental.join(), technical.join(), capital.join()))
        .call()
        .content();
}
```

**特点**：完美解决"上下文污染"+"并行加速"。适用：**深度分析报告、多维度综合判断**。

### 5 种方式对比

| 方式 | 隔离程度 | 实现复杂度 | 灵活度 | 适用场景 |
|---|---|---|---|---|
| **1. 独立 ChatClient** | ⭐⭐⭐⭐⭐ | 低 | 高 | 复用的专业角色（≈ Subagent） |
| **2. 无状态函数** | ⭐⭐⭐⭐⭐ | 极低 | 低 | 数据清洗、格式化 |
| **3. 会话隔离** | ⭐⭐⭐⭐ | 中 | 中 | 多用户/多租户 |
| **4. Advisor 裁剪** | ⭐⭐ | 低 | 中 | 轻度污染治理 |
| **5. Map-Reduce** | ⭐⭐⭐⭐⭐ | 高 | 高 | 深度分析、并行任务 |

### 关键认知升级

**"拆分任务"和"隔离上下文"是两个独立诉求，可以单独解决：**

| 诉求 | 解决什么 | 实现方式 |
|---|---|---|
| **拆分任务** | 任务太复杂，一个 Agent 干不完 | Skill 分阶段 / Map-Reduce |
| **隔离上下文** | 防止污染、防止串扰 | 独立 ChatClient / 会话隔离 |

**Subagent 是把这两个诉求打包在一起的方案。但自研时，可以独立解决任一个：**
- 只想拆任务，不想隔离 → 用 Skill（动态 Prompt）
- 只想隔离，不想拆任务 → 用独立 ChatClient 做单一任务
- 两个都要 → 才用 Subagent 模式（独立 ChatClient + 独立 Prompt + 独立工具）

---

## 7. 静态编排 vs 动态编排：90% 场景应该写死（重点）

> **核心判断：自己实现时，编排该写死还是动态？答案：大部分写死，关键决策点动态。**

### 7.1 判断框架：看任务边界

```
任务边界清晰吗？
       │
       ├─ 清晰（流程固定、可预判）→ 写死（静态编排）
       │  例：基本面分析、风控审核、盘后复盘
       │
       └─ 不清晰（开放性强、用户随意问）→ 动态（LLM 路由）
          例：闲聊式的投资咨询、多变的研报需求
```

### 7.2 静态 vs 动态的本质区别

**静态编排（写死）= 你是导演，LLM 是演员**：流程是代码写死的，LLM 只在每个节点内部做具体推理。

**动态编排 = LLM 是导演，自己决定怎么拆**：用一个小 LLM 先判断路由，再决定调哪些子任务。

### 7.3 为什么"写死"反而更好（反直觉的结论）

| 维度 | 静态写死 | 动态路由 |
|---|---|---|
| 同样的输入 | 永远走相同流程 | 可能走不同流程（路由 LLM 误判） |
| 出问题排查 | 看 DAG 就知道哪一步错了 | 要回放路由决策，难复现 |
| 合规审计 | 流程固定，能画清责任链 | 流程飘忽，审计难 |
| 成本和延迟 | 直接开干 | 多调一次 LLM 判断路由 |
| 稳定性 | 高 | 路由 LLM 会误判和漂移 |
| 调试迭代 | 改一个节点只影响那个节点 | 改路由 Prompt 整个系统行为都变 |

**金融场景出问题要担责，流程必须可追溯。静态编排天然可追溯。**

### 7.4 动态什么时候才用？三个前提

1. **任务组合真的不可预判**：开放性问答，用户意图模糊
2. **分支数量多且组合多样**：80+ Skill，硬写 if-else 写不过来
3. **用户期待"灵活"而非"标准化"**：个人助手要自适应

### 7.5 实战建议：混合模式（推荐）

实际工程里几乎都是**混合模式**——大部分写死，入口层动态分类。

```
┌─────────────────────────────────────────────────┐
│  L1：意图分类（动态，只判断大类）                │
│  ─ 用小模型（qwen-turbo），快、便宜              │
│  ─ 输出：属于"个股分析" / "复盘" / "研报" / ...  │
└────────────────────┬────────────────────────────┘
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
┌───────────┐ ┌───────────┐ ┌───────────┐
│ 个股分析   │ │ 盘后复盘   │ │ 生成研报  │
│ (静态流程) │ │ (静态流程) │ │ (静态流程)│
└───────────┘ └───────────┘ └───────────┘
```

**核心原则：第一层动态分类（入口），之后每条分支都是静态流程。**

### 7.6 混合模式代码实现

```java
@Service
public class StockAgentOrchestrator {
    
    private final IntentClassifier classifier;  // L1：动态意图分类
    private final Map<String, AnalysisFlow> flows;  // 各场景的静态流程
    
    public String handle(String userQuery, String sessionId) {
        String intent = classifier.classify(userQuery);
        AnalysisFlow flow = flows.get(intent);
        return flow.execute(userQuery, sessionId);
    }
}

@Component
public class IntentClassifier {
    private final ChatClient classifier;  // 用便宜的小模型
    
    public String classify(String query) {
        return classifier.prompt()
            .system("""
                判断用户意图，只返回以下之一：
                - stock_analysis：个股分析
                - market_review：盘面/复盘
                - report：生成深度研报
                - news：查资讯/公告
                - chitchat：闲聊
                """)
            .user(query)
            .call()
            .content();
    }
}

// 每条分支是静态流程（写死）
@Component
public class StockAnalysisFlow implements AnalysisFlow {
    private final DataFetchService dataService;
    private final FundamentalService fundamental;    // 独立上下文
    private final TechnicalService technical;        // 独立上下文
    private final CapitalService capital;            // 独立上下文
    private final RiskService risk;                  // 必须独立上下文
    private final ChatClient aggregator;
    
    public String execute(String query, String sessionId) {
        String stockCode = extractCode(query);
        Map<String, String> rawData = dataService.fetch(stockCode);
        
        // 并行独立分析
        var f = fundamental.analyze(rawData);
        var t = technical.analyze(rawData);
        var c = capital.analyze(rawData);
        
        String summary = aggregator.prompt()
            .user(format("汇总：基本面{}/技术面{}/资金面{}", f, t, c))
            .call().content();
        
        return risk.review(summary);  // 风控独立审核
    }
}
```

---

## 8. 完整决策清单：什么时候独立上下文 + 什么编排方式

| 任务特征 | 要不要独立上下文 | 编排方式 |
|---|---|---|
| 风控审核 | ✅ **必须**（不能被乐观分析污染） | 静态，写死 |
| 专业角色分析（基本面/技术面） | ✅ 推荐（避免数据污染主上下文） | 静态，写死 |
| 多维度并行分析 | ✅ 必须（并行天然要独立） | 静态，Map-Reduce |
| 单纯数据查询 | ❌ 不用（返回字符串就行） | 无状态函数 |
| 简单的对话式问答 | ❌ 不用（保持上下文连贯） | 单 Agent |
| 闲聊/通用咨询 | ❌ 不用 | 单 Agent |
| 开放式研究（"帮我调研新能源行业"） | ⚠️ 视情况 | 半动态 |

### 股票系统的推荐组合

```
日常问答（90% 场景）
  → 单 Agent + 多 Skill（动态 Prompt 模板）
  → 不需要独立上下文，加载快、响应快

中度分析（需要查多类数据）
  → 方式 1（独立 ChatClient） 
  → 比如"基本面分析"用独立的 fundamentalClient

深度报告（需要多维度综合）
  → 方式 5（Map-Reduce）
  → 基本面/技术面/资金面并行独立分析，主 Agent 只汇总结论

风控审核（必须独立，防污染）
  → 方式 1（独立 ChatClient）
  → 风控 Agent 绝对不能用被分析数据污染的上下文
```

---

## 9. 一句话总结

- **Skill 是给 LLM 看的工作手册，不是代码**。好 Skill 的核心是写清楚"先澄清意图，再分类收集，再按框架分析，再规范输出"。
- **Skill 和 Subagent 是两个独立机制**：Skill 在当前上下文执行，Subagent 在独立上下文执行。
- **Subagent 是预先写好的静态文件**，动态的是"主 Agent 要不要调它"。
- **独立上下文 ≠ 必须 Subagent**：有 5 种实现方式，独立 ChatClient 是最直接的等价物。
- **"拆分任务"和"隔离上下文"是两个独立诉求**，可以单独解决。
- **90% 企业场景应该静态编排（写死）**，只在入口层用动态意图分类。金融场景流程必须可追溯，静态编排天然合规。

---

## 关联笔记

- [AI 应用落地技术栈全景（总览）](../AI应用落地技术栈全景.md)
- [TRAE vs 自研 AI Agent 差异与选型](../TRAE与自研AI-Agent差异与选型.md)
- [AI 落地决策框架与组合策略](../AI落地决策框架与组合策略.md)
- [Wind AIFin Skill 能力清单（业务资产）](Wind金融Skill能力清单.md)
- [股票 AI 系统设计（实战案例）](../../09-业务项目/stock-ai-system/股票AI问答系统设计.md)

## 参考资料

- [TRAE 官方文档 - 技能（Skill）](https://docs.trae.cn/ide/skills)
- [TRAE 官方文档 - 子智能体（Subagent）](https://docs.trae.cn/ide/subagents)
- [TRAE 官方文档 - 智能体概述](https://docs.trae.cn/ide/agent-overview)
