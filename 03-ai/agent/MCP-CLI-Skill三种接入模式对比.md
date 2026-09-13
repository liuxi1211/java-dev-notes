---
tags: [llm, agent, mcp, cli, skill]
date: 2026-08-25
source: ai-conversation
---

# MCP-CLI-Skill三种接入模式对比

> Agent 接入外部能力的三条路线：MCP 协议、CLI 命令行、Skill 知识注入。核心问题：为什么 GitLab/飞书选择 CLI？MCP 的价值如何被 Skill 蚕食？相关笔记：[Agent与Loop的本质](Agent与Loop的本质.md)、[自研CLI接入Agent实践](自研CLI接入Agent实践.md)

## 1. 为什么平台选择 CLI 而不是 MCP

GitLab（glab）、GitHub（gh）、飞书（lark-cli）等押注 CLI，因为 **CLI 是 Agent 的最大公约数**：所有编码类 Agent 都有 shell 执行能力——一次开发，处处可用，零接入成本。

### CLI 的标准契约（40 年沉淀，已写入 LLM 训练数据）

| 契约 | 内容 |
|---|---|
| 自描述 | `-h` / `--help`，Agent 可自探索用法 |
| 参数语法 | `-v` 短选项、`--verbose` 长选项、`--` 分隔符 |
| 三流分离 | stdin/stdout/stderr，报错走 stderr |
| 管道组合 | `\| jq` `&&` `xargs` 自由组合（MCP 做不到） |
| 退出码 | exit 0 成功，非 0 失败，Agent 无需解析自然语言 |
| 配置认证 | 环境变量、配置文件、`auth login` 子命令 |

### 自探索闭环（CLI 对 Agent 自文档化的原因）

```
不认识 glab？→ glab --help → 看到子命令
不知道参数？→ glab mr create --help
失败了？    → 读 stderr + 退出码 → 自纠错
```

### MCP 的现实痛点

- 生态碎片化：各客户端实现有差异
- 上下文开销：工具 schema 常驻挤爆上下文
- 部署复杂：单独进程、配置、授权
- 调试隔着协议层

### 能力阶梯（CLI 方案的完整版图）

| 层级 | 方式 | 前提 |
|---|---|---|
| 1 | 官方 CLI（glab） | 平台提供 CLI |
| 2 | 通用 HTTP CLI（curl + API 文档） | 平台有 REST API |
| 3 | 临时脚本（python -c / node -e） | 有运行时 |

第 2 层是底气：HTTP API 覆盖率远大于 MCP Server。

### MCP 守住的阵地（CLI 替代不了）

- **凭证托管**：token 在 Server 端，不落到 Agent 环境（企业安全刚需）
- **流式/实时**：SSE、WebSocket、订阅推送
- **有状态会话**：连接池、事务、游标
- **结构化发现**：工具列表自描述

现实最优解常是混合：简单 CRUD 走 CLI/curl，涉密/流式走 MCP。

> 附：企业微信无官方 CLI，接入靠 curl 直连 API / 企业自建封装 / 社区 MCP。若未来有 CLI，必然独立分发（服务器/CI/Agent 环境不装 GUI）。

## 2. Skill 的角色：解决"知道"，CLI 解决"做到"

**能力存在 ≠ Agent 知道存在**。CLI 装在机器上，上下文里没提，Agent 就不会用。

```
Skill（Markdown 说明书）→ 解决"知道"：何时用、怎么装、常用命令、坑
CLI（可执行程序）      → 解决"做到"：真正执行
--help 自描述          → 兜底：现场探索
```

## 3. MCP 价值被 Skill 蚕食的部分

| MCP 的三层价值 | Skill 能否替代 | 原因 |
|---|---|---|
| 工具发现 | ✅ 大部分替代 | 索引+按需加载 vs schema 常驻 |
| 用法知识 | ✅ 替代甚至更强 | 自然语言能表达经验/坑/流程，JSON Schema 只能描述参数 |
| 凭证托管/流式/有状态 | ❌ 不可替代 | CLI 凭证必然落在 Agent 环境 |

**蚕食的含义**：2024 年接入默认答案"做个 MCP Server"→ 2025 年变成"有 CLI 就给 CLI + Skill，只有涉密/流式才上 MCP"。默认选项变了。

## 4. 为什么 MCP 难做按需加载

MCP 协议：连接时 `tools/list` 全量上报，schema 注入上下文——**"感知"和"占用"绑死**。几百命令 × 200-500 token/工具 = 几万~十几万 token，连接即付费。模型不知道工具存在就不会调用，知道就得先加载（鸡生蛋问题）。

补救方向（进行中）：MCP Tool Search 提案（只上报一个 `search_tools` 元工具，按需搜索）、客户端延迟加载。但命令空间越大，schema 维护成本越高于一份 Markdown——结构性劣势不变。

### Progressive Disclosure（渐进式披露）

| 维度 | MCP schema | Skill |
|---|---|---|
| 上下文占用 | 常驻，随工具数线性膨胀 | 索引常驻（几十 token），全文按需 |
| 扩展性 | 装多了挤爆 | 可挂几百个 skill |
| 新增能力成本 | 改 Server 代码、重新部署 | 改一份 Markdown |

## 5. 平台型产品（TRAE 等）的实际架构

TRAE 对用户暴露 MCP 作为统一接入协议（应用商店货架标准：分发/授权/计费/安全/跨端），但货架上的商品内部实现自由：

```
 ├── 纯 MCP 路线：ifind、tdx 等数据类（token、流式行情）
 ├── MCP壳+CLI+Skill：lark 系列（MCP 做发现授权，CLI 承载命令，Skill 承载用法）
 └── 纯 Skill 路线：frontend-design 等知识注入类
```
