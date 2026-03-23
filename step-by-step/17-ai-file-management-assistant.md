# AI 智能文件管理助手

## 业务场景

你的团队有一个共享文件夹，里面累积了数百个文档文件（会议纪要、项目方案、周报、合同等），命名混乱、分类不清。你需要一个 AI 助手来帮你浏览、搜索、整理和管理这些文件。AI 可以读取文件内容后做摘要、分类、重命名、移动到合适的文件夹。

**典型应用：** 文件整理与归档、知识库管理、合同/文档搜索、团队文件助手

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent** | 智能体框架 + 工具调用 |
| **Agent Bridge Filesystem** | 完整的文件系统操作（读/写/移动/列出等） |
| **Chat** | 持久化文件管理对话 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-agent-filesystem
```

---

## Step 1：创建文件管理 Agent

Filesystem Bridge 提供了 10 个工具方法，让 AI 能完整操作文件系统。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Filesystem\Filesystem;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Filesystem\Filesystem as SymfonyFilesystem;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 创建 Filesystem 工具
// 注意安全设置：允许写入和移动，但限制文件类型
$filesystem = new Filesystem(
    filesystem: new SymfonyFilesystem(),
    basePath: '/data/team-files',           // 只能在此目录内操作
    allowWrite: true,                        // 允许创建和修改文件
    allowDelete: false,                      // 不允许删除（安全起见）
    allowedExtensions: ['txt', 'md', 'json', 'csv', 'log'],
    deniedExtensions: ['php', 'sh', 'exe'],  // 禁止操作可执行文件
    deniedPatterns: ['.*', '*.env*'],         // 排除隐藏文件和环境配置
);

$toolbox = new Toolbox([$filesystem]);
$processor = new AgentProcessor($toolbox);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个智能文件管理助手。你可以浏览、读取、创建、移动文件。'
    . "\n\n你的能力："
    . "\n- filesystem_list：列出目录内容"
    . "\n- filesystem_read：读取文件内容"
    . "\n- filesystem_write：创建或覆盖文件"
    . "\n- filesystem_append：追加内容到文件"
    . "\n- filesystem_copy：复制文件"
    . "\n- filesystem_move：移动或重命名文件"
    . "\n- filesystem_mkdir：创建目录"
    . "\n- filesystem_exists：检查文件是否存在"
    . "\n- filesystem_info：获取文件元信息"
    . "\n\n执行文件操作前，先向用户确认操作内容。"
);

$agent = new Agent(
    $platform, 'gpt-4o-mini',
    [$systemPrompt, $processor],
    [$processor],
);
```

---

## Step 2：浏览和查找文件

```php
<?php

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

---

## Step 3：文件内容摘要

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

---

## Step 4：自动整理和归档

AI 可以帮助创建目录结构、移动和重命名文件。

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

---

## Step 5：生成文件摘要索引

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
```

**AI 的工作流程：**
1. `filesystem_list(".", recursive: true)` → 获取所有文件
2. 对每个文件 `filesystem_read(file)` → 读取内容
3. AI 生成摘要
4. `filesystem_write("INDEX.md", content)` → 写入索引文件

---

## Step 6：交互式文件管理对话

使用 Chat 组件让文件管理变成持续的对话。

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as ChatStore;

$chat = new Chat($agent, new ChatStore());
$sessionId = 'file-manager-session';

$chat->initiate(new MessageBag(
    Message::forSystem('你是文件管理助手。帮用户管理团队文件夹。操作前先确认。'),
), $sessionId);

// 对话 1
$response = $chat->submit(
    Message::ofUser('帮我看看有哪些文件？'),
    $sessionId,
);
echo "助手：" . $response->getContent() . "\n\n";

// 对话 2
$response = $chat->submit(
    Message::ofUser('那个 meeting-notes-jan.txt 里面写了什么？'),
    $sessionId,
);
echo "助手：" . $response->getContent() . "\n\n";

// 对话 3
$response = $chat->submit(
    Message::ofUser('把它移到 会议纪要/ 文件夹下，重命名为 2025-01-会议纪要.txt'),
    $sessionId,
);
echo "助手：" . $response->getContent() . "\n\n";

// 对话 4
$response = $chat->submit(
    Message::ofUser('好的，执行吧。然后帮我创建一个新文件 周报/2025-W3.md，写一个周报模板。'),
    $sessionId,
);
echo "助手：" . $response->getContent() . "\n";
```

---

## Filesystem 工具速查

| 工具名 | 功能 | 参数 |
|--------|------|------|
| `filesystem_list` | 列出目录内容 | `path`, `recursive` |
| `filesystem_read` | 读取文件内容 | `path` |
| `filesystem_write` | 创建/覆盖文件 | `path`, `content` |
| `filesystem_append` | 追加内容 | `path`, `content` |
| `filesystem_copy` | 复制文件 | `source`, `destination` |
| `filesystem_move` | 移动/重命名 | `source`, `destination` |
| `filesystem_mkdir` | 创建目录 | `path` |
| `filesystem_exists` | 检查存在性 | `path` |
| `filesystem_info` | 获取元信息 | `path` |
| `filesystem_delete` | 删除文件/目录 | `path`（需 `allowDelete: true`） |

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Filesystem` Bridge | 10 个文件操作工具，沙箱化安全 |
| `basePath` | 限制 AI 只能在指定目录操作 |
| `allowWrite` / `allowDelete` | 权限控制，防止误操作 |
| `allowedExtensions` / `deniedExtensions` | 文件类型限制 |
| `deniedPatterns` | 排除模式（隐藏文件、配置文件等） |
| 多步文件操作 | AI 自动编排：列出 → 读取 → 分析 → 移动 |

## 下一步

如果你需要 AI 对收到的各种文档自动分类并路由到不同处理流程，请看 [18-document-classification-routing.md](./18-document-classification-routing.md)。
