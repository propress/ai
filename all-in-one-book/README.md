# Symfony AI 完全学习手册

> 一本从零开始掌握 Symfony AI 的中文实战手册，涵盖所有组件、集成方式与真实业务场景。

## 关于本书

**Symfony AI** 是一套为 PHP 开发者打造的 AI 能力集成工具包，提供了对 33+ AI 平台的统一接口、
智能代理框架、向量存储、多轮对话管理等核心能力。

本手册将从基础概念出发，逐步深入每个组件，最后通过实战场景将所有知识融会贯通。
无论你是 AI 开发新手还是经验丰富的 PHP 开发者，都能在这里找到适合自己的学习路径。

## 前置要求

| 要求 | 说明 |
|------|------|
| **PHP** | 8.4 或更高版本 |
| **Composer** | 最新版本 |
| **Symfony 知识** | 了解基本的 Symfony 项目结构和依赖注入概念 |
| **AI API 密钥** | 至少拥有一个 AI 平台的 API Key（如 OpenAI、Anthropic 等） |

## 目录

### 基础篇（第 0-8 章）

掌握 Symfony AI 各核心组件的概念与用法。

| 章节 | 标题 | 内容简介 |
|------|------|----------|
| [第 0 章](00-preface.md) | 前言与导读 | 项目概览、架构总览、环境准备、Hello AI |
| [第 1 章](01-quick-start.md) | 快速入门 | 安装配置、第一个完整示例、核心概念速览 |
| [第 2 章](02-platform.md) | Platform 组件：AI 平台统一接口 | 多平台接入、模型调用、流式响应、多模态 |
| [第 3 章](03-agent.md) | Agent 组件：智能代理框架 | 工具调用、多代理编排、结构化输出、记忆系统 |
| [第 4 章](04-store.md) | Store 组件：向量存储与 RAG | 向量数据库、文档索引、检索增强生成 |
| [第 5 章](05-chat.md) | Chat 组件：多轮对话管理 | 对话历史、存储后端、上下文管理 |
| [第 6 章](06-ai-bundle.md) | AI Bundle：Symfony 框架集成 | 服务配置、依赖注入、框架深度集成 |
| [第 7 章](07-mcp-bundle.md) | MCP Bundle：模型上下文协议 | MCP 协议、工具暴露、与 AI 助手交互 |
| [第 8 章](08-mate.md) | Mate 组件：AI 开发助手 | MCP 开发服务器、PHP 应用调试与交互 |

### 实战篇（第 9-24 章）

通过 16 个真实业务场景，将组件知识融会贯通。

#### 入门实战（第 9-14 章）

| 章节 | 标题 | 涉及组件 | 难度 |
|------|------|----------|------|
| [第 9 章](09-scenario-chatbot.md) | 基础问答聊天机器人 | Platform | ⭐ |
| [第 10 章](10-scenario-multi-turn-chat.md) | 多轮对话与会话持久化 | Platform + Chat | ⭐⭐ |
| [第 11 章](11-scenario-structured-extraction.md) | 结构化数据提取 | Platform | ⭐⭐ |
| [第 12 章](12-scenario-multimodal.md) | 多模态内容理解 | Platform | ⭐⭐ |
| [第 13 章](13-scenario-translation.md) | 多语言翻译助手 | Platform | ⭐ |
| [第 14 章](14-scenario-streaming.md) | 实时流式对话 | Platform | ⭐⭐ |

#### 进阶实战（第 15-19 章）

| 章节 | 标题 | 涉及组件 | 难度 |
|------|------|----------|------|
| [第 15 章](15-scenario-tool-agent.md) | 工具增强型 AI 助手 | Agent + Platform | ⭐⭐⭐ |
| [第 16 章](16-scenario-rag.md) | RAG 知识库问答 | Store + Agent + Platform | ⭐⭐⭐ |
| [第 17 章](17-scenario-multi-agent-routing.md) | 多智能体客服路由 | Agent (MultiAgent) | ⭐⭐⭐ |
| [第 18 章](18-scenario-web-search.md) | 联网搜索研究助手 | Agent + Tavily | ⭐⭐⭐ |
| [第 19 章](19-scenario-memory-agent.md) | 带记忆的个性化助手 | Agent + Store | ⭐⭐⭐ |

#### 高级实战（第 20-24 章）

| 章节 | 标题 | 涉及组件 | 难度 |
|------|------|----------|------|
| [第 20 章](20-scenario-high-availability.md) | 高可用 AI 服务架构 | Platform (Failover/Cache) | ⭐⭐⭐⭐ |
| [第 21 章](21-scenario-local-model.md) | 本地模型私有化部署 | Platform (Ollama) | ⭐⭐⭐ |
| [第 22 章](22-scenario-content-moderation.md) | 内容审核流水线 | Agent + Platform | ⭐⭐⭐⭐ |
| [第 23 章](23-scenario-crew-multi-agent.md) | CrewAI 风格多智能体团队 | Agent (MultiAgent) | ⭐⭐⭐⭐⭐ |
| [第 24 章](24-scenario-enterprise-kb.md) | 端到端企业知识库系统 | All Components | ⭐⭐⭐⭐⭐ |

### 参考篇（第 25-26 章）

架构指南与速查参考。

| 章节 | 标题 | 内容简介 |
|------|------|----------|
| [第 25 章](25-architecture.md) | 架构与最佳实践 | 设计模式、错误处理、安全性、可观测性 |
| [第 26 章](26-appendix.md) | 附录：API 速查与升级指南 | 常用 API 参考、版本升级、常见问题 |

## 推荐阅读路径

### 快速上手（1-2 小时）

适合想要快速体验 Symfony AI 的开发者：

```text
第 0 章（前言） → 第 1 章（快速入门） → 第 9 章（基础聊天机器人）
```

### 系统学习（1-2 周）

适合希望全面掌握 Symfony AI 的开发者：

```text
第 0-8 章（基础篇） → 第 9-24 章（实战篇） → 第 25 章（架构）
```

### Symfony 集成专项

适合已有 Symfony 项目、希望集成 AI 能力的团队：

```text
第 0-1 章 → 第 6 章（AI Bundle）→ 第 2-5 章（核心组件）
→ 第 7-8 章（MCP & Mate）→ 第 15-19 章（进阶实战）
```

### RAG 应用专项

适合需要构建知识库、文档检索等 RAG 应用的开发者：

```text
第 0-1 章 → 第 2 章（Platform）→ 第 4 章（Store）
→ 第 3 章（Agent）→ 第 16 章（RAG 知识库问答）→ 第 24 章（企业知识库）
```

## 官方资源

| 资源 | 链接 |
|------|------|
| **GitHub 仓库** | [github.com/symfony/ai](https://github.com/symfony/ai) |
| **官方文档** | [ai.symfony.com](https://ai.symfony.com) |
| **Symfony 官网** | [symfony.com](https://symfony.com) |
| **Packagist** | [packagist.org/packages/symfony/ai-platform](https://packagist.org/packages/symfony/ai-platform) |

## 使用提示

- 每章都包含完整的代码示例，建议边学边实践
- 代码示例均基于 Symfony AI 最新版本
- 遇到问题可查阅[第 26 章](26-appendix.md)的常见问题部分
- 欢迎在 GitHub 上提 Issue 或 PR 帮助改进本手册

---

> 让我们开始 Symfony AI 的学习之旅吧！请从 [第 0 章：前言与导读](00-preface.md) 开始。
