# 0 

## 

- Symfony AI PHP 
- 
- 
- "Hello AI" 
- 

---

## 1. Symfony AI

**Symfony AI** Symfony PHP AI 
** PHP Symfony AI **

 AI Python JavaScript AI 
 PHP AI Symfony AI 

### Symfony AI 

| | |
|------|------|
| ** AI ** | Platform API 33+ AI OpenAIAnthropicGemini |
| **** | Agent AI |
| ** RAG** | Store 25+ |
| **** | Chat 10+ |
| **Symfony ** | AI Bundle MCP Bundle AI Symfony |
| **AI ** | Mate AI PHP MCP |

 **** LangChainPython Vercel AI SDKJavaScript
 Symfony AI PHP —— Symfony 

---

## 2. PHP AI 

### PHP AI

AI 

- ****
- ****
- ****RAG
- ****
- **** AI 

### Symfony AI

 REST API Symfony AI 

```
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

## 3. 

### 3.1 

 Symfony AI 

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

### 3.2 

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

### 3.3 



| | | |
|------|------|------|
| **** | Platform | AI / |
| **** | Agent | Platform |
| **** | Store | Agent RAG |
| **** | Chat | Agent |
| **** | Mate | MCP AI PHP |
| **** | AI Bundle | PlatformAgentStore Symfony |
| **** | MCP Bundle | MCP Symfony |

 ******Platform ** Agent ChatPlatform 

---

## 4. 

| | |
|------|----------|
| ** 1 ** | AI |
| ** 2 Platform** | Platform —— |
| ** 3 Agent** | ——Function Calling |
| ** 4 Store** | ——RAG |
| ** 5 Chat** | —— |
| ** 6 AI Bundle** | Symfony ——YAML |
| ** 7 MCP Bundle** | MCP —— |
| ** 8 Mate** | AI ——MCP Cursor/Claude AI |
| ** 9 ** | |
| ** 10 ** | RAG |
| ** 11 ** | |
| ** 12 ** | |
| ** 13 ** | API |

---

## 5. 

### 5.1 

```bash
# 检查 PHP 版本（需要 8.4+）
php -v

# 检查 Composer 版本
composer --version
```

 ****Symfony AI **PHP 8.4 ** PHP 

### 5.2 Symfony AI



** Symfony **

```bash
# 安装 AI Bundle（推荐，包含 Platform + Agent + Store 的框架集成）
composer require symfony/ai-bundle

# 如果只需要基础的 AI 平台调用
composer require symfony/ai-platform
```

** Symfony **

```bash
# 安装核心 Platform 组件
composer require symfony/ai-platform

# 按需安装其他组件
composer require symfony/ai-agent   # 智能代理
composer require symfony/ai-store   # 向量存储
composer require symfony/ai-chat    # 多轮对话
```

**Bridge**

```bash
# OpenAI 桥接
composer require symfony/ai-open-ai-platform

# Anthropic 桥接
composer require symfony/ai-anthropic-platform

# Google Gemini 桥接
composer require symfony/ai-gemini-platform
```

### 5.3 API 

 AI API 

| | | |
|------|----------|------|
| **OpenAI** | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) | GPT-4oDALL-EWhisper |
| **Anthropic** | [console.anthropic.com](https://console.anthropic.com) | Claude |
| **Google AI** | [aistudio.google.com](https://aistudio.google.com) | Gemini |

 **** OpenAI Anthropic 


### 5.4 

 API 

```bash
# .env 文件（不要将此文件提交到版本控制！）
OPENAI_API_KEY=sk-your-openai-api-key
ANTHROPIC_API_KEY=sk-ant-your-anthropic-api-key
GOOGLE_API_KEY=your-google-api-key
```

 **** API Git 
 `.env` 

---

## 6. Hello AI

 AI 

### 6.1 

```bash
mkdir hello-ai && cd hello-ai
composer init --no-interaction
composer require symfony/ai-platform
```

### 6.2 

 `hello.php` 

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

### 6.3 

```bash
OPENAI_API_KEY=sk-your-key php hello.php
```



```
Symfony AI 是一套为 PHP 开发者打造的 AI 集成工具包，让你可以通过统一的接口轻松调用
OpenAI、Anthropic 等 33+ AI 平台的能力。
```

 **** Symfony AI 

### 6.4 



```
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

 ****`invoke()` `DeferredResult` 
 HTTP `asText()``asStream()` 


### 6.5 

Symfony AI ** AI **

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

 **** `PlatformFactory` `use` 


---

## 7. 

 Symfony AI AI 

 **[ 1 ](01-quick-start.md)** 

- Symfony AI 
- Platform 
- Agent 
- 

---

> [](README.md) | [ →](01-quick-start.md)
