# PHP 版 CrewAI：协作智能体团队

## 业务场景

你要做一个类似 CrewAI 的多智能体协作系统。一个复杂任务（如"写一篇深度技术博客"）被拆解为多个子任务，每个子任务分配给一个角色明确的 Agent，它们协作完成：研究员搜索资料 → 写手撰写初稿 → 编辑审校优化 → 最终交付。

**这是本系列最高级的场景，综合运用了库中几乎所有模块。**

**典型应用：** 内容创作团队、软件开发流程、营销方案策划、研究报告生产

## CrewAI 概念对应

| CrewAI 概念 | Symfony AI 对应 | 说明 |
|-------------|-----------------|------|
| **Crew** | `MultiAgent` | 智能体团队协调器 |
| **Agent** | `Agent` | 单个角色 Agent |
| **Task** | 消息 + 结构化指令 | 任务描述和期望输出 |
| **Tool** | `#[AsTool]` + Bridge | 工具（搜索、文件等） |
| **Process** (sequential) | 链式 Agent 调用 | 顺序执行流水线 |
| **Process** (hierarchical) | `MultiAgent` + 编排器 | 层级编排 |
| **Memory** | `MemoryInputProcessor` | 跨任务记忆 |
| **Delegation** | `Subagent` 工具 | Agent 间委托任务 |
| **Callback** | 工具事件系统 | 任务完成回调 |

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | AI 平台 |
| **Agent** | 各角色 Agent |
| **MultiAgent** | 团队编排 |
| **Toolbox + Subagent** | Agent 间委托 |
| **Agent Bridge Tavily** | 网络搜索 |
| **Agent Bridge Filesystem** | 文件读写 |
| **Agent Bridge Clock** | 时间感知 |
| **StructuredOutput** | 结构化任务输出 |
| **Memory** | 跨任务记忆传递 |
| **Chat** | 团队协作对话持久化 |
| **Toolbox Events** | 任务进度回调 |

---

## Step 1：定义团队角色

每个 Agent 有明确的角色、目标和能力。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Filesystem\Filesystem;
use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\Filesystem\Filesystem as SymfonyFilesystem;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

// === 工具箱准备 ===

$tavily = new Tavily($httpClient, $_ENV['TAVILY_API_KEY']);
$clock = new Clock(new SymfonyClock());
$filesystem = new Filesystem(
    new SymfonyFilesystem(),
    '/tmp/crew-output',
    allowWrite: true,
);

// 研究员的工具：搜索 + 时钟
$researchToolbox = new Toolbox([$tavily, $clock]);
$researchProcessor = new AgentProcessor($researchToolbox, includeSources: true);

// 写手的工具：文件系统
$writerToolbox = new Toolbox([$filesystem]);
$writerProcessor = new AgentProcessor($writerToolbox);

// === 角色 1：研究员 ===
$researcher = new Agent(
    $platform, 'gpt-4o-mini',
    [
        new SystemPromptInputProcessor(
            "# 角色：高级技术研究员\n"
            . "## 目标\n"
            . "深入研究指定主题，收集最新、最准确的信息和数据。\n\n"
            . "## 行为准则\n"
            . "- 使用 tavily_search 搜索多个角度的信息\n"
            . "- 优先引用权威来源（官方文档、论文、知名媒体）\n"
            . "- 区分事实和观点\n"
            . "- 标注信息来源\n"
            . "- 搜索至少 3 个不同子主题\n\n"
            . "## 输出格式\n"
            . "结构化研究报告：关键发现、数据点、来源列表"
        ),
        $researchProcessor,
    ],
    [$researchProcessor],
    name: 'researcher',
);

// === 角色 2：技术写手 ===
$writer = new Agent(
    $platform, 'gpt-4o',  // 用更强的模型写作
    [
        new SystemPromptInputProcessor(
            "# 角色：高级技术写手\n"
            . "## 目标\n"
            . "根据研究材料撰写高质量的技术博客文章。\n\n"
            . "## 写作风格\n"
            . "- 深入浅出，技术准确但易懂\n"
            . "- 有结构：引言 → 核心内容 → 代码示例 → 总结\n"
            . "- 中文写作，专业术语保留英文\n"
            . "- 每个段落都有明确论点\n"
            . "- 适当使用代码块和示例\n\n"
            . "## 输出\n"
            . "完整的 Markdown 格式博客文章，使用 filesystem_write 保存"
        ),
        $writerProcessor,
    ],
    [$writerProcessor],
    name: 'writer',
);

// === 角色 3：编辑 ===
$editor = new Agent(
    $platform, 'gpt-4o',
    [
        new SystemPromptInputProcessor(
            "# 角色：主编\n"
            . "## 目标\n"
            . "审核和优化文章质量，确保可发布标准。\n\n"
            . "## 审核要点\n"
            . "- 技术准确性：有无事实错误\n"
            . "- 逻辑连贯性：段落之间是否流畅\n"
            . "- 语言质量：是否有语法错误或表达不清\n"
            . "- SEO 优化：标题是否吸引人，关键词覆盖\n"
            . "- 可读性：段落长度、标题层级是否合理\n\n"
            . "## 输出\n"
            . "给出修改建议，然后输出最终版本"
        ),
    ],
    name: 'editor',
);

// === 角色 4：项目经理（编排器） ===
$projectManager = new Agent(
    $platform, 'gpt-4o-mini',
    [
        new SystemPromptInputProcessor(
            "# 角色：项目经理\n"
            . "## 目标\n"
            . "分析任务需求，将其拆解为子任务，分配给团队成员。\n\n"
            . "## 团队成员\n"
            . "- researcher：技术研究员，擅长搜索和整理信息\n"
            . "- writer：技术写手，擅长撰写技术文章\n"
            . "- editor：主编，擅长审核和优化文章\n\n"
            . "## 工作流程\n"
            . "1. 分析需求 → 2. 分配研究任务 → 3. 分配写作任务 → 4. 分配审核任务"
        ),
    ],
    name: 'project_manager',
);
```

---

## Step 2：顺序流水线模式（Sequential Process）

类似 CrewAI 的 Sequential Process，任务按顺序传递。

```php
<?php

/**
 * 顺序执行流水线：研究 → 写作 → 编辑
 * 每一步的输出作为下一步的输入
 */
function sequentialProcess(
    Agent $researcher,
    Agent $writer,
    Agent $editor,
    string $topic,
): string {
    echo "🔄 启动顺序流水线\n";
    echo "📋 主题：{$topic}\n\n";

    // === 阶段 1：研究 ===
    echo "━━━ 阶段 1/3：研究 ━━━\n";
    $researchResult = $researcher->call(new MessageBag(
        Message::ofUser("请深入研究以下主题，收集最新信息和数据：\n\n{$topic}"),
    ));
    $researchReport = $researchResult->getContent();
    echo "✅ 研究完成（" . mb_strlen($researchReport) . " 字）\n\n";

    // === 阶段 2：写作 ===
    echo "━━━ 阶段 2/3：写作 ━━━\n";
    $writeResult = $writer->call(new MessageBag(
        Message::ofUser(
            "根据以下研究材料，撰写一篇深度技术博客文章。\n\n"
            . "【研究材料】\n{$researchReport}\n\n"
            . "要求：2000-3000 字，Markdown 格式，保存到 article.md"
        ),
    ));
    $draft = $writeResult->getContent();
    echo "✅ 初稿完成\n\n";

    // === 阶段 3：编辑 ===
    echo "━━━ 阶段 3/3：编辑审核 ━━━\n";
    $editResult = $editor->call(new MessageBag(
        Message::ofUser(
            "请审核以下文章，提出修改建议，然后输出最终版本：\n\n{$draft}"
        ),
    ));
    $finalArticle = $editResult->getContent();
    echo "✅ 终稿完成\n\n";

    return $finalArticle;
}

// 执行
$article = sequentialProcess(
    $researcher, $writer, $editor,
    'PHP 在 AI/LLM 应用开发中的最新趋势和最佳实践（2025）',
);

echo "=== 最终文章 ===\n\n";
echo $article . "\n";
```

---

## Step 3：层级编排模式（Hierarchical Process）

使用 MultiAgent 实现层级编排，由项目经理自动分配任务。

```php
<?php

use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;

// 用 MultiAgent 构建层级系统
$crew = new MultiAgent(
    orchestrator: $projectManager,
    handoffs: [
        new Handoff(
            to: $researcher,
            when: ['研究', '调研', '搜索', '收集信息', '数据', '分析趋势'],
        ),
        new Handoff(
            to: $writer,
            when: ['撰写', '写作', '草稿', '文章', '博客', '文案'],
        ),
        new Handoff(
            to: $editor,
            when: ['审核', '校对', '修改', '优化', '编辑', '润色'],
        ),
    ],
    fallback: $writer,  // 默认交给写手
);

// 项目经理自动分析并路由
$result = $crew->call(new MessageBag(
    Message::ofUser('帮我收集最新的 PHP AI 开发生态信息。'),
));
echo "研究结果：\n" . $result->getContent() . "\n\n";

$result = $crew->call(new MessageBag(
    Message::ofUser('根据这些信息撰写一篇技术博客。'),
));
echo "文章初稿：\n" . $result->getContent() . "\n\n";

$result = $crew->call(new MessageBag(
    Message::ofUser('审核并优化这篇文章。'),
));
echo "最终版本：\n" . $result->getContent() . "\n";
```

---

## Step 4：Agent 间委托（Delegation via Subagent）

让 Agent 之间可以直接委托任务，类似 CrewAI 的 delegation。

```php
<?php

use Symfony\AI\Agent\Toolbox\Tool\Subagent;

// 将研究员包装为写手的工具
// 写手在写作过程中如果需要补充信息，可以直接委托研究员搜索
$researcherAsTool = new Subagent($researcher);

$enhancedWriterToolbox = new Toolbox([
    $filesystem,
    $researcherAsTool,  // 写手可以"委托"研究员
]);
$enhancedWriterProcessor = new AgentProcessor($enhancedWriterToolbox);

$enhancedWriter = new Agent(
    $platform, 'gpt-4o',
    [
        new SystemPromptInputProcessor(
            "# 角色：高级技术写手（带研究能力）\n"
            . "## 目标\n"
            . "撰写深度技术文章。如果写作中发现信息不足，可以委托研究员补充。\n\n"
            . "## 可用工具\n"
            . "- agent（研究员）：委托搜索和研究任务\n"
            . "- filesystem_write：保存文件\n\n"
            . "## 工作方式\n"
            . "先写初稿，遇到不确定的技术细节时，调用研究员工具获取准确信息。"
        ),
        $enhancedWriterProcessor,
    ],
    [$enhancedWriterProcessor],
    name: 'enhanced_writer',
);

// 写手在写作过程中会自动调用研究员搜索信息
$result = $enhancedWriter->call(new MessageBag(
    Message::ofUser(
        '撰写一篇关于 "Symfony AI 组件在企业 AI 应用中的实践" 的技术博客。'
        . '需要包含最新的代码示例和生态数据。'
    ),
));

echo $result->getContent() . "\n";
```

---

## Step 5：任务进度回调（事件系统）

使用 Toolbox 的事件系统监控任务执行进度。

```php
<?php

use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Event\ToolCallSucceeded;
use Symfony\AI\Agent\Toolbox\Event\ToolCallFailed;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();

// 监听工具调用请求
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    $toolName = $event->toolCall->name;
    echo "📡 工具调用：{$toolName}\n";
});

// 监听工具调用成功
$dispatcher->addListener(ToolCallSucceeded::class, function (ToolCallSucceeded $event) {
    $toolName = $event->tool->name;
    echo "✅ 工具完成：{$toolName}\n";
});

// 监听工具调用失败
$dispatcher->addListener(ToolCallFailed::class, function (ToolCallFailed $event) {
    $toolName = $event->tool->name;
    $error = $event->exception->getMessage();
    echo "❌ 工具失败：{$toolName} — {$error}\n";
});

// 将事件分发器注入到 Toolbox
$monitoredToolbox = new Toolbox(
    [$tavily, $clock, $filesystem],
    eventDispatcher: $dispatcher,
);
```

---

## Step 6：跨任务记忆

团队在多次协作中积累经验。

```php
<?php

use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

// 团队共享知识库（记忆）
$teamKnowledge = new StaticMemoryProvider(
    '品牌风格：技术文章偏向深入浅出风格，避免过度使用术语。',
    '目标读者：中高级 PHP 开发者，有 2-5 年经验。',
    '写作规范：标题用二级标题，代码块标注语言，每段不超过 5 句话。',
    '上次任务反馈：编辑建议增加更多实战代码示例，减少理论描述。',
    'SEO 要求：文章标题包含关键词，摘要在 160 字以内。',
);

$memoryProcessor = new MemoryInputProcessor([$teamKnowledge]);

// 将记忆注入到写手
$writerWithMemory = new Agent(
    $platform, 'gpt-4o',
    [
        new SystemPromptInputProcessor('你是技术写手。遵循团队的写作规范和历史反馈。'),
        $memoryProcessor,  // 记忆注入
        $writerProcessor,
    ],
    [$writerProcessor],
    name: 'writer_with_memory',
);
```

---

## Step 7：完整的 CrewAI 风格执行器

```php
<?php

/**
 * PHP 版 CrewAI 执行器
 *
 * 将所有模式组合：角色定义 + 顺序执行 + 委托 + 记忆 + 事件
 */
final class Crew
{
    /** @var array<string, Agent> */
    private array $agents = [];

    /** @var array<array{agent: string, task: string}> */
    private array $taskQueue = [];

    /** @var array<string, string> */
    private array $results = [];

    public function addAgent(string $role, Agent $agent): self
    {
        $this->agents[$role] = $agent;

        return $this;
    }

    /**
     * 添加任务到队列
     */
    public function addTask(string $agentRole, string $taskDescription): self
    {
        $this->taskQueue[] = ['agent' => $agentRole, 'task' => $taskDescription];

        return $this;
    }

    /**
     * 顺序执行所有任务
     *
     * @return array<string, string>
     */
    public function kickoff(): array
    {
        $context = '';  // 跨任务上下文

        foreach ($this->taskQueue as $i => $task) {
            $role = $task['agent'];
            $agent = $this->agents[$role];

            echo "━━━ 任务 " . ($i + 1) . "/" . count($this->taskQueue) . " ━━━\n";
            echo "👤 执行者：{$role}\n";
            echo "📋 任务：{$task['task']}\n\n";

            $messages = new MessageBag(
                Message::ofUser(
                    '' !== $context
                        ? "前序任务的结果：\n{$context}\n\n你的任务：{$task['task']}"
                        : $task['task']
                ),
            );

            $result = $agent->call($messages);
            $output = $result->getContent();

            $this->results[$role] = $output;
            $context = $output;  // 传递给下一个任务

            echo "✅ 完成\n\n";
        }

        return $this->results;
    }
}

// === 使用 ===

$crew = new Crew();

$crew
    ->addAgent('researcher', $researcher)
    ->addAgent('writer', $enhancedWriter)
    ->addAgent('editor', $editor);

$crew
    ->addTask('researcher', '深入研究 2025 年 PHP AI 开发生态的最新动态')
    ->addTask('writer', '根据研究材料撰写一篇 3000 字的深度技术博客')
    ->addTask('editor', '审核文章并输出发布就绪的最终版本');

echo "🚀 启动 Crew 执行\n\n";
$results = $crew->kickoff();

echo "\n=== 最终成果 ===\n\n";
echo $results['editor'] . "\n";
```

---

## Step 8：异步执行（结合 Symfony Messenger）

> 在生产环境中，长时间运行的 AI 任务应该放到消息队列中异步执行。

```php
<?php

// 定义任务消息
namespace App\Message;

final class CrewTaskMessage
{
    public function __construct(
        public readonly string $crewId,
        public readonly string $agentRole,
        public readonly string $task,
        public readonly string $previousContext = '',
    ) {
    }
}
```

```php
<?php

// 消息处理器
namespace App\MessageHandler;

use App\Message\CrewTaskMessage;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;
use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Notifier\NotifierInterface;
use Symfony\Component\Notifier\Notification\Notification;

#[AsMessageHandler]
final class CrewTaskHandler
{
    public function __construct(
        private array $agents,           // 注入 Agent 列表
        private MessageBusInterface $bus, // 消息总线
        private NotifierInterface $notifier, // 通知组件
    ) {
    }

    public function __invoke(CrewTaskMessage $message): void
    {
        $agent = $this->agents[$message->agentRole];

        // 执行 AI 任务
        $result = $agent->call(new MessageBag(
            Message::ofUser(
                '' !== $message->previousContext
                    ? "前序结果：\n{$message->previousContext}\n\n任务：{$message->task}"
                    : $message->task
            ),
        ));

        $output = $result->getContent();

        // 可以触发下一个任务（链式消息）
        // $this->bus->dispatch(new CrewTaskMessage(...));

        // 通知结果
        $this->notifier->send(
            new Notification(
                "Crew 任务完成：{$message->agentRole}",
                ['email', 'slack'],
            ),
        );
    }
}
```

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            crew_tasks:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 2
                    delay: 1000
        routing:
            'App\Message\CrewTaskMessage': crew_tasks
```

---

## 完整架构图

```
                    ┌──────────────┐
                    │  用户/API    │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ 项目经理 PM  │  ← MultiAgent 编排器
                    └──────┬───────┘
                           │ 分配任务
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
     ┌─────────┐    ┌─────────┐    ┌─────────┐
     │ 研究员  │    │  写手   │    │  编辑   │
     │Researcher│   │ Writer  │    │ Editor  │
     └────┬────┘    └────┬────┘    └─────────┘
          │              │
     ┌────┴────┐    ┌────┴────┐
     │ Tavily  │    │Subagent │ ← 委托研究员
     │ Clock   │    │Filesys  │
     └─────────┘    └─────────┘

     ↕ 事件系统（进度回调）
     ↕ 记忆系统（跨任务上下文）
     ↕ Messenger（异步队列）
     ↕ Notifier（完成通知）
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Agent` (name 参数) | 每个 Agent 有明确角色名 |
| `MultiAgent` | 项目经理编排器，自动路由 |
| `Handoff` | 基于关键词的任务路由规则 |
| `Subagent` 工具 | Agent 间委托，A 可以调用 B |
| 顺序流水线 | 上下文传递：研究→写作→编辑 |
| 层级编排 | PM 分析需求后自动分配 |
| 事件系统 | `ToolCallRequested/Succeeded/Failed` |
| 跨任务记忆 | `StaticMemoryProvider` 注入团队知识 |
| Symfony Messenger | 异步执行耗时 AI 任务 |
| Symfony Notifier | 任务完成后发送通知 |

## 与 Python CrewAI 的对比

| 维度 | Python CrewAI | PHP Symfony AI |
|------|---------------|----------------|
| 语言生态 | Python AI 原生 | PHP Web 原生 |
| Agent 定义 | 装饰器 + YAML | `#[AsTool]` 属性 + PHP 对象 |
| 工具系统 | LangChain Tools | Toolbox + Bridge |
| 编排模式 | Sequential / Hierarchical | 手动链式 / MultiAgent |
| 异步能力 | asyncio | Symfony Messenger（消息队列） |
| Web 集成 | 需额外框架 | Symfony 原生（路由/控制器/模板） |
| 持久化 | 需外部存储 | Symfony 原生（Doctrine/Redis） |
| 适合场景 | 数据科学/ML | Web 应用/企业系统 |

**PHP 版的优势：**
- 与 Symfony 生态无缝集成（Messenger、Notifier、Doctrine、Security）
- 天然适合 Web 应用场景
- 强类型系统（PHP 8.x 属性、只读属性、联合类型）
- 成熟的部署和运维生态

---

## 系列总结

🎉 恭喜！你已经从最基础的聊天机器人（01）走到了最高级的 CrewAI 风格多智能体协作系统（23）。

**回顾学习路径：**
- **01-07**：掌握单一模块（Platform、Chat、Agent、Store、StructuredOutput、多模态）
- **08-10**：学会模块组合（记忆、搜索、全模块集成）
- **11-18**：真实业务场景（邮件、审核、翻译、报告等）
- **19-22**：多模态应用（图片生成、语音交互、视频分析、内容流水线）
- **23**：终极场景 — 多智能体协作团队
