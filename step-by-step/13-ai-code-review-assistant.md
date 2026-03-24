# AI 代码审查助手

## 业务场景

你在一个开发团队中，每天有大量 Pull Request 需要 Code Review。你需要一个 AI 助手来辅助代码审查：它可以读取项目中的文件，分析代码质量，发现潜在 bug，建议改进方案。开发者可以和它对话讨论代码。

**典型应用：** 代码审查自动化、代码质量检查、安全漏洞扫描、新人代码指导

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-anthropic-platform` | 连接 AI 平台，发送消息与接收回复 |
| **Agent** | `symfony/ai-agent` | 智能体框架 + 工具调用 |
| **Agent Bridge Filesystem** | `symfony/ai-agent-filesystem` | 安全的文件读取/列出项目文件 |
| **StructuredOutput** | `symfony/ai-platform` 内置 | 结构化的审查报告 DTO |
| **Chat** | `symfony/ai-chat` | 持久化代码讨论对话 |

> **💡 提示：** 本教程使用 **Anthropic Claude** 作为主要平台。Claude 在代码理解和安全分析方面表现出色，非常适合代码审查场景。如果你更熟悉 OpenAI，只需替换 `PlatformFactory` 和模型名称即可——工具系统与具体平台无关。

## 项目流程图

```
┌──────────────┐     ┌───────────────┐     ┌──────────────────┐     ┌──────────────┐
│  开发者输入    │ ──▶ │ InputProcessor │ ──▶ │ Agent 调用 LLM    │ ──▶ │ Claude API   │
│ (审查请求)    │     │ (系统提示+工具)  │     │ (携带工具定义)     │     │ (代码分析)    │
└──────────────┘     └───────────────┘     └──────────────────┘     └──────┬───────┘
                                                                           │
                                                                           ▼
                                                                   ┌──────────────┐
                                                             ┌──── │ LLM 返回结果   │
                                                             │     └──────────────┘
                                                             │
                                 ┌───────────────────────────┼───────────────────────┐
                                 │                           │                       │
                                 ▼                           ▼                       ▼
                        ┌───────────────┐          ┌───────────────┐       ┌───────────────┐
                        │ 直接生成审查报告 │          │ 调用 filesystem │       │ 调用多个工具    │
                        │ (代码已在消息中) │          │ 读取/浏览文件   │       │ (并行读文件)    │
                        └───────┬───────┘          └───────┬───────┘       └───────┬───────┘
                                │                          │                       │
                                │                          ▼                       ▼
                                │                  ┌───────────────┐       ┌───────────────┐
                                │                  │FaultTolerant  │       │FaultTolerant  │
                                │                  │Toolbox 安全执行│       │Toolbox 安全执行│
                                │                  └───────┬───────┘       └───────┬───────┘
                                │                          │                       │
                                │                          ▼                       ▼
                                │                  ┌───────────────┐       ┌───────────────┐
                                │                  │ 文件内容发回LLM │       │ 全部结果发回LLM │
                                │                  │ → 分析代码质量  │       │ → 综合分析报告  │
                                │                  └───────┬───────┘       └───────┬───────┘
                                │                          │                       │
                                ▼                          ▼                       ▼
                           ┌──────────────────────────────────────────────────────────┐
                           │          StructuredOutput → CodeReviewReport DTO          │
                           └────────────────────────────┬─────────────────────────────┘
                                                        │
                                                        ▼
                                                 ┌──────────────┐
                                                 │ Chat 持久化    │
                                                 │ 支持多轮讨论   │
                                                 └──────────────┘
```

> **💡 提示：** `FaultTolerantToolbox` 包裹在 `Toolbox` 外层。当文件读取失败（权限不足、文件不存在等）时，它不会抛异常中断流程，而是将错误信息作为工具结果返回给 LLM，让 AI 自行决定下一步操作。

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- Anthropic API 密钥（[获取地址](https://console.anthropic.com/settings/keys)）

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform
composer require symfony/ai-agent symfony/ai-agent-filesystem
composer require symfony/ai-chat
```

> **🔒 安全建议：** 不要把 API 密钥硬编码在源码中。使用环境变量或 Symfony Secrets 管理敏感信息。

---

## Step 1：配置 Filesystem 安全沙箱

Filesystem Bridge 是代码审查的核心工具。它把 AI 的文件访问限制在安全沙箱中——只能在指定目录操作，可精确控制文件类型和敏感文件排除。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Bridge\Filesystem\Filesystem;
use Symfony\Component\Filesystem\Filesystem as SymfonyFilesystem;

// 创建 Filesystem 工具：精确控制安全边界
$filesystem = new Filesystem(
    filesystem: new SymfonyFilesystem(),
    basePath: '/path/to/your/project/src',  // 沙箱根目录，所有路径相对于此

    // === 操作权限 ===
    allowWrite: false,           // 只读模式，AI 不能修改任何文件
    allowDelete: false,          // 禁止删除操作

    // === 文件类型过滤 ===
    allowedExtensions: ['php', 'js', 'ts', 'py', 'yaml', 'json', 'md', 'twig'],
    // 只允许访问代码和配置文件，其他类型一律拒绝

    deniedExtensions: ['phar', 'sh', 'exe', 'bat'],
    // 即使在 allowedExtensions 中也会被拒绝（双重保险）

    // === 敏感文件排除 ===
    deniedPatterns: [
        '.*',        // 所有隐藏文件（.gitignore, .htaccess 等）
        '*.env*',    // 环境配置（含密钥、数据库密码）
        '*.key',     // 私钥文件
        '*.pem',     // 证书文件
        '*secret*',  // 任何包含 secret 的文件
    ],

    maxReadSize: 10485760,  // 单文件最大读取 10MB
);
```

> **🔒 安全建议：** `deniedPatterns` 使用 glob 模式匹配。务必排除 `.env*`、`*.key`、`*.pem` 等敏感文件。在代码审查场景中，始终设置 `allowWrite: false` 和 `allowDelete: false`——AI 只需要读取权限。

> **💡 提示：** `basePath` 定义了沙箱边界，所有路径都会被验证确保在此目录内。尝试通过 `../` 等方式突破沙箱会触发 `PathSecurityException`。

---

## Step 2：创建带容错能力的代码审查 Agent

用 `FaultTolerantToolbox` 包裹工具箱，确保文件读取错误不会中断审查流程。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\Component\HttpClient\HttpClient;

// 使用 Anthropic Claude —— 代码理解能力强
$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    HttpClient::create(),
);

// 用 FaultTolerantToolbox 包裹，处理文件读取错误
$innerToolbox = new Toolbox([$filesystem]);
$toolbox = new FaultTolerantToolbox($innerToolbox);

$processor = new AgentProcessor($toolbox);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个资深的代码审查专家。你可以使用文件系统工具来读取和浏览项目代码。'
    . "\n\n审查要点："
    . "\n1. 代码逻辑是否正确，有无潜在 bug"
    . "\n2. 是否有安全漏洞（SQL 注入、XSS、未验证输入等）"
    . "\n3. 代码可读性和命名规范"
    . "\n4. 是否有性能问题"
    . "\n5. 是否遵循了 SOLID 原则"
    . "\n\n审查时：先浏览目录了解项目结构，再逐个分析关键文件。给出具体的行号和改进建议。"
    . "\n如果读取某个文件失败，跳过它并继续审查其他文件。"
);

$agent = new Agent(
    $platform, 'claude-sonnet-4-20250514',
    [$systemPrompt, $processor],
    [$processor],
);
```

> **💡 提示：** 如果你使用 OpenAI，只需将 `PlatformFactory` 换为 `Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory`，模型改为 `gpt-4o-mini`，其余代码完全不变：
> ```php
> use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
> $platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
> // Agent 中模型改为 'gpt-4o-mini'
> ```

> **🏭 生产建议：** `FaultTolerantToolbox` 在生产环境中必不可少。当工具抛出异常时，它将错误信息作为 `ToolResult` 返回给 LLM，LLM 会据此生成友好提示（如"该文件无法读取，已跳过"）。如果工具不存在，它还会返回可用工具列表帮助 LLM 重新选择。

---

## Step 3：让 AI 浏览项目并做审查

```php
<?php

use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 场景 1：审查整个目录
$result = $agent->call(new MessageBag(
    Message::ofUser('请先浏览 src/ 目录的结构，然后对 Controller 目录下的文件做代码审查。'),
));

echo "=== 代码审查报告 ===\n";
echo $result->getContent() . "\n";
```

**AI 的工作流程：**
1. 调用 `filesystem_list` 列出 `src/` 目录结构
2. 调用 `filesystem_list` 列出 `Controller/` 目录的文件
3. 调用 `filesystem_read` 逐个读取 Controller 文件
4. 分析每个文件的代码质量
5. 生成综合审查报告

```php
<?php

// 场景 2：针对特定文件做安全审查
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '请读取 Service/UserService.php 文件，重点审查：'
        . "\n1. 有没有 SQL 注入风险"
        . "\n2. 输入验证是否完善"
        . "\n3. 异常处理是否得当"
    ),
));

echo $result->getContent() . "\n";
```

> **⚠️ 注意：** 对于大型项目，避免一次性让 AI 审查所有文件。建议按目录或模块分批审查。`maxReadSize` 参数（默认 10MB）限制了单文件读取上限，超大文件会被拒绝读取。你可以在系统提示中指导 AI 优先审查关键文件（Controller、Service、Security 等）。

---

## Step 4：结构化审查报告

将审查结果映射为 StructuredOutput DTO，方便集成到 CI/CD 流水线或生成标准化报告。

```php
<?php

namespace App\Dto;

final class CodeIssue
{
    /**
     * @param string $file       文件路径
     * @param ?int   $line       行号
     * @param string $severity   严重性（critical/warning/info/suggestion）
     * @param string $category   类别（security/bug/performance/style/logic）
     * @param string $message    问题描述
     * @param string $suggestion 改进建议
     */
    public function __construct(
        public readonly string $file,
        public readonly ?int $line,
        public readonly string $severity,
        public readonly string $category,
        public readonly string $message,
        public readonly string $suggestion,
    ) {
    }
}

final class CodeReviewReport
{
    /**
     * @param string      $overallRating 整体评分（A/B/C/D/F）
     * @param string      $summary       审查总结
     * @param CodeIssue[] $issues        发现的问题列表
     * @param string[]    $strengths     代码亮点
     */
    public function __construct(
        public readonly string $overallRating,
        public readonly string $summary,
        public readonly array $issues,
        public readonly array $strengths = [],
    ) {
    }
}
```

### 使用结构化报告：

```php
<?php

use App\Dto\CodeReviewReport;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// 注册 PlatformSubscriber 以启用 StructuredOutput
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 传入代码内容（也可以让 Agent 通过 Filesystem 工具读取）
$codeToReview = <<<'PHP'
<?php
class UserController {
    public function getUser($id) {
        $sql = "SELECT * FROM users WHERE id = " . $id;  // 潜在 SQL 注入
        $result = $this->db->query($sql);
        return $result;
    }

    public function deleteUser($request) {
        $id = $request->get('id');
        $this->db->query("DELETE FROM users WHERE id = $id");  // 没有权限检查
        return 'deleted';
    }
}
PHP;

$messages = new MessageBag(
    Message::forSystem(
        '你是代码审查专家。分析代码并返回结构化审查报告。'
        . '严重性：critical（必须修复）、warning（建议修复）、info（信息）、suggestion（优化建议）。'
        . '类别：security（安全）、bug（缺陷）、performance（性能）、style（风格）、logic（逻辑）。'
    ),
    Message::ofUser("请审查以下 PHP 代码：\n\n```php\n{$codeToReview}\n```"),
);

$result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => CodeReviewReport::class,
]);

$report = $result->asObject();

echo "=== 代码审查报告 ===\n";
echo "整体评分：{$report->overallRating}\n";
echo "总结：{$report->summary}\n\n";

// 按严重性排序显示问题
echo "发现的问题：\n";
foreach ($report->issues as $issue) {
    $icon = match ($issue->severity) {
        'critical' => '🔴',
        'warning' => '🟡',
        'info' => '🔵',
        'suggestion' => '💡',
        default => '❓',
    };
    echo "{$icon} [{$issue->category}] {$issue->file}";
    if (null !== $issue->line) {
        echo ":{$issue->line}";
    }
    echo "\n   {$issue->message}\n";
    echo "   建议：{$issue->suggestion}\n\n";
}

if ([] !== $report->strengths) {
    echo "代码亮点：\n";
    foreach ($report->strengths as $strength) {
        echo "  ✅ {$strength}\n";
    }
}
```

> **💡 提示：** `StructuredOutput` 利用 LLM 的 JSON 模式将回复直接映射到 PHP 对象。DTO 的属性名和 PHPDoc 注释会作为 schema 描述传给 LLM，所以中文注释也能帮助 AI 理解每个字段的含义。

---

## Step 5：交互式代码讨论（Chat 持久化）

开发者可以和 AI 就代码问题进行多轮讨论，Chat 组件会保存完整对话上下文。

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as ChatStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$chat = new Chat($agent, new ChatStore());

// 用 PR 编号作为会话 ID，方便后续查找
$reviewId = 'review-pr-42';

$chat->initiate(
    new MessageBag(
        Message::forSystem(
            '你是代码审查专家，可以读取项目文件。'
            . '开发者会和你讨论代码问题，给出具体可操作的建议。'
            . '引用代码时请给出文件名和行号。'
        ),
    ),
    $reviewId,
);

// 第一轮：开发者请求审查
$response = $chat->submit(
    Message::ofUser('帮我看看 Service/PaymentService.php 有没有问题。'),
    $reviewId,
);
echo "AI：" . $response->getContent() . "\n\n";

// 第二轮：追问具体问题（AI 记得之前的上下文）
$response = $chat->submit(
    Message::ofUser('你说的那个竞态条件能详细解释一下吗？怎么修复？'),
    $reviewId,
);
echo "AI：" . $response->getContent() . "\n\n";

// 第三轮：请求生成修复代码
$response = $chat->submit(
    Message::ofUser('能给我写一个使用数据库事务来修复这个问题的代码示例吗？'),
    $reviewId,
);
echo "AI：" . $response->getContent() . "\n";
```

> **💡 提示：** 使用 PR 编号（如 `review-pr-42`）作为 `$reviewId`，可以让同一个 PR 的所有讨论共享上下文。在生产环境中，可以将 `InMemory\Store` 替换为数据库持久化实现，这样即使服务重启也能恢复讨论。

> **💡 提示（Git 集成概念）：** 在实际 CI/CD 中，你可以结合 `git diff` 的输出只审查变更的文件。例如在 CI 脚本中执行 `git diff --name-only origin/main...HEAD` 获取变更文件列表，然后将文件内容传给 AI 做定向审查。这样既节省 Token 又聚焦于本次 PR 的变更。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Filesystem` Bridge | 安全的文件读取工具，沙箱化到 `basePath` 指定目录 |
| `allowWrite: false` | 只读模式，防止 AI 修改代码 |
| `allowedExtensions` | 白名单机制，只允许访问指定类型的文件 |
| `deniedExtensions` | 黑名单机制，强制排除的文件类型 |
| `deniedPatterns` | glob 模式排除敏感文件（`.env*`、`*.key` 等） |
| `maxReadSize` | 单文件最大读取大小（默认 10MB），防止大文件耗尽内存 |
| `FaultTolerantToolbox` | 包裹 `Toolbox`，将工具异常转为 LLM 可理解的错误信息 |
| `StructuredOutput` | 将 LLM 回复映射为 `CodeReviewReport` DTO |
| Chat 持久化 | 用 PR 编号作为会话 ID，支持多轮代码讨论 |
| `filesystem_list` | AI 浏览目录结构的工具 |
| `filesystem_read` | AI 读取文件内容的工具 |

> **🏭 生产建议：** 将代码审查集成到 CI/CD 流水线时，可以考虑：① 使用 `git diff` 只审查变更文件，减少 Token 消耗；② 对审查结果做缓存，相同文件内容不重复审查；③ 将 `CodeReviewReport` 的 critical 级别问题设为 CI 阻断条件；④ 结合静态分析工具（PHPStan、Psalm）的结果一起提供给 AI，提升审查准确度。

## 下一步

如果你需要 AI 帮助做多语言内容翻译，保持品牌风格一致性，请看 [14-multilingual-translation.md](./14-multilingual-translation.md)。
