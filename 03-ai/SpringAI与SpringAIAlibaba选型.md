---
title: Spring AI vs Spring AI Alibaba：选型对比
date: 2026-08-10
tags: [ai, spring-ai, spring-ai-alibaba, java, 框架选型]
source: ai-conversation
related:
  - AI应用落地技术栈全景.md
  - AI落地决策框架与组合策略.md
---

# Spring AI vs Spring AI Alibaba：选型对比

> 这两个是 Java 团队做 AI 应用最主流的两个框架，这里做一个简洁的选型对比。更多 AI 落地内容见关联笔记。

---

## 1. 两者关系

- **Spring AI**：Spring 官方出品，1.0 已 GA。提供统一的 LLM 抽象（类似 Spring Data 统一数据库访问），是基础规范层。
- **Spring AI Alibaba**：阿里基于 Spring AI 的增强实现，集成通义千问、DashScope、百炼等阿里系 AI 服务，并提供 Graph（多 Agent 编排）、Prompt 模板、RAG 组件等开箱即用能力。

**关系**：Spring AI Alibaba 是 Spring AI 的超集 + 阿里生态适配。用了 Spring AI Alibaba，自动也用了 Spring AI。

---

## 2. 核心能力对比

| 能力 | Spring AI | Spring AI Alibaba |
|---|---|---|
| LLM 抽象（ChatClient） | ✅ | ✅ |
| Function Calling / @Tool | ✅ | ✅ |
| RAG（VectorStore、Retriever） | ✅ | ✅ |
| 多模型支持（OpenAI/Anthropic/Ollama...） | ✅ 国际模型为主 | ✅ 通义系为主，兼容国际 |
| **Graph（多 Agent 编排）** | ⚠️ 较弱 | ✅ 原生支持 |
| **阿里系集成**（通义/DashScope/百炼） | ❌ | ✅ 开箱即用 |
| Prompt Template / Nestable | ✅ | ✅ 增强 |
| 记忆与对话状态 | ✅ | ✅ |
| MCP Client | ✅ 1.x 支持 | ✅ |
| 中文社区与文档 | ⚠️ 国际为主 | ✅ 中文优先 |

---

## 3. 选型建议

### 国内 Java 团队 → **首选 Spring AI Alibaba**

理由：
1. 用通义千问等国内模型，开箱即用，不用自己封装
2. Graph 能力直接做多 Agent 编排
3. 中文文档和社区活跃
4. 阿里官方持续维护，国内合规

### 这些情况选纯 Spring AI

- 主要用 OpenAI / Anthropic / Ollama 等国际模型
- 不想依赖阿里生态
- 海外项目

### LangChain4j 何时考虑

- 想要更纯粹的不绑定 Spring 的 LLM 框架
- 团队习惯 LangChain 的 API 风格
- 需要一些 Spring AI 还没实现的能力（社区生态略领先）

---

## 4. 一个最小可运行骨架（Spring AI Alibaba）

```java
// pom.xml 核心依赖
// spring-boot-starter-web
// com.alibaba.cloud.ai:spring-ai-alibaba-starter-dashscope

@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("你是一位专业的A股分析师...")
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
```

配置 `application.yml`：

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-max
```

---

## 5. 一句话总结

> **国内 Java 团队做 AI 应用，无脑选 Spring AI Alibaba**——它就是 Spring AI 的国内增强版，能力更多、模型更顺、文档更中文化。

---

## 关联笔记

- [AI 应用落地技术栈全景（总览）](AI应用落地技术栈全景.md)
- [AI 落地决策框架与组合策略](AI落地决策框架与组合策略.md)
- [Skill 编排本质与多 Agent 协作机制](agent/Skill编排与多Agent协作机制.md)
