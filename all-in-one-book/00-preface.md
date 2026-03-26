# 第 0 章：前言与导读

## 🎯 本章学习目标

- 了解 Symfony AI 是什么、为什么它对 PHP 开发者至关重要
- 掌握项目整体架构和组件间的依赖关系
- 完成开发环境的搭建
- 运行你的第一个 "Hello AI" 程序
- 明确后续章节的学习路径

---

## 1. 什么是 Symfony AI？

**Symfony AI** 是由 Symfony 社区维护的一套 PHP AI 集成工具包。它的目标很简单：
**让 PHP 开发者能够像使用其他 Symfony 组件一样，轻松地在应用中集成 AI 能力。**

在 AI 迅速改变软件开发方式的今天，Python 和 JavaScript 生态拥有大量成熟的 AI 开发库，
而 PHP 开发者长期缺少一套统一、高质量的 AI 集成方案。Symfony AI 正是为填补这一空白而诞生的。

### Symfony AI 提供了什么？

| 能力 | 说明 |
|------|------|
| **统一的 AI 平台接口** | 通过 Platform 组件，使用相同的 API 调用 33+ AI 平台（OpenAI、Anthropic、Gemini 等） |
| **智能代理框架** | 通过 Agent 组件构建能使用工具、协作完成任务的 AI 代理 |
| **向量存储与 RAG** | 通过 Store 组件，支持 25+ 向量数据库后端，轻松构建检索增强生成应用 |
| **多轮对话管理** | 通过 Chat 组件，管理对话历史，支持 10+ 存储后端 |
| **Symfony 深度集成** | 通过 AI Bundle 和 MCP Bundle，将 AI 能力无缝融入 Symfony 框架 |
| **AI 开发助手** | 通过 Mate 组件，为 AI 助手提供与 PHP 应用交互的 MCP 开发服务器 |

💡 **提示**：如果你曾经使用过 LangChain（Python）或 Vercel AI SDK（JavaScript），
那么 Symfony AI 就是 PHP 世界的同类解决方案——而且它完全遵循 Symfony 的设计哲学和编码规范。

---

## 2. PHP 开发者的 AI 新时代

### 为什么 PHP 开发者需要 AI？

AI 不再只是实验室里的技术，它正在融入每一个业务场景：

- 🤖 **智能客服**：自动回答用户问题，减少人工成本
- 📝 **内容生成**：自动生成营销文案、产品描述、技术文档
- 🔍 **智能搜索**：基于语义理解的知识库检索（RAG）
- 📊 **数据分析**：从非结构化文本中提取结构化信息
- 🔄 **工作流自动化**：让 AI 代理协调多个系统完成复杂任务

### 为什么选择 Symfony AI？

与直接调用各平台 REST API 相比，Symfony AI 提供了显著的优势：

```text
直接调用 REST API：
  ❌ 每个平台的 API 格式不同，需要分别学习
  ❌ 缺少类型安全，容易出错
  ❌ 切换平台需要大量代码修改
  ❌ 缺少高级特性（工具调用、RAG、流式输出等）的抽象

使用 Symfony AI：
  ✅ 统一 API，一套代码适配所有平台
  ✅ 完整的类型安全和 IDE 支持
  ✅ 切换平台只需更改一行配置
  ✅ 开箱即用的工具调用、RAG、流式输出、多代理编排
  ✅ 与 Symfony 框架深度集成
```

---

## 3. 项目架构总览

### 3.1 组件关系图

下面的架构图展示了 Symfony AI 各组件之间的依赖关系：

```mermaid
graph TB
    subgraph "应用层"
        APP["你的 Symfony 应用"]
    end

    subgraph "集成层"
        AIB["AI Bundle<br/>Symfony 框架集成"]
        MCPB["MCP Bundle<br/>模型上下文协议"]
    end

    subgraph "功能层"
        AGENT["Agent 组件<br/>智能代理框架"]
        CHAT["Chat 组件<br/>多轮对话管理"]
        STORE["Store 组件<br/>向量存储与 RAG"]
        MATE["Mate 组件<br/>AI 开发助手"]
    end

    subgraph "基础层"
        PLATFORM["Platform 组件<br/>AI 平台统一接口"]
    end

    subgraph "外部服务"
        OPENAI["OpenAI"]
        ANTHROPIC["Anthropic"]
        GEMINI["Google Gemini"]
        OTHERS["33+ 其他平台..."]
        VDB["向量数据库<br/>Pinecone / Qdrant / ..."]
    end

    APP --> AIB
    APP --> MCPB
    AIB --> AGENT
    AIB --> STORE
    AIB --> PLATFORM
    MCPB --> MATE
    AGENT --> PLATFORM
    AGENT --> STORE
    CHAT --> AGENT
    MATE --> PLATFORM
    PLATFORM --> OPENAI
    PLATFORM --> ANTHROPIC
    PLATFORM --> GEMINI
    PLATFORM --> OTHERS
    STORE --> VDB

    style PLATFORM fill:#4CAF50,color:#fff
    style AGENT fill:#2196F3,color:#fff
    style STORE fill:#FF9800,color:#fff
    style CHAT fill:#9C27B0,color:#fff
    style AIB fill:#F44336,color:#fff
    style MCPB fill:#607D8B,color:#fff
    style MATE fill:#795548,color:#fff
```

### 3.2 组件依赖关系

```mermaid
graph LR
    PLATFORM["Platform"]
    AGENT["Agent"]
    STORE["Store"]
    CHAT["Chat"]
    AIB["AI Bundle"]
    MCPB["MCP Bundle"]
    MATE["Mate"]

    AGENT -->|依赖| PLATFORM
    CHAT -->|依赖| AGENT
    AGENT -.->|可选依赖| STORE
    AIB -->|集成| PLATFORM
    AIB -->|集成| AGENT
    AIB -->|集成| STORE
    MCPB -->|集成| MATE
    MATE -->|依赖| PLATFORM

    style PLATFORM fill:#4CAF50,color:#fff
    style AGENT fill:#2196F3,color:#fff
    style STORE fill:#FF9800,color:#fff
    style CHAT fill:#9C27B0,color:#fff
```

### 3.3 组件协作方式

各组件通过明确的分层职责协同工作：

| 层级 | 组件 | 职责 |
|------|------|------|
| **基础层** | Platform | 提供与 AI 平台通信的统一接口，处理请求/响应转换 |
| **功能层** | Agent | 在 Platform 之上构建智能代理，支持工具调用和任务编排 |
| **功能层** | Store | 提供向量存储抽象，为 Agent 提供 RAG 检索能力 |
| **功能层** | Chat | 在 Agent 之上管理多轮对话的上下文和历史 |
| **功能层** | Mate | 提供 MCP 开发服务器，让 AI 助手与 PHP 应用交互 |
| **集成层** | AI Bundle | 将 Platform、Agent、Store 集成到 Symfony 框架中 |
| **集成层** | MCP Bundle | 将 MCP 协议集成到 Symfony 框架中 |

📌 **要点**：**Platform 是所有功能的基石**。即使你只想使用 Agent 或 Chat，Platform 也是必须安装的核心依赖。

---

## 4. 各章内容预览

| 章节 | 你将学到 |
|------|----------|
| **第 1 章：快速入门** | 项目安装、环境配置，用最少的代码运行一个完整的 AI 应用 |
| **第 2 章：Platform** | 深入理解 Platform 组件——多平台接入、模型调用、流式响应、多模态输入、嵌入向量 |
| **第 3 章：Agent** | 构建智能代理——工具调用（Function Calling）、多代理编排、结构化输出、记忆与上下文 |
| **第 4 章：Store** | 向量数据库集成——文档索引、语义检索、RAG 应用构建 |
| **第 5 章：Chat** | 多轮对话管理——对话历史持久化、上下文窗口控制、多后端存储 |
| **第 6 章：AI Bundle** | Symfony 框架集成——服务定义、YAML 配置、依赖注入、事件系统 |
| **第 7 章：MCP Bundle** | MCP 协议集成——工具暴露、资源提供、提示模板 |
| **第 8 章：Mate** | AI 开发助手——MCP 开发服务器搭建、与 Cursor/Claude 等 AI 工具交互 |
| **第 9 章：基础实战** | 智能客服、内容生成、文本摘要、翻译助手等常见场景 |
| **第 10 章：进阶实战** | RAG 知识库、多代理协作、复杂工作流编排 |
| **第 11 章：高级实战** | 自定义平台桥接、性能优化、生产环境部署与监控 |
| **第 12 章：架构与最佳实践** | 设计模式、错误处理策略、安全防护、可观测性 |
| **第 13 章：附录** | API 速查表、版本升级指南、常见问题解答 |

---

## 5. 环境准备

### 5.1 系统要求

```bash
# 检查 PHP 版本（需要 8.4+）
php -v

# 检查 Composer 版本
composer --version
```

⚠️ **注意**：Symfony AI 要求 **PHP 8.4 或更高版本**。如果你的 PHP 版本较低，请先升级。

### 5.2 安装 Symfony AI

根据你的使用场景，选择合适的安装方式：

**场景一：在现有 Symfony 项目中使用**

```bash
# 安装 AI Bundle（推荐，包含 Platform + Agent + Store 的框架集成）
composer require symfony/ai-bundle

# 如果只需要基础的 AI 平台调用
composer require symfony/ai-platform
```

**场景二：在非 Symfony 项目中使用**

```bash
# 安装核心 Platform 组件
composer require symfony/ai-platform

# 按需安装其他组件
composer require symfony/ai-agent   # 智能代理
composer require symfony/ai-store   # 向量存储
composer require symfony/ai-chat    # 多轮对话
```

**场景三：安装特定平台的桥接（Bridge）**

```bash
# OpenAI 桥接
composer require symfony/ai-open-ai-platform

# Anthropic 桥接
composer require symfony/ai-anthropic-platform

# Google Gemini 桥接
composer require symfony/ai-gemini-platform
```

### 5.3 获取 API 密钥

要使用 AI 服务，你需要至少一个平台的 API 密钥：

| 平台 | 获取地址 | 说明 |
|------|----------|------|
| **OpenAI** | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) | GPT-4o、DALL-E、Whisper 等 |
| **Anthropic** | [console.anthropic.com](https://console.anthropic.com) | Claude 系列模型 |
| **Google AI** | [aistudio.google.com](https://aistudio.google.com) | Gemini 系列模型 |

💡 **提示**：推荐从 OpenAI 或 Anthropic 开始，因为本手册的大部分示例使用这两个平台。
大多数平台提供免费额度，足够学习和实验使用。

### 5.4 配置环境变量

将你的 API 密钥配置为环境变量：

```bash
# .env 文件（不要将此文件提交到版本控制！）
OPENAI_API_KEY=sk-your-openai-api-key
ANTHROPIC_API_KEY=sk-ant-your-anthropic-api-key
GOOGLE_API_KEY=your-google-api-key
```

⚠️ **安全提示**：永远不要将 API 密钥硬编码到源代码中或提交到 Git 仓库。
请使用 `.env` 文件或环境变量管理工具来管理密钥。

---

## 6. Hello AI：你的第一个程序

让我们用最少的代码，完成你与 AI 的第一次对话！

### 6.1 创建项目

```bash
mkdir hello-ai && cd hello-ai
composer init --no-interaction
composer require symfony/ai-platform
```

### 6.2 编写代码

创建 `hello.php` 文件：

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建 Platform 实例
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

// 2. 构建消息
$messages = new MessageBag(
    Message::forSystem('你是一个友好的 PHP 助手。'),
    Message::ofUser('你好！请用一句话介绍 Symfony AI。'),
);

// 3. 调用模型
$response = $platform->invoke('gpt-4o-mini', $messages);

// 4. 输出结果
echo $response->asText();
```

### 6.3 运行程序

```bash
OPENAI_API_KEY=sk-your-key php hello.php
```

你会看到类似这样的输出：

```text
Symfony AI 是一套为 PHP 开发者打造的 AI 集成工具包，让你可以通过统一的接口轻松调用
OpenAI、Anthropic 等 33+ AI 平台的能力。
```

🎉 **恭喜！** 你已经成功完成了第一个 Symfony AI 程序！

### 6.4 代码解析

让我们逐行理解刚才的代码：

```php
┌─────────────────────────────────────────────────────────────┐
│ PlatformFactory::create()                                   │
│  ↓ 创建一个 Platform 实例，封装了与 OpenAI 的通信细节         │
├─────────────────────────────────────────────────────────────┤
│ MessageBag + Message                                        │
│  ↓ 构建对话消息，包含系统提示和用户输入                       │
├─────────────────────────────────────────────────────────────┤
│ $platform->invoke('gpt-4o-mini', $messages)                 │
│  ↓ 调用指定模型，返回 DeferredResult（延迟求值）             │
├─────────────────────────────────────────────────────────────┤
│ $response->asText()                                         │
│  ↓ 此时才真正发送 HTTP 请求，获取文本结果                     │
└─────────────────────────────────────────────────────────────┘
```

📌 **关键概念**：`invoke()` 返回的是 `DeferredResult` 对象（延迟结果）。
实际的 HTTP 请求在你调用 `asText()`、`asStream()` 等方法时才会发送。
这种延迟求值设计让你可以灵活地决定如何消费结果。

### 6.5 换一个平台试试

Symfony AI 的强大之处在于：**切换 AI 平台只需更改两行代码**：

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

// 使用 Anthropic（Claude）
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一个友好的 PHP 助手。'),
    Message::ofUser('你好！请用一句话介绍 Symfony AI。'),
);

// 注意：模型名称对应 Anthropic 的模型
$response = $platform->invoke('claude-sonnet-4-20250514', $messages);

echo $response->asText();
```

💡 **提示**：除了 `PlatformFactory` 的 `use` 语句和模型名称，其余代码完全不需要修改！
这就是统一接口的威力。

---

## 7. 下一步

现在你已经了解了 Symfony AI 的全貌，并成功运行了第一个 AI 程序。

在 **[第 1 章：快速入门](01-quick-start.md)** 中，我们将：

- 搭建一个完整的 Symfony AI 开发环境
- 探索更多 Platform 的核心功能（流式输出、多模态输入）
- 初步了解 Agent 组件的工具调用能力
- 完成一个更具实际意义的应用示例

---

> 📖 [返回目录](README.md) | [下一章：快速入门 →](01-quick-start.md)
