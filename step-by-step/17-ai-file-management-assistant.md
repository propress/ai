# AI 智能文件管理助手

## 业务场景

你的团队有一个共享文件夹，里面累积了数百个文档文件（会议纪要、项目方案、周报、合同等），命名混乱、分类不清。你需要一个 AI 助手来帮你浏览、搜索、整理和管理这些文件。AI 可以读取文件内容后做摘要、分类、重命名、移动到合适的文件夹。

**典型应用：** 文件整理与归档、知识库管理、合同/文档搜索、团队文件助手

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-anthropic-platform` | 连接 Anthropic Claude AI 平台 |
| **Agent** | `symfony/ai-agent` | 智能体框架 + 工具调用 + 容错处理 |
| **Agent Bridge Filesystem** | `symfony/ai-filesystem-tool` | 完整的文件系统操作（10 个工具） |
| **Chat** | `symfony/ai-chat` | 持久化文件管理对话 |

> **💡 提示：** 本教程使用 **Anthropic（Claude）** 作为主要平台。前面的教程分别使用了 OpenAI、Gemini、Mistral 等平台。
> Symfony AI 的 Bridge 架构使切换平台只需更换一行工厂方法，所有业务代码保持不变。
> 如果你更熟悉 OpenAI，只需将 `PlatformFactory` 替换为 `Symfony\AI\Platform\Bridge\OpenAI\PlatformFactory`，模型改为 `gpt-4o-mini` 即可。

## 项目流程图

```
┌──────────┐      ┌───────────────────┐      ┌─────────────────┐      ┌──────────────┐
│  用户输入  │ ──▶ │  InputProcessor    │ ──▶ │ Agent 调用 LLM   │ ──▶ │ Claude API   │
│ (文件操作) │      │ (系统提示+工具列表) │      │ (携带工具定义)    │      │ (AI 推理)     │
└──────────┘      └───────────────────┘      └─────────────────┘      └──────┬───────┘
                                                                              │
                                                                              ▼
                                                                     ┌───────────────┐
                                                               ┌──── │ LLM 返回结果   │
                                                               │     └───────────────┘
                                                               │
                                  ┌────────────────────────────┼────────────────────────┐
                                  │                            │                        │
                                  ▼                            ▼                        ▼
                         ┌───────────────┐          ┌──────────────────┐     ┌──────────────────┐
                         │ 直接文本回复    │          │ 调用文件系统工具    │     │ 调用多个工具       │
                         │ (无需工具)     │          │ (read/write/...)  │     │ (list → read → …) │
                         └───────┬───────┘          └────────┬─────────┘     └────────┬─────────┘
                                 │                           │                        │
                                 │                           ▼                        ▼
                                 │                  ┌──────────────────┐     ┌──────────────────┐
                                 │                  │ FaultTolerant    │     │ FaultTolerant    │
                                 │                  │ Toolbox 安全执行  │     │ Toolbox 逐个执行  │
                                 │                  └────────┬─────────┘     └────────┬─────────┘
                                 │                           │                        │
                                 │                           ▼                        ▼
                                 │                  ┌──────────────────┐     ┌──────────────────┐
                                 │                  │ 结果发回 LLM      │     │ 全部结果发回 LLM  │
                                 │                  │ → 生成最终回复    │     │ → 生成最终回复    │
                                 │                  └────────┬─────────┘     └────────┬─────────┘
                                 │                           │                        │
                                 ▼                           ▼                        ▼
                             ┌────────────────────────────────────────────────────────────┐
                             │           Chat 层：持久化对话历史，支持多轮文件管理            │
                             └──────────────────────────┬─────────────────────────────────┘
                                                        │
                                                        ▼
                                                 ┌──────────────┐
                                                 │  返回给用户    │
                                                 └──────────────┘
```

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- Anthropic API 密钥（[获取地址](https://console.anthropic.com/)）

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform
composer require symfony/ai-agent symfony/ai-filesystem-tool
composer require symfony/ai-chat
```

### 设置 API 密钥

```bash
export ANTHROPIC_API_KEY="your-anthropic-api-key"
```

> **🔒 安全建议：** 不要把 API 密钥硬编码在源码中。使用环境变量或 Symfony Secrets 管理敏感信息。

---

## Step 1：配置 Filesystem 安全沙箱

Filesystem Bridge 提供了 10 个文件系统工具方法。在让 AI 操作文件之前，必须先配置安全边界。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Bridge\Filesystem\Filesystem;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\Component\Filesystem\Filesystem as SymfonyFilesystem;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY'], HttpClient::create());

// 创建 Filesystem 工具 —— 安全配置是关键
$filesystem = new Filesystem(
    filesystem: new SymfonyFilesystem(),
    basePath: '/data/team-files',           // 沙箱根目录：AI 只能在此目录内操作
    allowWrite: true,                        // 允许创建和修改文件
    allowDelete: false,                      // 禁止删除（生产环境建议关闭）
    allowedExtensions: ['txt', 'md', 'json', 'csv', 'log'],  // 白名单
    deniedExtensions: ['php', 'phar', 'sh', 'exe', 'bat'],   // 黑名单（可执行文件）
    deniedPatterns: ['.*', '*.env*'],        // 排除隐藏文件和环境配置
    maxReadSize: 10485760,                   // 单文件读取上限 10MB
);
```

### 安全参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `basePath` | `string` | — | **必填**。沙箱根目录，AI 无法逃逸此路径（内置路径遍历防护） |
| `allowWrite` | `bool` | `true` | 控制 `write`、`append`、`copy`、`move`、`mkdir` 操作 |
| `allowDelete` | `bool` | `false` | 控制 `delete` 操作，默认关闭 |
| `allowedExtensions` | `array` | `[]` | 文件扩展名白名单（空数组表示不限制） |
| `deniedExtensions` | `array` | `['php', ...]` | 文件扩展名黑名单，优先于白名单 |
| `deniedPatterns` | `array` | `['.*', '*.env*']` | Glob 模式排除规则（隐藏文件、敏感配置） |
| `maxReadSize` | `int` | `10485760` | 单文件读取大小上限（字节），防止超大文件消耗内存 |

> **🔒 安全建议：** `basePath` 是最重要的安全机制。内置的 `PathValidator` 会检查所有路径操作，
> 阻止 `../` 路径遍历攻击，并验证 `realpath()` 确保无法通过符号链接逃逸沙箱。
> 生产环境中，建议为 AI 操作目录设置独立的 Linux 用户权限，做到操作系统级隔离。

---

## Step 2：创建容错文件管理 Agent

使用 `FaultTolerantToolbox` 包装工具箱，当文件操作失败时（如路径不存在、权限不足），
错误信息会安全地返回给 LLM，让 AI 自行调整策略，而不是直接抛出异常中断对话。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;
use Symfony\AI\Agent\Toolbox\Toolbox;

// 用 FaultTolerantToolbox 包装，提高错误容忍度
$toolbox = new FaultTolerantToolbox(new Toolbox([$filesystem]));
$processor = new AgentProcessor($toolbox);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个智能文件管理助手。你可以浏览、读取、创建、移动和整理文件。'
    . "\n\n安全准则："
    . "\n- 执行写入、移动、删除操作前，先向用户确认操作内容"
    . "\n- 移动或覆盖文件前，先用 filesystem_exists 检查目标路径"
    . "\n- 批量操作时逐步汇报进度"
    . "\n- 遇到错误时，解释原因并建议替代方案"
);

$agent = new Agent(
    $platform,
    'claude-sonnet-4-20250514',
    [$systemPrompt, $processor],
    [$processor],
);
```

> **💡 提示：** `FaultTolerantToolbox` 采用装饰器模式包装内部 `Toolbox`。当工具执行抛出异常时，
> 它会捕获错误并将描述性消息返回给 LLM，而不是终止整个请求。这对文件操作尤为重要——
> 路径不存在、权限不足等情况非常常见，AI 收到错误消息后可以自动尝试其他路径或方案。

---

## Step 3：浏览和查找文件

```php
<?php

use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 场景 1：浏览目录结构
$result = $agent->call(new MessageBag(
    Message::ofUser('请帮我看看团队文件夹里都有什么？列出文件名和大小。'),
));
echo $result->getContent() . "\n\n";

// 场景 2：查找特定文件
$result = $agent->call(new MessageBag(
    Message::ofUser('帮我找一下有没有关于"Q4 预算"的文件。查看每个文件的内容，找到相关的。'),
));
echo $result->getContent() . "\n\n";
```

**AI 的工作流程：**
1. `filesystem_list(".", recursive: true)` → 获取完整目录树
2. 遍历文件名查找匹配
3. `filesystem_read("预算方案.txt")` → 读取内容确认
4. `filesystem_info("预算方案.txt")` → 获取文件大小、修改时间等元信息

---

## Step 4：文件内容摘要与信息提取

让 AI 读取文件内容并生成摘要。

```php
<?php

// 场景 3：会议纪要摘要
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '请读取 meetings/ 目录下所有文件，给每个文件写一句话摘要，'
        . '然后按日期排序列出。'
    ),
));
echo $result->getContent() . "\n\n";

// 场景 4：从多个文件提取关键信息
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '读取 contracts/ 目录下的所有合同文件，'
        . '列出每份合同的：甲方、乙方、金额、有效期。做成一个表格。'
    ),
));
echo $result->getContent() . "\n";
```

> **💡 提示：** Claude 会自动编排多步工具调用：先 `filesystem_list` 获取文件列表，
> 再逐个 `filesystem_read` 读取内容，最后综合分析并生成结构化输出。
> 整个过程中 `FaultTolerantToolbox` 确保单个文件读取失败不会中断整批操作。

---

## Step 5：自动整理和归档

AI 可以帮助创建目录结构、复制和移动文件。

```php
<?php

// 场景 5：创建目录结构
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '请在团队文件夹里创建以下目录结构：'
        . "\n- 会议纪要/"
        . "\n- 项目方案/"
        . "\n- 周报月报/"
        . "\n- 合同文档/"
        . "\n- 其他/"
    ),
));
echo $result->getContent() . "\n\n";

// 场景 6：智能分类和移动
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '请浏览根目录下所有未归类的文件，读取内容判断类别，'
        . '然后告诉我你打算怎么移动它们。移动之前先告诉我方案，等我确认。'
    ),
));
echo $result->getContent() . "\n\n";

// 确认后执行
$result = $agent->call(new MessageBag(
    Message::ofUser('方案没问题，请执行移动。'),
));
echo $result->getContent() . "\n";
```

> **⚠️ 注意：** 移动操作（`filesystem_move`）会将文件从原路径转移到新路径。
> 建议在系统提示词中要求 AI 先用 `filesystem_copy` 创建备份，再执行移动，
> 尤其在批量整理场景下——如果归类错误，还能从备份恢复。

---

## Step 6：生成文件摘要索引

让 AI 读取所有文件，生成一个摘要索引文件，方便后续查找。

```php
<?php

// 场景 7：生成索引
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '请遍历所有文件，为每个文件生成一行摘要，'
        . '格式为：[文件名] - [一句话摘要]'
        . '然后把这个索引写入 INDEX.md 文件中。'
    ),
));
echo $result->getContent() . "\n\n";

// 场景 8：追加内容到已有文件
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '在 INDEX.md 末尾追加一行：'
        . '"最后更新时间：2025年1月"'
    ),
));
echo $result->getContent() . "\n";
```

**AI 的工作流程：**
1. `filesystem_list(".", recursive: true)` → 获取所有文件
2. 对每个文件 `filesystem_read(file)` → 读取内容
3. AI 生成摘要
4. `filesystem_write("INDEX.md", content)` → 写入索引文件
5. `filesystem_append("INDEX.md", footer)` → 追加更新时间

---

## Step 7：交互式文件管理对话

使用 Chat 组件让文件管理变成持续的对话，AI 能记住上下文并连续执行多步操作。

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as ChatStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$chat = new Chat($agent, new ChatStore());

// 初始化对话，注入系统指令
$chat->initiate(new MessageBag(
    Message::forSystem(
        '你是文件管理助手。帮用户管理团队文件夹。'
        . '写入或移动操作前先确认。遇到错误时给出建议。'
    ),
));

// 对话 1：浏览文件
$response = $chat->submit(Message::ofUser('帮我看看有哪些文件？'));
echo "助手：" . $response->getContent() . "\n\n";

// 对话 2：读取内容（AI 记住上下文，知道用户在浏览哪些文件）
$response = $chat->submit(
    Message::ofUser('那个 meeting-notes-jan.txt 里面写了什么？'),
);
echo "助手：" . $response->getContent() . "\n\n";

// 对话 3：移动和重命名（AI 记住用户刚才在看哪个文件）
$response = $chat->submit(
    Message::ofUser('把它移到 会议纪要/ 文件夹下，重命名为 2025-01-会议纪要.txt'),
);
echo "助手：" . $response->getContent() . "\n\n";

// 对话 4：连续操作
$response = $chat->submit(
    Message::ofUser('好的，执行吧。然后帮我创建一个新文件 周报/2025-W3.md，写一个周报模板。'),
);
echo "助手：" . $response->getContent() . "\n";
```

> **💡 提示：** `Chat` 组件每次 `submit()` 时都会加载完整的对话历史发送给 LLM，
> 因此 AI 能在多轮对话中保持上下文——记住之前列出的文件、读取的内容、执行的操作。
> 生产环境中，可将 `InMemory\Store` 替换为 `Bridge\Redis\MessageStore`（高性能、支持 TTL 过期）
> 或 `Bridge\Doctrine\DoctrineDbalMessageStore`（关系数据库持久化）来保留跨会话的对话记录。

---

## Filesystem 工具速查

| 工具名 | 功能 | 参数 | 需要权限 |
|--------|------|------|----------|
| `filesystem_list` | 列出目录内容 | `path`, `recursive` | 读取 |
| `filesystem_read` | 读取文件内容 | `path` | 读取 |
| `filesystem_write` | 创建/覆盖文件 | `path`, `content` | `allowWrite` |
| `filesystem_append` | 追加内容到文件 | `path`, `content` | `allowWrite` |
| `filesystem_copy` | 复制文件 | `source`, `destination` | `allowWrite` |
| `filesystem_move` | 移动/重命名 | `source`, `destination` | `allowWrite` |
| `filesystem_mkdir` | 创建目录 | `path` | `allowWrite` |
| `filesystem_exists` | 检查文件是否存在 | `path` | 读取 |
| `filesystem_info` | 获取文件元信息 | `path` | 读取 |
| `filesystem_delete` | 删除文件/目录 | `path` | `allowDelete` |

---

## 生产环境建议

> **🏭 生产建议：** 在生产环境中部署 AI 文件管理助手时，请注意以下几点：
>
> **审计日志**：记录所有 AI 执行的文件操作（工具名、参数、时间戳、用户身份），
> 可通过 Agent 的 `EventDispatcher` 监听工具调用事件来实现。
>
> **文件版本控制**：对于重要文件，在 AI 执行 `write` 或 `move` 前自动创建备份（如 `.bak` 或时间戳后缀），
> 或将工作目录放在 Git 仓库中以便追踪变更和回滚。
>
> **操作系统级隔离**：为 AI 操作的 `basePath` 目录创建专用系统用户，通过 Linux 文件权限限制可访问范围，
> 即使代码层面出现漏洞，操作系统也能提供额外防护。
>
> **速率与规模限制**：通过 `AgentProcessor` 的 `maxToolCalls` 参数限制单次对话的工具调用次数，
> 避免 AI 在大目录中无限遍历消耗 Token 和时间。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Filesystem` Bridge | 10 个文件操作工具，沙箱化安全访问 |
| `basePath` | 沙箱根目录，AI 无法逃逸此路径 |
| `allowWrite` / `allowDelete` | 分别控制写入类和删除类操作的权限 |
| `allowedExtensions` / `deniedExtensions` | 文件类型白名单/黑名单过滤 |
| `deniedPatterns` | Glob 模式排除规则（隐藏文件、敏感配置等） |
| `FaultTolerantToolbox` | 装饰器模式包装 Toolbox，错误信息返回给 LLM 而非抛异常 |
| `Chat` + `MessageStore` | 持久化对话历史，支持跨请求的多轮文件管理会话 |
| 多步文件操作 | AI 自动编排：列出 → 读取 → 分析 → 确认 → 移动 |

## 下一步

如果你需要 AI 对收到的各种文档自动分类并路由到不同处理流程，请看 [18-document-classification-routing.md](./18-document-classification-routing.md)。
