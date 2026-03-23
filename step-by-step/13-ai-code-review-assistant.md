# AI 代码审查助手

## 业务场景

你在一个开发团队中，每天有大量 Pull Request 需要 Code Review。你需要一个 AI 助手来辅助代码审查：它可以读取项目中的文件，分析代码质量，发现潜在 bug，建议改进方案。开发者可以和它对话讨论代码。

**典型应用：** 代码审查自动化、代码质量检查、安全漏洞扫描、新人代码指导

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent** | 智能体框架 + 工具调用 |
| **Agent Bridge Filesystem** | 读取/列出项目文件 |
| **StructuredOutput** | 结构化的审查报告 |
| **Chat** | 持久化代码讨论对话 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-agent-filesystem
```

---

## Step 1：创建带文件访问能力的代码审查 Agent

Filesystem Bridge 让 AI 能安全地读取项目文件。它内置了安全限制：只能在指定目录操作，可以限制文件类型。

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

// 创建 Filesystem 工具：只读模式，只能访问项目目录
$filesystem = new Filesystem(
    filesystem: new SymfonyFilesystem(),
    basePath: '/path/to/your/project/src',
    allowWrite: false,           // 只读，不允许修改文件
    allowDelete: false,          // 不允许删除
    allowedExtensions: ['php', 'js', 'ts', 'py', 'yaml', 'json', 'md'],  // 只允许代码文件
    deniedPatterns: ['.*', '*.env*', '*.key', '*.pem'],  // 排除敏感文件
);

$toolbox = new Toolbox([$filesystem]);
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
);

$agent = new Agent(
    $platform, 'gpt-4o-mini',
    [$systemPrompt, $processor],
    [$processor],
);
```

---

## Step 2：让 AI 浏览项目并做审查

```php
<?php

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

---

## Step 3：针对特定文件审查

```php
<?php

// 场景 2：审查特定文件
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

---

## Step 4：结构化审查报告

将审查结果映射为结构化对象，方便集成到 CI/CD 流水线。

```php
<?php

namespace App\Dto;

final class CodeIssue
{
    /**
     * @param string $file     文件路径
     * @param ?int   $line     行号
     * @param string $severity 严重性（critical/warning/info/suggestion）
     * @param string $category 类别（security/bug/performance/style/logic）
     * @param string $message  问题描述
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
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 需要注册 PlatformSubscriber
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 重新创建 Agent（使用新 platform）
// ... （创建过程同 Step 1）

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
    ),
    Message::ofUser("请审查以下 PHP 代码：\n\n```php\n{$codeToReview}\n```"),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
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

---

## Step 5：交互式代码讨论

开发者可以和 AI 就代码问题进行多轮讨论。

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as ChatStore;

$chat = new Chat($agent, new ChatStore());

// 开始代码讨论
$reviewId = 'review-pr-42';

$chat->initiate(
    new MessageBag(
        Message::forSystem(
            '你是代码审查专家，可以读取项目文件。'
            . '开发者会和你讨论代码问题，给出具体可操作的建议。'
        ),
    ),
    $reviewId,
);

// 开发者提问
$response = $chat->submit(
    Message::ofUser('帮我看看 Service/PaymentService.php 有没有问题。'),
    $reviewId,
);
echo "AI：" . $response->getContent() . "\n\n";

// 开发者追问具体问题
$response = $chat->submit(
    Message::ofUser('你说的那个竞态条件能详细解释一下吗？怎么修复？'),
    $reviewId,
);
echo "AI：" . $response->getContent() . "\n\n";

// 开发者请求生成修复代码
$response = $chat->submit(
    Message::ofUser('能给我写一个使用数据库事务来修复这个问题的代码示例吗？'),
    $reviewId,
);
echo "AI：" . $response->getContent() . "\n";
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Filesystem` Bridge | 安全的文件读取工具，沙箱化到指定目录 |
| `allowWrite: false` | 只读模式，防止 AI 修改代码 |
| `allowedExtensions` | 限制可访问的文件类型 |
| `deniedPatterns` | 排除敏感文件（.env、密钥等） |
| `filesystem_list` | AI 浏览目录结构 |
| `filesystem_read` | AI 读取文件内容 |
| 结构化报告 | `CodeReviewReport` 对象，可集成到 CI/CD |
| 交互讨论 | Chat 持久化代码讨论上下文 |

## 下一步

如果你需要 AI 帮助做多语言内容翻译，保持品牌风格一致性，请看 [14-multilingual-translation.md](./14-multilingual-translation.md)。
