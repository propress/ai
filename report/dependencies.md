# 模块依赖关系总览

## 1. 依赖矩阵

下表展示各模块之间的依赖关系（✅ = 依赖，❌ = 不依赖，行=依赖方，列=被依赖方）：

| 依赖方 \ 被依赖方 | platform | agent | store | chat | ai-bundle | mcp-bundle | mate |
|-----------------|:--------:|:-----:|:-----:|:----:|:---------:|:----------:|:----:|
| **platform**    | —        | ❌    | ❌    | ❌  | ❌        | ❌         | ❌  |
| **agent**       | ✅ (require) | — | ❌  | ❌  | ❌        | ❌         | ❌  |
| **store**       | ✅ (require) | ❌ | —   | ❌  | ❌        | ❌         | ❌  |
| **chat**        | ✅ (require) | ✅ (require) | ❌ | — | ❌   | ❌         | ❌  |
| **ai-bundle**   | ✅ (require) | ✅ (require-dev) | ✅ (require-dev) | ✅ (require-dev) | — | ❌ | ❌ |
| **mcp-bundle**  | ❌       | ❌    | ❌    | ❌  | ❌        | —          | ❌  |
| **mate**        | ❌       | ❌    | ❌    | ❌  | ❌        | ❌         | —   |

> **说明**：
> - `require`：运行时必须依赖
> - `require-dev`：开发/测试时依赖（ai-bundle 在 require-dev 中引用各组件以支持测试各桥接器集成）
> - mcp-bundle 核心依赖 `mcp/sdk`（官方 PHP MCP SDK），不直接依赖 symfony/ai 内部模块
> - mate 依赖 `mcp/sdk`，不依赖 symfony/ai 内部模块（独立 MCP 服务器工具）

---

## 2. 架构层次图（ASCII）

```
╔══════════════════════════════════════════════════════════════════════╗
║                         外部 AI 工具层                                ║
║                                                                      ║
║     Claude Desktop    Cursor IDE    Continue.dev    自定义 MCP 客户端 ║
╚═══════════════╤══════════════╤═══════════════════════════════════════╝
                │ MCP 协议      │ HTTP / stdio
                ▼              ▼
╔══════════════════════════════════════════════════════════════════════╗
║                         应用集成层                                    ║
║                                                                      ║
║          ai-bundle              mcp-bundle           mate            ║
║       (DI 集成容器)           (MCP 协议适配)      (开发助手工具)       ║
║                                                                      ║
║  symfony/ai-platform (require)  mcp/sdk (require)  mcp/sdk (require) ║
╚══════════════╤════════════════════════════════════════╤══════════════╝
               │                                        │
               │ 依赖（require/require-dev）              │ 独立运行
               ▼                                        ▼
╔══════════════════════════════════════════════════════╗
║                      业务逻辑层                        ║
║                                                      ║
║        agent              store             chat     ║
║  (AI Agent 框架)     (向量存储抽象)      (对话管理)   ║
║                                                      ║
║  symfony/ai-platform  symfony/ai-platform  agent +   ║
║      (require)            (require)       platform   ║
╚══════════════╤══════════════╤════════════════════════╝
               │              │
               └──────┬───────┘
                      │ 全部依赖
                      ▼
╔══════════════════════════════════════════════════════╗
║                      基础设施层                        ║
║                                                      ║
║                      platform                        ║
║                  (AI 平台统一接口)                    ║
║                                                      ║
║  无内部依赖 — 仅依赖 symfony/* 标准组件               ║
║  symfony/serializer  symfony/event-dispatcher        ║
║  symfony/property-info  symfony/type-info  等        ║
╚══════════════════════════════════════════════════════╝
               │
               │ Bridge 桥接器（独立 Composer 包）
               ▼
╔══════════════════════════════════════════════════════════════════════╗
║                       外部 AI 服务层（Bridges）                       ║
║                                                                      ║
║  OpenAI   Anthropic  Azure  Gemini  VertexAI  Ollama  Mistral  ...  ║
║  (各自独立 Composer 包，按需安装)                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

**层次说明**：
- **基础设施层**：`platform` 是整个生态的基石，定义与 AI 服务通信的标准接口
- **业务逻辑层**：`agent`、`store`、`chat` 构建在 `platform` 之上，提供核心业务能力
- **应用集成层**：`ai-bundle`、`mcp-bundle`、`mate` 将下层能力集成到 Symfony 应用
- **外部 AI 服务层**：各桥接器独立安装，实现具体 AI 服务的对接

---

## 3. 各模块依赖详情

### 3.1 platform（symfony/ai-platform）

**包名**：`symfony/ai-platform`  
**当前版本**：`^0.6`  
**定位**：AI 平台通信的统一抽象层

**运行时依赖（require）**：

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `php` | >=8.2 | PHP 版本要求 |
| `ext-fileinfo` | * | 多模态文件类型检测 |
| `oskarstark/enum-helper` | ^1.5 | 枚举辅助工具 |
| `phpdocumentor/reflection-docblock` | ^5.4\|^6.0 | PHPDoc 解析（工具参数提取） |
| `phpstan/phpdoc-parser` | ^2.1 | PHPDoc 解析器 |
| `psr/log` | ^3.0 | PSR 日志接口 |
| `symfony/clock` | ^7.3\|^8.0 | 时间抽象 |
| `symfony/event-dispatcher` | ^7.3\|^8.0 | 事件系统（平台请求/响应事件） |
| `symfony/property-access` | ^7.3\|^8.0 | 属性访问器 |
| `symfony/property-info` | ^7.3\|^8.0 | 属性类型信息（JSON Schema 生成） |
| `symfony/serializer` | ^7.3\|^8.0 | 消息序列化/反序列化 |
| `symfony/type-info` | ^7.3\|^8.0 | 类型系统 |
| `symfony/uid` | ^7.3\|^8.0 | UUID 生成（消息 ID） |

**开发依赖（require-dev）**：
- `symfony/http-client` — 实际 HTTP 请求（生产中由桥接器提供）
- `symfony/cache` — 缓存测试
- `symfony/console` — CLI 工具测试
- `symfony/expression-language` — 条件表达式测试
- `symfony/validator` — 输入验证测试
- PHPStan、PHPUnit 等质量工具

**内部依赖**：**无** — platform 是整个生态的根节点

---

### 3.2 agent（symfony/ai-agent）

**包名**：`symfony/ai-agent`  
**当前版本**：`^0.6`  
**定位**：构建 AI Agent 的核心框架，提供工具调用、消息处理、推理循环

**运行时依赖（require）**：

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `php` | >=8.2 | PHP 版本要求 |
| `ext-fileinfo` | * | 文件处理 |
| `phpdocumentor/reflection-docblock` | ^5.4\|^6.0 | 工具注解解析 |
| `phpstan/phpdoc-parser` | ^2.1 | PHPDoc 解析 |
| `psr/log` | ^3.0 | 日志接口 |
| **`symfony/ai-platform`** | **^0.6** | **AI 平台通信（内部依赖）** |
| `symfony/clock` | ^7.3\|^8.0 | 时间抽象 |
| `symfony/http-client` | ^7.3\|^8.0 | MCP 客户端 HTTP 请求 |
| `symfony/polyfill-php85` | ^1.33 | PHP 8.5 特性向下兼容 |
| `symfony/property-access` | ^7.3\|^8.0 | 属性访问 |
| `symfony/property-info` | ^7.3\|^8.0 | 类型信息（Toolbox 参数解析） |
| `symfony/serializer` | ^7.3\|^8.0 | 消息序列化 |
| `symfony/type-info` | ^7.3\|^8.0 | 类型系统 |

**开发依赖（require-dev）**：
- `symfony/ai-store ^0.6` — 测试 Store 集成（RAG 工具）
- `symfony/event-dispatcher` — 事件测试
- `symfony/translation` — 国际化测试
- PHPStan、PHPUnit

**内部依赖**：`platform`（唯一直接内部依赖）

---

### 3.3 store（symfony/ai-store）

**包名**：`symfony/ai-store`  
**当前版本**：`^0.6`  
**定位**：向量数据库抽象层，支持文档存储、向量检索、RAG 应用

**运行时依赖（require）**：

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `php` | >=8.2 | PHP 版本要求 |
| `ext-fileinfo` | * | 文件类型检测 |
| `psr/log` | ^3.0 | 日志接口 |
| **`symfony/ai-platform`** | **^0.6** | **向量生成（内部依赖）** |
| `symfony/clock` | ^7.3\|^8.0 | 时间抽象 |
| `symfony/http-client` | ^7.3\|^8.0 | 向量数据库 HTTP 请求 |
| `symfony/polyfill-php83` | ^1.32 | PHP 8.3 特性向下兼容 |
| `symfony/service-contracts` | ^2.5\|^3 | 服务契约 |
| `symfony/event-dispatcher-contracts` | ^3.0 | 事件契约 |
| `symfony/uid` | ^7.3\|^8.0 | 文档 UUID |

**开发依赖（require-dev）**：
- `symfony/console` — CLI 命令（数据导入工具）
- `symfony/dom-crawler` — HTML 解析（文档处理）
- `symfony/dependency-injection` — DI 容器集成测试
- `symfony/filesystem` — 文件系统工具
- `symfony/string` — 字符串处理
- `symfony/json-path` — JSON 路径查询
- PHPStan、PHPUnit

**内部依赖**：`platform`（用于生成文档嵌入向量）

---

### 3.4 chat（symfony/ai-chat）

**包名**：`symfony/ai-chat`  
**当前版本**：`^0.6`  
**定位**：对话管理组件，提供消息历史存储、多轮对话支持

**运行时依赖（require）**：

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `php` | >=8.2 | PHP 版本要求 |
| **`symfony/ai-agent`** | **^0.6** | **Agent 核心（内部依赖）** |
| **`symfony/ai-platform`** | **^0.6** | **平台接口（内部依赖）** |
| `symfony/service-contracts` | ^2.5\|^3 | 服务契约 |

**开发依赖（require-dev）**：
- `symfony/clock` — 时间戳测试
- `symfony/console` — CLI 测试
- `symfony/dependency-injection` — DI 测试
- `symfony/serializer` — 消息序列化测试
- PHPStan、PHPUnit

**内部依赖**：`platform` + `agent`（直接依赖两个核心模块）

---

### 3.5 ai-bundle（symfony/ai-bundle）

**包名**：`symfony/ai-bundle`  
**当前版本**：`^0.6`  
**定位**：将所有 Symfony AI 组件集成到 Symfony 框架的一体化 Bundle

**运行时依赖（require）**：

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `php` | >=8.2 | PHP 版本要求 |
| **`symfony/ai-platform`** | **^0.6** | **平台集成（内部依赖，运行时）** |
| `symfony/clock` | ^7.3\|^8.0 | 时间服务 |
| `symfony/config` | ^7.3\|^8.0 | Bundle 配置定义 |
| `symfony/console` | ^7.3\|^8.0 | Console 命令 |
| `symfony/dependency-injection` | ^7.3\|^8.0 | DI 容器集成 |
| `symfony/framework-bundle` | ^7.3\|^8.0 | Symfony 框架基础 |
| `symfony/service-contracts` | ^2.5\|^3 | 服务契约 |
| `symfony/string` | ^7.3\|^8.0 | 字符串工具 |

**主要开发依赖（require-dev，部分）**：
- `symfony/ai-agent ^0.6` — Agent 组件集成测试
- `symfony/ai-store ^0.6` — Store 组件集成测试
- `symfony/ai-chat ^0.6` — Chat 组件集成测试
- 所有平台桥接器（openai、anthropic、azure、gemini 等 50+ 个）
- 所有 Store 桥接器（qdrant、pinecone、mongodb 等 20+ 个）
- `symfony/expression-language` — 表达式语言支持
- `symfony/security-core` — 权限控制
- `symfony/translation` — 国际化
- `symfony/validator` — 输入验证
- PHPStan、PHPUnit

**内部依赖**：
- 运行时：`platform`
- 开发时：`agent`、`store`、`chat`（测试各组件集成）

---

### 3.6 mcp-bundle（symfony/mcp-bundle）

**包名**：`symfony/mcp-bundle`  
**当前版本**：基于 `mcp/sdk ^0.4`  
**定位**：将 MCP 协议集成到 Symfony 应用，支持服务端和客户端

**运行时依赖（require）**：

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| **`mcp/sdk`** | **^0.4** | **MCP 官方 PHP SDK（核心依赖）** |
| `php-http/discovery` | ^1.20 | PSR HTTP 工厂自动发现 |
| `symfony/config` | ^7.3\|^8.0 | Bundle 配置 |
| `symfony/console` | ^7.3\|^8.0 | stdio 模式 Console 命令 |
| `symfony/dependency-injection` | ^7.3\|^8.0 | DI 容器 |
| `symfony/framework-bundle` | ^7.3\|^8.0 | Symfony 框架 |
| `symfony/http-foundation` | ^7.3\|^8.0 | HTTP 请求/响应 |
| `symfony/http-kernel` | ^7.3\|^8.0 | HTTP 内核 |
| `symfony/psr-http-message-bridge` | ^7.3\|^8.0 | Symfony↔PSR-7 转换 |
| `symfony/routing` | ^7.3\|^8.0 | 路由注册 |
| `symfony/service-contracts` | ^2.5\|^3 | 服务契约 |

**开发依赖（require-dev）**：
- `symfony/monolog-bundle ^3.10\|^4.0` — 日志集成测试
- PHPStan、PHPUnit

**内部依赖**：**无 symfony/ai-* 直接依赖**（仅依赖 `mcp/sdk` 和 Symfony 框架组件）

---

### 3.7 mate（symfony/ai-mate）

**包名**：`symfony/ai-mate`  
**当前版本**：基于 `mcp/sdk ^0.4`  
**定位**：面向 Symfony 开发者的 AI 开发助手，作为 MCP 服务器暴露开发工具

**运行时依赖（require）**：

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `php` | >=8.2 | PHP 版本要求 |
| **`mcp/sdk`** | **^0.4** | **MCP 协议 SDK** |
| `psr/log` | ^2.0\|^3.0 | 日志接口 |
| `symfony/config` | ^5.4\|^6.4\|^7.3\|^8.0 | 配置系统 |
| `symfony/console` | ^5.4\|^6.4\|^7.3\|^8.0 | CLI 命令（bin/mate） |
| `symfony/dependency-injection` | ^5.4\|^6.4\|^7.3\|^8.0 | DI 容器 |
| `symfony/finder` | ^5.4\|^6.4\|^7.3\|^8.0 | 文件查找 |

**开发依赖（require-dev）**：
- `ext-simplexml` — XML 解析（PHPUnit 报告）
- `symfony/dotenv` — 环境变量测试
- PHPStan、PHPUnit

**内部依赖**：**无 symfony/ai-* 依赖**

**特殊说明**：
- 比 mcp-bundle 支持更宽的 Symfony 版本（从 5.4 开始）
- 提供 `bin/mate` 可执行文件，可作为独立 CLI 工具运行
- 通过 `extra.ai-mate.scan-dirs` 配置自动扫描能力目录

---

## 4. 核心依赖链分析

### 4.1 依赖树（有向无环图）

```
platform ◄─────── agent ◄─────── chat
    ▲                │
    │                │（require-dev）
    │                ▼
    └──────── store

ai-bundle ──► platform（require）
          ──► agent, store, chat（require-dev）

mcp-bundle ──► mcp/sdk（require，无内部依赖）
mate       ──► mcp/sdk（require，无内部依赖）
```

### 4.2 逐层分析

**platform — 生态基石**
- 无任何内部依赖
- 定义所有 AI 平台通信的接口（`ModelClientInterface`、`Message` 等）
- 所有需要调用 AI 的模块都依赖 platform
- 桥接器（Bridge）扩展 platform，实现具体平台对接

**agent — 核心业务层**
- 直接依赖 platform（运行时）
- 提供 `Agent` 类、`Toolbox`、工具调用循环
- store 在 require-dev 中出现（测试 RAG 工具），运行时不强制依赖

**store — 向量存储层**
- 直接依赖 platform（运行时）
- 使用 platform 的 Embeddings 接口生成文档向量
- 与 agent 是平行关系（store 不依赖 agent）

**chat — 对话管理层**
- 同时依赖 platform 和 agent（运行时）
- 是依赖最多的业务模块
- 提供消息历史管理，建立在 agent 之上

**ai-bundle — Symfony 集成层**
- 运行时仅需 platform（最小依赖）
- 通过条件配置支持 agent、store、chat（开发依赖中列出以测试集成）
- 实际使用时按需引入各组件

**mcp-bundle — MCP 协议适配层**
- 完全独立于 symfony/ai 内部模块
- 仅依赖官方 `mcp/sdk` 和 Symfony 框架组件
- 通过 `mcp/sdk` 的 `#[McpTool]` 等属性与 ai-bundle/agent 间接协作

**mate — 独立开发工具**
- 最宽松的 Symfony 版本兼容（^5.4 起）
- 不依赖任何 symfony/ai 内部模块
- 作为独立 MCP 服务器，为 IDE 提供 Symfony 项目开发能力

---

## 5. 各模块可独立使用性分析

### 5.1 完全独立使用

**platform**（依赖独立性：★★★★★）
```bash
# 最小安装：仅需平台库和一个桥接器
composer require symfony/ai-platform
composer require symfony/ai-open-ai-platform  # 选择一个桥接器
```
```php
// 无需其他 symfony/ai 模块
use Symfony\AI\Platform\Bridge\OpenAI\PlatformFactory;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$response = $platform->request(new GPT('gpt-4o'), new UserMessage('Hello'));
```

**mate**（依赖独立性：★★★★★）
```bash
# 作为全局工具安装
composer global require symfony/ai-mate
# 或项目内安装
composer require symfony/ai-mate
```
```bash
# 直接运行，不需要其他 symfony/ai 组件
./vendor/bin/mate
```

### 5.2 需要 platform 支持

**agent**（依赖独立性：★★★★☆）
```bash
composer require symfony/ai-agent symfony/ai-open-ai-platform
```
- 运行时仅需 platform，与 store/chat 完全解耦
- 可独立构建不依赖 store 的 Agent（纯工具调用场景）

**store**（依赖独立性：★★★★☆）
```bash
composer require symfony/ai-store symfony/ai-open-ai-platform symfony/ai-qdrant-store
```
- 运行时仅需 platform（生成嵌入向量）
- 不需要 agent 即可使用（独立文档存储/检索）

**chat**（依赖独立性：★★★☆☆）
```bash
composer require symfony/ai-chat symfony/ai-agent symfony/ai-open-ai-platform
```
- 依赖 platform + agent，需要两个内部模块
- 但仍可独立于 store 使用（无 RAG 的纯对话）

### 5.3 集成层（不可独立使用）

**ai-bundle**（依赖独立性：★★☆☆☆）
- 必须在 Symfony 应用中使用
- 运行时需要至少一个平台桥接器
- 本身不提供 AI 能力，仅提供 DI 配置集成

**mcp-bundle**（依赖独立性：★★★☆☆）
- 必须在 Symfony 应用中使用（需要 framework-bundle）
- 独立于 symfony/ai 其他模块，可单独安装
- 仅需 `mcp/sdk` 和 Symfony 框架

---

## 6. Bridge（桥接器）的独立性

### 6.1 桥接器设计原则

每个桥接器是独立的 Composer 包，位于对应组件的 `src/Bridge/` 目录：

```
src/platform/src/Bridge/
├── OpenAI/           → symfony/ai-open-ai-platform
├── Anthropic/        → symfony/ai-anthropic-platform
├── Azure/            → symfony/ai-azure-platform
├── Gemini/           → symfony/ai-gemini-platform
├── VertexAI/         → symfony/ai-vertex-ai-platform
├── Ollama/           → symfony/ai-ollama-platform
├── Mistral/          → symfony/ai-mistral-platform
├── Meta/             → symfony/ai-meta-platform
├── HuggingFace/      → symfony/ai-hugging-face-platform
└── ... （共 25+ 个平台桥接器）

src/store/src/Bridge/
├── Qdrant/           → symfony/ai-qdrant-store
├── Pinecone/         → symfony/ai-pinecone-store
├── MongoDB/          → symfony/ai-mongo-db-store
├── Elasticsearch/    → symfony/ai-elasticsearch-store
├── Chroma/           → symfony/ai-chroma-db-store
└── ... （共 20+ 个存储桥接器）
```

### 6.2 桥接器独立性优势

| 优势 | 说明 |
|------|------|
| **按需安装** | 只安装需要的桥接器，不引入不必要的 SDK 依赖 |
| **依赖隔离** | OpenAI SDK 和 Anthropic SDK 不会互相干扰 |
| **独立版本** | 各桥接器可独立发布新版本 |
| **轻量部署** | 生产包体积最小化 |
| **安全性** | 减少攻击面，不安装未使用的 SDK |

### 6.3 桥接器外部 SDK 依赖示例

| 桥接器 | 外部 SDK |
|--------|---------|
| OpenAI Platform Bridge | `openai-php/client`（可选） 或 `symfony/http-client` |
| Anthropic Platform Bridge | 直接使用 HTTP，无官方 PHP SDK |
| Google Gemini Bridge | `google/auth ^1.47` |
| Voyage Platform Bridge | HTTP 直连 |
| TransformersPHP Bridge | `codewithkyrian/transformers` |

---

## 7. 推荐安装组合

### 7.1 最小安装（仅文本对话）

```bash
composer require symfony/ai-platform
composer require symfony/ai-open-ai-platform
```

**适用场景**：直接调用 AI API，无需 Agent、工具调用或对话历史

```php
use Symfony\AI\Platform\Bridge\OpenAI\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAI\GPT;
use Symfony\AI\Platform\Message\UserMessage;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$response = $platform->request(new GPT('gpt-4o-mini'), new UserMessage('你好'));
echo $response->getContent();
```

### 7.2 标准 Symfony Web 应用（带 DI 集成）

```bash
composer require symfony/ai-bundle
composer require symfony/ai-open-ai-platform
composer require symfony/ai-agent
```

**适用场景**：Symfony 应用内集成 AI 功能，使用 DI 容器管理

```yaml
# config/packages/symfony_ai.yaml
symfony_ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
    agent:
        default_model: gpt-4o
```

### 7.3 完整 RAG 应用（检索增强生成）

```bash
composer require symfony/ai-bundle
composer require symfony/ai-open-ai-platform  # 语言模型 + 嵌入模型
composer require symfony/ai-qdrant-store      # 向量数据库
composer require symfony/ai-agent             # Agent 工具调用
composer require symfony/ai-chat              # 对话历史管理
```

**适用场景**：知识库问答、文档检索、企业搜索助手

数据流：
```
用户提问 → Chat 加载历史 → Agent 调用 Store 工具 → Store 向量检索
→ Platform 生成回答 → Chat 保存历史 → 返回用户
```

### 7.4 MCP 服务器（暴露 Symfony 工具给 AI 客户端）

```bash
composer require symfony/mcp-bundle
```

**适用场景**：让 Claude Desktop、Cursor 等 AI 工具调用 Symfony 应用功能

```yaml
symfony_ai_mcp:
    app: 'My App'
    client_transports:
        http: true   # 或 stdio: true
```

### 7.5 Symfony 开发助手（Mate）

```bash
composer require --dev symfony/ai-mate
# 或全局安装
composer global require symfony/ai-mate
```

**适用场景**：IDE 集成（Cursor、Continue.dev），AI 辅助 Symfony 开发

### 7.6 企业级多模型 AI 平台

```bash
composer require symfony/ai-bundle
composer require symfony/ai-agent
composer require symfony/ai-store
composer require symfony/ai-chat
composer require symfony/mcp-bundle
# 多个平台桥接器
composer require symfony/ai-open-ai-platform
composer require symfony/ai-anthropic-platform
composer require symfony/ai-azure-platform
# 多个存储桥接器
composer require symfony/ai-qdrant-store
composer require symfony/ai-postgres-store
composer require symfony/ai-redis-message-store
```

---

## 8. 与 Symfony 生态系统集成点

### 8.1 各模块依赖的 Symfony 组件汇总

| Symfony 组件 | platform | agent | store | chat | ai-bundle | mcp-bundle | mate |
|-------------|:--------:|:-----:|:-----:|:----:|:---------:|:----------:|:----:|
| `symfony/clock` | ✅ | ✅ | ✅ | dev | ✅ | ❌ | ❌ |
| `symfony/config` | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| `symfony/console` | dev | ❌ | dev | dev | ✅ | ✅ | ✅ |
| `symfony/dependency-injection` | ❌ | ❌ | dev | dev | ✅ | ✅ | ✅ |
| `symfony/event-dispatcher` | ✅ | dev | ❌ | ❌ | ❌ | ❌ | ❌ |
| `symfony/event-dispatcher-contracts` | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `symfony/finder` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| `symfony/framework-bundle` | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |
| `symfony/http-client` | dev | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `symfony/http-foundation` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `symfony/http-kernel` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `symfony/property-access` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `symfony/property-info` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `symfony/psr-http-message-bridge` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `symfony/routing` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `symfony/security-core` | ❌ | ❌ | ❌ | ❌ | dev | ❌ | ❌ |
| `symfony/serializer` | ✅ | ✅ | ❌ | dev | ❌ | ❌ | ❌ |
| `symfony/service-contracts` | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `symfony/string` | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| `symfony/translation` | ❌ | dev | ❌ | ❌ | dev | ❌ | ❌ |
| `symfony/type-info` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `symfony/uid` | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `symfony/validator` | dev | ❌ | ❌ | ❌ | dev | ❌ | ❌ |

> ✅ = require（运行时），dev = require-dev（开发时），❌ = 不依赖

### 8.2 关键 Symfony 组件作用分析

**`symfony/serializer`**（platform、agent 运行时依赖）
- platform 用于序列化/反序列化 AI 平台请求响应（JSON 转 Message 对象）
- agent 用于序列化工具调用参数

**`symfony/event-dispatcher`**（platform 运行时依赖）
- platform 发出 AI 请求前/后事件（用于日志、缓存、监控）
- 实现请求拦截器模式

**`symfony/http-client`**（agent、store 运行时依赖）
- agent 用于 MCP 客户端 HTTP 通信（连接远程 MCP 服务器）
- store 用于连接向量数据库 HTTP API

**`symfony/property-info` + `symfony/type-info`**（platform、agent 运行时依赖）
- 从 PHP 类方法签名自动生成 JSON Schema
- 支持 `#[AsTool]`（agent）和 `#[McpTool]`（mcp-bundle）参数解析

**`symfony/psr-http-message-bridge`**（mcp-bundle 运行时依赖）
- 在 Symfony Request/Response 和 PSR-7 接口之间转换
- mcp/sdk 使用 PSR-7，Symfony 使用自己的 HTTP 抽象

---

## 9. 版本兼容性

### 9.1 各模块版本要求汇总

| 模块 | 包版本 | PHP 要求 | Symfony 要求 |
|------|--------|---------|------------|
| platform | ^0.6 | >=8.2 | ^7.3\|^8.0 |
| agent | ^0.6 | >=8.2 | ^7.3\|^8.0 |
| store | ^0.6 | >=8.2 | ^7.3\|^8.0 |
| chat | ^0.6 | >=8.2 | ^7.3\|^8.0 |
| ai-bundle | ^0.6 | >=8.2 | ^7.3\|^8.0 |
| mcp-bundle | — | 未显式指定 | ^7.3\|^8.0 |
| mate | — | >=8.2 | ^5.4\|^6.4\|^7.3\|^8.0 |

### 9.2 特殊版本说明

**mate 的广泛 Symfony 兼容性**：
- 支持从 Symfony 5.4 LTS 开始（其他模块要求 7.3+）
- 原因：mate 是开发工具，需要支持维护旧项目的开发者

**minimum-stability: dev**：
- 所有模块均设置 `"minimum-stability": "dev"`
- 配合 `"prefer-stable": true`（mcp-bundle）
- 允许安装 dev 版本依赖，适合快速迭代的生态

### 9.3 PHP 8.2+ 特性使用

各模块广泛使用 PHP 8.2+ 特性：
- `readonly` 属性（不可变值对象）
- `enum`（模型类型、消息角色）
- `#[Attribute]` PHP 8.0+ 原生属性
- `fibers`（异步流式处理，PHP 8.1+）
- 联合类型、命名参数

---

## 10. 循环依赖检查

### 10.1 依赖图分析

对 7 个核心模块进行有向图分析：

```
platform → （无内部依赖）
agent    → platform
store    → platform
chat     → platform, agent
ai-bundle → platform（运行时）; agent, store, chat（开发时）
mcp-bundle → （无内部依赖）
mate     → （无内部依赖）
```

### 10.2 循环依赖检测结果

**结论：✅ 无循环依赖**

拓扑排序验证：
```
Level 0（无依赖）: platform, mcp-bundle, mate
Level 1（依赖 Level 0）: agent, store
Level 2（依赖 Level 1）: chat
Level 3（集成层）: ai-bundle
```

每个模块的依赖均指向更低层级，形成干净的有向无环图（DAG）。

### 10.3 依赖方向规则

| 规则 | 说明 |
|------|------|
| ✅ 上层依赖下层 | agent 依赖 platform（合法） |
| ✅ 平行层不互依 | agent 不依赖 store（运行时） |
| ❌ 下层不能依赖上层 | platform 不依赖 agent（正确，未违反） |
| ❌ 同层不能互依（运行时） | agent ↔ store 无运行时互依（正确） |

---

## 11. 外部 AI SDK 依赖

### 11.1 平台桥接器外部依赖

| 桥接器 | 外部 SDK/库 | 安装命令 |
|--------|-----------|--------|
| OpenAI | `openai-php/client`（可选） | `composer require openai-php/client` |
| Anthropic | 纯 HTTP（无官方 PHP SDK） | 无额外依赖 |
| Azure OpenAI | `microsoft/azure-storage-blob`（部分功能） | — |
| Google Gemini | `google/auth ^1.47` | `composer require google/auth` |
| VertexAI | `google/auth ^1.47` | `composer require google/auth` |
| Ollama | 纯 HTTP | 无额外依赖 |
| Mistral | 纯 HTTP | 无额外依赖 |
| HuggingFace | 纯 HTTP | 无额外依赖 |
| Bedrock (AWS) | `aws/aws-sdk-php`（推断） | `composer require aws/aws-sdk-php` |
| TransformersPHP | `codewithkyrian/transformers` | `composer require codewithkyrian/transformers` |

### 11.2 存储桥接器外部依赖

| 桥接器 | 外部 SDK/库 |
|--------|-----------|
| Qdrant | 纯 HTTP（symfony/http-client） |
| Pinecone | 纯 HTTP |
| MongoDB | `mongodb/mongodb` |
| Elasticsearch | `elasticsearch/elasticsearch` |
| Redis Store | `symfony/cache`（Redis 适配器） |
| PostgreSQL | Doctrine DBAL 或 PDO |
| ChromaDB | 纯 HTTP |
| Milvus | 纯 HTTP |
| Neo4j | `laudis/neo4j-php-client` |
| Weaviate | 纯 HTTP |

### 11.3 MCP SDK

| 模块 | SDK | 版本 |
|------|-----|------|
| mcp-bundle | `mcp/sdk` | ^0.4 |
| mate | `mcp/sdk` | ^0.4 |

`mcp/sdk` 是 MCP 协议的官方 PHP 实现，提供：
- `Server`、`Builder`：服务器构建和运行
- `Registry`：工具/资源/提示词注册表
- `McpTool`、`McpResource`、`McpPrompt`、`McpResourceTemplate`：PHP Attribute
- `StreamableHttpTransport`：HTTP/SSE 传输层
- `StdioTransport`：stdio 传输层
- `InMemorySessionStore`、`FileSessionStore`、`Psr16SessionStore`：会话存储

---

## 12. 数据流图

### 12.1 完整 RAG 应用数据流

```
用户输入
    │
    │ "帮我总结一下我们Q3的销售策略文档"
    ▼
┌─────────────────────────────────────────────────────┐
│  Chat 组件                                           │
│  1. 加载对话历史（MessageStore）                      │
│  2. 构建完整消息列表                                  │
└──────────────────────────┬──────────────────────────┘
                           │ messages[]
                           ▼
┌─────────────────────────────────────────────────────┐
│  Agent 组件                                          │
│  3. 调用 Platform 发送消息（第一次）                   │
│  4. AI 决定调用 store_search 工具                     │
│  5. 执行工具调用（Toolbox）                           │
└──────────────────────────┬──────────────────────────┘
                           │ tool_call: store_search("Q3销售策略")
                           ▼
┌─────────────────────────────────────────────────────┐
│  Store 组件                                          │
│  6. 将查询文本转为向量（Platform Embeddings）          │
│     Platform → OpenAI text-embedding-3-small         │
│     返回：float[1536]                                │
│  7. 向量数据库相似度搜索（Qdrant）                    │
│     cosine similarity, top-k=5                       │
│  8. 返回相关文档片段                                  │
└──────────────────────────┬──────────────────────────┘
                           │ documents[]（相关文档）
                           ▼
┌─────────────────────────────────────────────────────┐
│  Agent 组件                                          │
│  9. 将文档添加到上下文                                │
│  10. 再次调用 Platform（含文档上下文）                │
└──────────────────────────┬──────────────────────────┘
                           │ messages + context
                           ▼
┌─────────────────────────────────────────────────────┐
│  Platform 组件                                       │
│  11. 发送请求到 OpenAI GPT-4o                        │
│  12. 接收流式响应（SSE）                              │
│  13. 触发 ResponseEvent（事件系统）                   │
└──────────────────────────┬──────────────────────────┘
                           │ AI 生成的摘要
                           ▼
┌─────────────────────────────────────────────────────┐
│  Chat 组件                                           │
│  14. 保存 AI 响应到消息历史（MessageStore）           │
│  15. 返回响应给调用方                                 │
└──────────────────────────┬──────────────────────────┘
                           │ response
                           ▼
                        用户收到回答
```

### 12.2 MCP 工具调用数据流

```
Claude Desktop（MCP 客户端）
    │
    │ POST /_mcp {"method":"tools/call","params":{"name":"search_products",...}}
    ▼
┌─────────────────────────────────────────────────────┐
│  mcp-bundle: McpController                           │
│  Symfony Request → PSR-7 Request                     │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  mcp/sdk: Server::run(StreamableHttpTransport)       │
│  解析 JSON-RPC → 路由到 tools/call 处理器             │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  mcp/sdk: Registry                                   │
│  查找工具 "search_products"                           │
│  → 找到 ProductService::searchProducts()             │
└──────────────────────────┬──────────────────────────┘
                           │ 执行 PHP 方法
                           ▼
┌─────────────────────────────────────────────────────┐
│  用户代码: ProductService::searchProducts()          │
│  Symfony DI 注入 → 执行业务逻辑                       │
│  返回 array $result                                  │
└──────────────────────────┬──────────────────────────┘
                           │ 工具执行结果
                           ▼
                  JSON-RPC 响应（SSE 流）
                           │
                           ▼
                  Claude Desktop 收到工具结果
                  → 生成最终回答给用户
```

### 12.3 多模型并行调用数据流（ai-bundle 高级场景）

```
用户请求
    │
    ▼
ai-bundle FailoverPlatform（故障转移）
    ├── 尝试 OpenAI GPT-4o ──── 成功 → 返回
    │                     └── 失败 ↓
    └── 降级到 Anthropic Claude-3 ── 成功 → 返回
                               └── 失败 → 抛出异常
```

---

*报告生成时间：基于各模块 composer.json 文件分析*  
*模块版本：symfony/ai 生态 ^0.6，mcp/sdk ^0.4，PHP >=8.2，Symfony ^7.3*
