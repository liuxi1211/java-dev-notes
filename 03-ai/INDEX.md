# 03-ai 目录索引

> AI 工程笔记总索引。子目录说明见 [README.md](README.md)。

## Harness 工程系列（2026-08-26 · 四篇）

源自同一次深度对话，互相引用，建议按序阅读：

1. [Harness-Engineering驾驭工程全景](agent/Harness-Engineering驾驭工程全景.md) — 定义、三阶段演进、协议栈（MCP/A2A/AGENTS.md/Skills）、各家差异与行业收敛现状
2. [Claude-Code-Harness深度拆解](agent/Claude-Code-Harness深度拆解.md) — 九层 harness 模型、Plan Mode 最小可行 spec 哲学、子 Agent 四种编排模式（串行/并行/层级/迭代）
3. [A2A落地边界与多Agent决策框架](agent/A2A落地边界与多Agent决策框架.md) — 多 Agent ≠ A2A 的三层拆解、A2A vs MCP、隐性成本与拆分决策树
4. [SDD规范驱动开发与过度spec化](agent/SDD规范驱动开发与过度spec化.md) — SDD 三层产物、vs TDD/BDD、Claude Code 不做 SDD 的四个原因、过度 spec 化四种形态与检查清单
5. [SDD本质论-定义、收益机理与演进史](agent/SDD本质论-定义、收益机理与演进史.md) — 补充篇：第一性定义（外化/可裁决/最高约束三要素）、五大收益机理、七十年演进脉络（图纸→瀑布→敏捷→TDD→AI-SDD）
6. [SDD开发实践总结](agent/SDD开发实践总结.md) — 四维实践复盘：核心流程、真实帮助、五个坑与改进、分档落地建议

## agent/ — Agent 设计与协作

- [Agent与Loop的本质](agent/Agent与Loop的本质.md) — Agent = LLM + 工具 + 自主决策循环；It's just loops
- [MCP-CLI-Skill三种接入模式对比](agent/MCP-CLI-Skill三种接入模式对比.md)
- [Skill编排与多Agent协作机制](agent/Skill编排与多Agent协作机制.md) — Skill 是 Prompt 不是代码；静态编排优先
- [SubAgent上下文隔离与蒸馏实践](agent/SubAgent上下文隔离与蒸馏实践.md)
- [自研CLI接入Agent实践](agent/自研CLI接入Agent实践.md)

## llm/ — 大模型基础

- [Chat API多轮对话机制与上下文管理](llm/Chat%20API多轮对话机制与上下文管理.md)

## 根目录 — 其他主题

- [AI应用落地技术栈全景](AI应用落地技术栈全景.md)
- [AI落地决策框架与组合策略](AI落地决策框架与组合策略.md)
- [AI编程验收链路分析-Playwright原理与E2E瓶颈](AI编程验收链路分析-Playwright原理与E2E瓶颈.md)
- [AI前端开发中的设计稿还原问题与解法](AI前端开发中的设计稿还原问题与解法.md)
- [TRAE与自研AI-Agent差异与选型](TRAE与自研AI-Agent差异与选型.md)
- [SpringAI与SpringAIAlibaba选型](SpringAI与SpringAIAlibaba选型.md)
- [Wind金融Skill能力清单](Wind金融Skill能力清单.md)
- [火山引擎 API 文档](火山引擎%20API%20文档.md)
