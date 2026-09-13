---
title: Wind AIFin Skill 能力清单：80+ 股票分析 Skill 的业务资产盘点
date: 2026-08-10
tags: [ai, skill, 股票, wind, mcp, 业务资产, prompt-engineering]
source: ai-conversation
related:
  - agent/Skill编排与多Agent协作机制.md
  - AI应用落地技术栈全景.md
  - ../09-业务项目/stock-ai-system/股票AI问答系统设计.md
---

# Wind AIFin Skill 能力清单：80+ 股票分析 Skill 的业务资产盘点

> 这份笔记盘点 Wind AIFin 插件（wind-aifin 0.0.2）提供的 80+ 个 Skill 的能力分类。
> **价值**：这些 Skill 是精心写的分析方法 SOP（Prompt 工程），自研股票 AI 系统时可以直接复用作为 Prompt 模板。
>
> 来源：基于 `~/.trae-cn/plugins/trae-remote-official/wind-aifin/0.0.2/skills/` 实际文件盘点。

---

## 0. 先搞清楚：这些是 Skill，不是 Subagent

经过文件系统盘点（详见 [Skill 编排笔记 §4](agent/Skill编排与多Agent协作机制.md)），Wind AIFin 这 80+ 个能力**全部是 Skill，没有 Subagent**。

| 机制 | 本插件用了多少 |
|---|---|
| Skill（SKILL.md） | **80+ 个** |
| Subagent | **0 个** |
| MCP Server（工具层） | 1 套（wind-mcp） |

**这印证了一个重要结论**：股票分析场景**主要用 Skill（动态 Prompt 模板）而非 Subagent（独立上下文）**。自研时应该照搬这个思路——单 Agent + 多 Skill 覆盖 90% 场景。

---

## 1. 能力分类总览

按业务用途分 10 大类：

| 类别             | Skill 数量 | 核心价值                |
| -------------- | -------- | ------------------- |
| 📈 选股筛选        | ~10      | 批量筛出候选股池            |
| 📊 个股深度分析      | ~10      | 单只股票多维度研究           |
| 💼 交易决策与执行     | ~10      | 买入/加仓/止盈止损的决策框架     |
| 📅 财报与事件解读     | ~10      | 财报季、公告、事件冲击分析       |
| 🧠 盘后复盘与盯盘     | ~8       | 日常节奏管理              |
| 🎯 大师思维 Avatar | 4        | 巴菲特/芒格/塔勒布/纳瓦尔的思维框架 |
| 🔍 估值与定价       | 2        | 估值框架 + DCF 建模       |
| 📰 资讯与公告摘要     | 3        | 批量信息消化              |
| 🏭 行业与板块       | ~5       | 产业信息、轮动、主线          |
| 🛠️ 工具与底座      | ~10      | 数据查询、报告生成、回测等       |

---

## 2. 选股筛选类（Batch Screening）

按不同策略批量筛出候选股池。

| Skill 名 | 用途 |
|---|---|
| `canslim_growth_scan_skill` | CANSLIM 成长股框架批量筛选 |
| `high_quality_compounder_finder_skill` | 高资本回报 + 护城河的复利资产筛选 |
| `value_dividend_candidate_skill` | 价值股息候选（安全边际 + 股息可持续） |
| `breakout_candidate_finder_skill` | 突破形态成熟的候选股筛选 |
| `pullback_opportunity_finder_skill` | 回调充分但趋势未坏的候选筛选 |
| `vcp_breakout_scan_skill` | 波动收敛（VCP）形态扫描 |
| `dividend_growth_entry_skill` | 股息持续增长 + 估值回落的候选 |
| `earnings_momentum_setup_skill` | 财报后业绩+指引共振的强势股 |
| `breakout_candidate_finder_skill` | 突破候选（重复） |
| `turnaround_story_validation_skill` | 困境反转股的验证 |

**自研映射**：这类 Skill 对应"批量调用 screener 工具 + 按策略过滤 + 输出候选池表格"，适合做成**定时任务**或**批量 API**。

---

## 3. 个股深度分析类（Single Stock Deep Dive）

单只股票的多维度研究，是日常最高频的场景。

| Skill 名 | 用途 |
|---|---|
| `stock_first_look_skill` | 首次接触个股的快速认知（业务/关注点/估值/风险） |
| `hot_stock_quick_read_skill` | 热门股的极简解读（业务/催化/预期/风险） |
| `business_model_decoder_skill` | 商业模式拆解（获客/交付/定价/扩张） |
| `moat_strength_review_skill` | 护城河评估（来源/强度/可持续性） |
| `growth_quality_check_skill` | 增长质量检查（含量/现金/可持续性） |
| `management_quality_check_skill` | 管理层质量（背景/激励/资本配置） |
| `peer_comparison_decision_skill` | 同业横向对比（二选一/优中选优） |
| `valuation_snapshot_skill` | 估值快照（高低/分位/重估条件） |
| `sec_filing_question_answer_skill` | 从 10-K/10-Q/招股书定位依据 |
| `shareholder_letter_digest_skill` | 股东信要点提炼 |

**自研映射**：这类 Skill 是"单 Agent + 多 Skill 模板"的核心用户，每个对应一个 Prompt 模板，按用户意图动态加载。

---

## 4. 交易决策与执行类（Decision & Execution）

具体的买卖决策框架，纪律性极强。

| Skill 名 | 用途 |
|---|---|
| `trade_plan_builder_skill` | 完整交易计划（入场/仓位/止损止盈/应急） |
| `position_sizing_decision_skill` | 单笔仓位大小决策（风险预算） |
| `position-sizer` | 仓位计算器（ATR/Kelly） |
| `stop_loss_discipline_skill` | 止损规则设计（价格/逻辑/时间止损） |
| `take_profit_ladder_skill` | 分批止盈路径设计 |
| `dip_buy_decision_skill` | 下跌承接决策（试错/分批/放弃） |
| `add_to_winner_decision_skill` | 盈利仓加仓决策（前提/节奏/边界） |
| `trim_or_hold_decision_skill` | 减仓还是持有决策 |
| `premarket_trade_checklist_skill` | 盘前交易逐项核查清单 |
| `price_target_reach_alert_skill` | 触达目标价的动作建议 |

**自研映射**：这类 Skill 必须有**强约束的输出格式**（明确的动作建议 + 风险提示），适合用 Advisor 做后置审核。

---

## 5. 财报与事件解读类（Earnings & Events）

| Skill 名 | 用途 |
|---|---|
| `earnings_preview_skill` | 财报前的预期/看点/情景推演 |
| `earnings-analysis` | 财报后的专业研究报告（8-12 页） |
| `earnings_reaction_interpreter_skill` | 财报后涨跌反应解读 |
| `conference_call_takeaway_skill` | 电话会议要点提炼 |
| `guidance_change_impact_skill` | 业绩指引上修/下修的影响 |
| `dividend_change_explainer_skill` | 分变变动解读 |
| `buyback_program_reviewer_skill` | 回购计划评估 |
| `major_announcement_impact_skill` | 并购/减持/定增/合同等公告影响 |
| `trading_halt_resume_tracker_skill` | 停复牌跟踪 |
| `pead_opportunity_skill` | 财报后漂移（PEAD）机会 |

**自研映射**：这类 Skill 强依赖**事件触发**（财报日、公告日），适合做成**事件驱动架构**。

---

## 6. 盘后复盘与盯盘类（Daily Routine）

| Skill 名 | 用途 |
|---|---|
| `post-market-debrief` | 盘后复盘（发生了什么/信号/明天怎么接） |
| `daily_watchlist_morning_brief_skill` | 盘前简报（隔夜变化/今日重点） |
| `after_close_watchlist_recap_skill` | 收盘自选股复盘 |
| `watchlist_news_impact_digest_skill` | 自选股新闻批量消化 |
| `intraday_abnormal_move_alert_skill` | 盘中异动预警 |
| `volume_spike_reasoning_skill` | 放量归因 |
| `gap_open_interpreter_skill` | 高开/低开解读 |
| `support_break_warning_skill` | 支撑位预警 |

**自研映射**：这类 Skill 是**高频定时任务**的典型用户——盘前 9:00、午盘 11:30、收盘 15:00 自动触发。

---

## 7. 大师思维 Avatar 类（Thinking Frameworks）

这是最有特色的一类，把经典投资大师的思维模型结构化成 Prompt。

| Skill 名 | 模拟人物 | 核心框架 |
|---|---|---|
| `avatar-warren-buffett-investing` | 巴菲特 | 能力圈 + 护城河 + 管理层诚信 + 所有者收益 |
| `avatar-charlie-munger-thinking` | 芒格 | 逆向思考 + 激励分析 + 多学科模型 |
| `avatar-nassim-taleb-risk` | 塔勒布 | 尾部风险 + 利益共担 + 杠铃策略 |
| `avatar-naval-ravikant-thinking` | 纳瓦尔 | 杠杆 + 特定知识 + 复利 |

**自研映射**：这类 Skill 可以作为**多视角分析**的"滤镜"——同一只股票，让四个 Avatar 各自给观点，再综合。适合 Map-Reduce 编排。

---

## 8. 估值与定价类（Valuation）

| Skill 名 | 用途 |
|---|---|
| `valuation-pricing-framework` | 估值与定价框架（总览） |
| `valuation_snapshot_skill` | 估值快照（个维度） |
| `dcf-model` | 完整 DCF 模型（含 Excel 输出） |

**自研映射**：DCF 这种需要生成可下载 Excel 的，是**重度任务**，适合用 Subagent / 独立 ChatClient + 文件生成。

---

## 9. 资讯与公告摘要类（News Digest）

| Skill 名 | 用途 |
|---|---|
| `policy_headline_interpreter_skill` | 政策新闻解读（受益方向/噪音区分） |
| `macro_event_market_impact_skill` | 宏观事件影响解读 |
| `earnings_calendar_planner_skill` | 财报季重点公司时间轴 |

---

## 10. 行业与板块类（Sector & Industry）

| Skill 名 | 用途 |
|---|---|
| `sector_rotation_radar_skill` | 板块轮动方向识别 |
| `theme_heat_tracker_skill` | 题材热度跟踪 |
| `theme_leader_identification_skill` | 题材龙头识别 |
| `theme-detector` | 趋势主题检测 |
| `industry_chain_signal_skill` | 产业链上下游信号 |
| `a-share-primary-theme-identification` | A 股主线识别 |
| `industry-research` | 完整行业研究（含交互图表） |

---

## 11. 工具与底座类（Infrastructure）

| Skill 名 | 用途 |
|---|---|
| `wind-mcp-skill` | Wind MCP 数据查询路由入口 |
| `wind-find-finance-skill` | Wind 能力发现与安装 |
| `wind-alice` | Alice Agent CLI 工具 |
| `tushare-finance` | Tushare 数据查询 |
| `backtest-expert` | 策略回测专家 |
| `stock_research_memo_writer_skill` | 研究备忘录生成 |
| `equity-investment-thesis` | 个股投资逻辑深度研究 |
| `market-environment-analysis` | 市场环境综合分析 |
| `market_breadth_health_skill` | 市场广度健康度 |
| `market_regime_switch_skill` | 市场风格切换判断 |
| `market_sentiment_temperature_skill` | 市场情绪温度计 |
| `northbound_capital_flow_skill` | 北向资金流向 |
| `institutional_position_shift_skill` | 机构持仓变化 |

---

## 12. 从这些 Skill 学到的 Prompt 工程套路

看了这 80+ Skill 的结构，总结出可复用的 Prompt 模板套路：

### 套路 1：决策类 Skill 的标准结构

```
# {Skill 名}

## 适用场景
{什么时候用}

## 判断维度
1. {维度 A}：{怎么判断 + 关键指标}
2. {维度 B}：{怎么判断 + 关键指标}
3. {维度 C}：{怎么判断 + 关键指标}

## 输出格式
- 优先级排序
- 关键前提
- 风险点
- 跟踪重点

## 失效边界
{什么情况下结论不成立}
```

### 套路 2：筛选类 Skill 的标准结构

```
# {筛选策略名} 筛选

## 筛选条件
- 业绩：{具体门槛}
- 预期：{具体门槛}
- 相对强度：{具体门槛}
- 供需：{具体门槛}

## 输出
- 候选分层（A/B/C 级）
- 每个候选的核心理由 + 跟踪重点
```

### 套路 3：大师思维 Skill 的结构

```
# {人物名} 思维框架

## 核心原则
{3-5 条原则}

## 分析维度
1. {该人物最看重的维度 1}
2. {该人物最看重的维度 2}
...

## 判断标准
- 该人物会买入：{条件}
- 该人物会观望：{条件}
- 该人物会放弃：{条件}

## 常见认知偏误（该人物会警惕的）
{列出}
```

---

## 13. 自研时的复用策略

### 优先级 1：直接复用（高价值）

把这些 Skill 的 SKILL.md 打开，**把正文翻译成 Spring AI 的 System Prompt**：

- 大师思维 4 个（巴菲特/芒格/塔勒布/纳瓦尔）—— 思维框架通用
- 交易决策 10 个（交易计划/仓位/止损/止盈）—— 纪律性 Prompt
- 个股分析 10 个（商业模式/护城河/估值）—— 分析框架

### 优先级 2：改造复用（中等价值）

- 选股筛选类 —— 需要适配你自研系统的数据源
- 复盘盯盘类 —— 需要适配定时任务框架

### 优先级 3：参考学习（学习价值）

- 工具与底座类 —— 主要看人家的 Prompt 组织方式

### 具体操作建议

```bash
# 查看某个 Skill 的完整内容
cat ~/.trae-cn/plugins/trae-remote-official/wind-aifin/0.0.2/skills/avatar-warren-buffett-investing/SKILL.md

# 批量导出所有 Skill 的正文（用于自研系统复用）
for skill in ~/.trae-cn/plugins/trae-remote-official/wind-aifin/0.0.2/skills/*/SKILL.md; do
  echo "=== $(dirname $skill | xargs basename) ==="
  cat "$skill"
  echo ""
done > wind-skills-dump.md
```

---

## 14. 关键洞察

1. **Skill 数量多 ≠ 必须多 Agent**：80+ Skill 全部在单 Agent 架构下工作，证明大部分场景不需要独立上下文。
2. **这些 Skill 是业务知识的精华**：每个 Skill 都是专业分析师的方法论沉淀，比代码更值钱。
3. **Prompt 工程有套路**：决策类、筛选类、思维类都有标准结构，可以批量生产。
4. **自研时优先复用 Skill 内容**：不要重新发明轮子，把别人的 SKILL.md 翻译成你的 System Prompt 即可。

---

## 关联笔记

- [Skill 编排本质、多 Agent 协作与上下文隔离机制](agent/Skill编排与多Agent协作机制.md) — Skill/Subagent/独立上下文的方法论
- [AI 应用落地技术栈全景](AI应用落地技术栈全景.md) — 技术栈总览
- [股票 AI 系统设计（实战案例）](../09-业务项目/stock-ai-system/股票AI问答系统设计.md) — 完整设计
- [AI 落地决策框架](AI落地决策框架与组合策略.md) — 选型决策
