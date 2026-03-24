# AI 智能招聘筛选助手

## 业务场景

你是一家快速增长公司的 HR 负责人。一个热门岗位收到了 500 份简历，需要快速筛选出 Top 20 进入面试。传统做法：HR 逐份看简历，至少要 2-3 天。现在用 AI 自动解析简历、提取关键信息、按岗位要求打分排序，还能为每位候选人自动生成面试问题。

**典型应用：** 简历自动解析与打分、候选人匹配排序、面试问题生成、人才库构建与检索

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform** | `symfony/ai-platform` | 核心抽象层——`PlatformInterface`、消息系统、`DeferredResult`、多模态内容 |
| **Bridge（Gemini）** | `symfony/ai-gemini-platform` | 主分析平台——Google Gemini 原生支持 PDF 文件解析，无需额外 OCR |
| **Bridge（Anthropic）** | `symfony/ai-anthropic-platform` | 岗位匹配打分——Claude 的推理能力适合复杂评估 |
| **StructuredOutput** | `symfony/ai-platform`（内置） | 简历信息、评分、面试问题映射为强类型 PHP 对象 |
| **Store** | `symfony/ai-store` | 人才库向量存储——候选人语义检索与匹配 |
| **Store Bridge（Redis）** | `symfony/ai-redis-store` | 本教程的向量数据库，高性能，适合实时检索 |
| **多模态内容** | `symfony/ai-platform`（内置） | `Document::fromFile()` 直接读取 PDF/图片文件 |

## 架构概述

本教程展示了 Symfony AI 的几个核心能力：

- **多模态输入**：`Document::fromFile()` 支持直接将 PDF、图片等文件作为消息内容发送给 AI。Gemini 原生支持 PDF 解析——不需要额外的 OCR 库或文件预处理。
- **嵌套 DTO**：`ResumeProfile` 包含 `WorkExperience[]` 数组——`PlatformSubscriber` 会递归生成嵌套 JSON Schema，确保 AI 输出每一层都类型安全。
- **多平台协作**：简历解析用 Gemini（PDF 能力强），匹配打分用 Anthropic（推理能力强），嵌入向量化用 OpenAI（模型成熟）——三个平台各司其职，通过统一的 `PlatformInterface` 无缝协作。
- **Store 语义检索**：将候选人信息向量化存入 Redis，新岗位开放时可以通过语义搜索快速找到最匹配的人才。

## 项目流程图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          AI 招聘筛选完整流程                                  │
└──────────────────────────────────────────────────────────────────────────────┘

  简历文件 (PDF)                                                结构化简历
      │                                                          (DTO 对象)
      ▼                                                              ▲
┌───────────┐    ┌──────────────┐    ┌─────────────────┐    ┌───────────────┐
│ Document:: │──▶│  MessageBag   │──▶│ Platform::invoke │──▶│ PlatformSub-  │
│ fromFile() │    │ System+User+ │    │   (Gemini)       │    │ scriber 反序  │
│ (读取 PDF) │    │ Document     │    │  (PDF 原生解析)   │    │ 列化 Resume-  │
└───────────┘    └──────────────┘    └─────────────────┘    │ Profile DTO   │
                                                             └───────┬───────┘
                                                                     │
      ┌──────────────────────────────────────────────────────────────┘
      │
      ▼                              ▼                              ▼
┌───────────┐              ┌──────────────┐              ┌──────────────┐
│ 岗位匹配打分│              │ 向量化存入     │              │ 面试问题生成  │
│ (Anthropic) │              │ 人才库 (Redis) │              │ (Anthropic)  │
│ Candidate-  │              │ Indexer +      │              │ Interview-   │
│ Score DTO  │              │ Retriever      │              │ Guide DTO    │
└───────────┘              └──────────────┘              └──────────────┘
```

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer
- Redis 服务器（用于人才库向量存储）

### 安装依赖

```bash
# Gemini Bridge（PDF 简历解析）
composer require symfony/ai-platform symfony/ai-gemini-platform

# Anthropic Bridge（匹配打分和面试问题生成）
composer require symfony/ai-anthropic-platform

# Redis 向量存储（人才库）
composer require symfony/ai-store symfony/ai-redis-store
```

> **💡 提示：** 为什么用两个 AI 平台？Gemini 的多模态能力可以直接解析 PDF 文件（无需 OCR），而 Anthropic Claude 的推理链更适合做复杂的匹配评估。Bridge 架构让混合使用不同平台变得轻松——业务代码只依赖 `PlatformInterface`。

### 设置 API 密钥

```bash
export GEMINI_API_KEY="your-gemini-api-key"
export ANTHROPIC_API_KEY="sk-ant-your-api-key-here"
export OPENAI_API_KEY="sk-your-api-key-here"  # 嵌入模型
```

> **🔒 安全建议：** 招聘系统处理大量个人隐私数据（姓名、联系方式、工作经历），务必确保：API 通信使用 HTTPS、分析结果加密存储、API 密钥通过环境变量或 Vault 管理、日志中脱敏处理候选人姓名和邮箱。

---

## Step 1：定义简历结构（嵌套 DTO）

嵌套 DTO 是本教程的核心——`ResumeProfile` 包含 `WorkExperience[]`，`PlatformSubscriber` 会递归生成完整的 JSON Schema。

```php
<?php

namespace App\Dto;

final class WorkExperience
{
    /**
     * @param string $company    公司名称
     * @param string $title      职位名称
     * @param string $duration   工作时长（如 "2 年 3 个月"）
     * @param string $highlights 工作亮点和主要成就
     */
    public function __construct(
        public readonly string $company,
        public readonly string $title,
        public readonly string $duration,
        public readonly string $highlights,
    ) {
    }
}

final class ResumeProfile
{
    /**
     * @param string           $name            姓名
     * @param string           $email           邮箱
     * @param int              $yearsExperience 总工作年限
     * @param string           $education       最高学历（bachelor/master/phd/other）
     * @param string           $educationSchool 毕业院校
     * @param string           $educationMajor  专业方向
     * @param string[]         $skills          技能列表（具体技术栈，如 "PHP", "MySQL"）
     * @param WorkExperience[] $experience      工作经历（按时间倒序）
     * @param string[]         $certifications  证书/专业认证
     * @param string           $summary         个人总结（一段话概括候选人特点）
     */
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly int $yearsExperience,
        public readonly string $education,
        public readonly string $educationSchool,
        public readonly string $educationMajor,
        public readonly array $skills,
        public readonly array $experience,
        public readonly array $certifications,
        public readonly string $summary,
    ) {
    }
}
```

> **📝 知识扩展：** 嵌套 DTO 的 Schema 生成过程：
> 1. `PlatformSubscriber` 检测到 `response_format` 为类名
> 2. `ResponseFormatFactory` 读取 `ResumeProfile` 的构造函数参数和 `@param` 注解
> 3. 发现 `$experience` 的类型为 `WorkExperience[]`，递归处理 `WorkExperience`
> 4. 生成完整的嵌套 JSON Schema，AI 模型严格按此结构输出
>
> 这意味着 `@param` 注解的质量直接决定 AI 输出的准确度——描述越具体，AI 越精准。

---

## Step 2：创建多平台环境

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicPlatformFactory;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory as GeminiPlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Platform\Event\ResultEvent;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();

// 事件系统：StructuredOutput + Token 统计
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$totalCost = ['gemini' => 0, 'anthropic' => 0, 'openai' => 0];
$dispatcher->addListener(ResultEvent::class, function (ResultEvent $event) use (&$totalCost) {
    $model = $event->result->getMetadata()->get('model') ?? '';
    $usage = $event->result->getMetadata()->get('token_usage');
    if (null === $usage) {
        return;
    }
    if (str_contains($model, 'gemini')) {
        $totalCost['gemini'] += $usage->totalTokens;
    } elseif (str_contains($model, 'claude')) {
        $totalCost['anthropic'] += $usage->totalTokens;
    } else {
        $totalCost['openai'] += $usage->totalTokens;
    }
});

// Gemini —— PDF 简历解析
$geminiPlatform = GeminiPlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// Anthropic —— 匹配打分 + 面试问题
$anthropicPlatform = AnthropicPlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// OpenAI —— 嵌入模型
$openaiPlatform = OpenAiPlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    $httpClient,
);
```

> **🏭 生产建议：** 通过事件系统按平台分别统计 Token 用量，可以精确计算每份简历的分析成本——对预算控制和定价策略至关重要。500 份简历的完整分析（解析 + 打分 + 面试问题）成本大约在 $5-15 之间（取决于简历长度和模型选择）。

---

## Step 3：AI 解析 PDF 简历

Gemini 的多模态能力允许直接将 PDF 文件作为消息内容——不需要额外的 OCR 处理。

```php
<?php

use App\Dto\ResumeProfile;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// Document::fromFile() 自动检测文件 MIME 类型并编码
$messages = new MessageBag(
    Message::forSystem(
        '你是专业的 HR 简历分析师。从简历中精准提取所有关键信息。'
        . '技能要细分到具体技术栈（不要只写"编程"，要写"PHP 8.x, Symfony 7, MySQL 8"）。'
        . '工作经历按时间倒序排列，每段经历提取最有价值的成就。'
    ),
    Message::ofUser(
        '请解析这份简历，提取完整的结构化信息：',
        Document::fromFile('/path/to/resume.pdf'),
    ),
);

$result = $geminiPlatform->invoke('gemini-2.0-flash', $messages, [
    'response_format' => ResumeProfile::class,
]);

/** @var ResumeProfile $resume */
$resume = $result->asObject();

echo "=== 简历解析结果 ===\n";
echo "姓名：{$resume->name}\n";
echo "学历：{$resume->education}（{$resume->educationSchool} - {$resume->educationMajor}）\n";
echo "经验：{$resume->yearsExperience} 年\n";
echo "技能：" . implode(', ', $resume->skills) . "\n";
echo "\n工作经历：\n";
foreach ($resume->experience as $exp) {
    echo "  📌 {$exp->company} | {$exp->title} | {$exp->duration}\n";
    echo "     {$exp->highlights}\n";
}
echo "\n证书：" . implode(', ', $resume->certifications) . "\n";
```

> **📝 知识扩展：** `Document::fromFile()` 的工作原理：
> 1. 读取文件并自动检测 MIME 类型（`application/pdf`、`image/png` 等）
> 2. 将文件内容 Base64 编码
> 3. 在 Contract 序列化时，转换为各平台特定的多模态格式（Gemini 的 `inlineData`、Anthropic 的 `source.data`）
>
> Gemini 对 PDF 的解析质量很高——表格、多栏排版、中英文混排都能准确提取。

### 查看解析的元数据

```php
$metadata = $result->getMetadata();
echo "\n模型：" . $metadata->get('model') . "\n";

$usage = $metadata->get('token_usage');
echo "Token：输入 {$usage->inputTokens} + 输出 {$usage->outputTokens} = {$usage->totalTokens}\n";
```

> **⚠️ 注意：** PDF 文件的 Token 消耗较高——一份 2 页简历大约消耗 1000-3000 输入 Token（取决于内容密度）。批量处理 500 份简历时，建议先用文件大小预估成本，对超大文件（>10MB）做预警。

---

## Step 4：岗位匹配打分

使用 Anthropic Claude 进行深度匹配评估——Claude 的推理链（Chain of Thought）特别适合多维度综合评分。

```php
<?php

namespace App\Dto;

final class CandidateScore
{
    /**
     * @param int    $skillMatch       技能匹配度（0-100）
     * @param int    $experienceMatch  经验匹配度（0-100）
     * @param int    $educationMatch   学历匹配度（0-100）
     * @param int    $overallScore     综合评分（0-100，加权计算）
     * @param string $tier             候选人等级（A/B/C/D）
     * @param string $strengths        核心优势（2-3 个要点）
     * @param string $concerns         主要顾虑（2-3 个要点）
     * @param string $recommendation   推荐意见（strong_yes/yes/maybe/no）
     */
    public function __construct(
        public readonly int $skillMatch,
        public readonly int $experienceMatch,
        public readonly int $educationMatch,
        public readonly int $overallScore,
        public readonly string $tier,
        public readonly string $strengths,
        public readonly string $concerns,
        public readonly string $recommendation,
    ) {
    }
}
```

```php
<?php

use App\Dto\CandidateScore;

$jobRequirements = <<<'JOB'
岗位：高级 PHP 后端工程师
要求：
- 5 年以上 PHP 开发经验
- 精通 Symfony 或 Laravel 框架
- 熟悉 MySQL, Redis, Elasticsearch
- 有高并发系统设计经验
- 了解 Docker, Kubernetes
- 有 AI/LLM 集成经验者优先
- 本科及以上学历
JOB;

$messages = new MessageBag(
    Message::forSystem(
        '你是资深的技术招聘专家。根据岗位要求，对候选人进行客观评分。'
        . "\n评分原则："
        . "\n- 只评估与岗位要求直接相关的技能和经验"
        . "\n- 技能匹配：精通核心要求得高分，只了解得中分，不具备得低分"
        . "\n- 经验匹配：考虑年限、技术深度和项目复杂度"
        . "\n- 综合评分权重：技能 50%，经验 30%，学历 20%"
        . "\n- A 级（90+）= 强烈推荐，B 级（75-89）= 推荐，C 级（60-74）= 考虑，D 级（<60）= 不推荐"
    ),
    Message::ofUser(
        "岗位要求：\n{$jobRequirements}\n\n"
        . "候选人信息：\n"
        . "姓名：{$resume->name}\n"
        . "经验：{$resume->yearsExperience} 年\n"
        . "学历：{$resume->education} - {$resume->educationSchool}（{$resume->educationMajor}）\n"
        . "技能：" . implode(', ', $resume->skills) . "\n"
        . "个人总结：{$resume->summary}\n"
    ),
);

$result = $anthropicPlatform->invoke('claude-sonnet', $messages, [
    'response_format' => CandidateScore::class,
]);

/** @var CandidateScore $score */
$score = $result->asObject();

echo "=== 候选人评估：{$resume->name} ===\n";
echo "技能匹配：{$score->skillMatch}/100\n";
echo "经验匹配：{$score->experienceMatch}/100\n";
echo "学历匹配：{$score->educationMatch}/100\n";
echo "综合评分：{$score->overallScore}/100（{$score->tier} 级）\n";
echo "推荐：{$score->recommendation}\n";
echo "优势：{$score->strengths}\n";
echo "顾虑：{$score->concerns}\n";
```

> **💡 提示：** 系统提示中的评分权重说明（"技能 50%，经验 30%，学历 20%"）非常重要——它确保 AI 的评分逻辑透明可解释。在正式招聘流程中，这些权重应该可配置，并记录到审计日志中以满足合规要求。

---

## Step 5：人才库向量存储与语义检索

将候选人信息向量化存入 Redis，当新岗位开放时，可以通过语义搜索快速找到最匹配的人才——不需要精确的关键词匹配。

```php
<?php

use Symfony\AI\Store\Bridge\Redis\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Indexer;
use Symfony\AI\Store\Retriever;

// Redis 向量存储
$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);
$store = new Store($redis, 'talent-pool');
$store->setup(['vector_size' => 1536]); // text-embedding-3-small 的维度

// 创建 Indexer
$indexer = new Indexer($openaiPlatform, 'text-embedding-3-small', $store);

// 将候选人信息向量化存入人才库
$document = new TextDocument(
    id: 'candidate-' . md5($resume->email),
    content: sprintf(
        '%s | %d年经验 | %s（%s）| 技能：%s | %s',
        $resume->name,
        $resume->yearsExperience,
        $resume->education,
        $resume->educationMajor,
        implode(', ', $resume->skills),
        $resume->summary,
    ),
    metadata: new Metadata([
        'name' => $resume->name,
        'email' => $resume->email,
        'years' => $resume->yearsExperience,
        'education' => $resume->education,
        'skills' => implode(',', $resume->skills),
        'score' => $score->overallScore,
        'tier' => $score->tier,
        'recommendation' => $score->recommendation,
    ]),
);

$indexer->index([$document]);
echo "✅ 候选人 {$resume->name} 已入人才库\n";
```

### 语义检索匹配候选人

```php
// 创建 Retriever
$retriever = new Retriever($store, $openaiPlatform, 'text-embedding-3-small');

// 新岗位：前端 + 后端全栈工程师
$results = $retriever->retrieve('React TypeScript Node.js PHP 全栈开发 3年以上经验 有微服务架构经验');

echo "=== 全栈岗位匹配候选人 ===\n";
foreach ($results as $i => $candidate) {
    $meta = $candidate->getMetadata();
    $similarity = $candidate->getScore();
    echo ($i + 1) . ". [{$meta['tier']}级] {$meta['name']} | {$meta['years']}年 | "
        . "评分：{$meta['score']} | 相似度：" . round($similarity, 3) . "\n";
}
```

> **📝 知识扩展：** `Retriever` 的工作流程：
> 1. 将查询文本通过 `EmbeddingProvider` 向量化
> 2. 构造 `VectorQuery` 发送给 Store
> 3. Store（Redis）执行 KNN 搜索，返回最相似的文档及分数
> 4. 返回 `VectorDocument` 列表，每个包含 `score`（相似度）和 `metadata`
>
> 语义检索的优势：即使候选人简历中写的是 "React.js"，搜索 "前端开发" 也能匹配到——因为向量空间中它们语义接近。

---

## Step 6：自动生成面试问题

针对候选人的优势和顾虑，自动生成有针对性的面试问题。

```php
<?php

namespace App\Dto;

final class InterviewQuestion
{
    /**
     * @param string $question    面试问题
     * @param string $type        类型（technical/behavioral/situational）
     * @param string $intent      考察意图
     * @param string $idealAnswer 理想回答要点
     */
    public function __construct(
        public readonly string $question,
        public readonly string $type,
        public readonly string $intent,
        public readonly string $idealAnswer,
    ) {
    }
}

final class InterviewGuide
{
    /**
     * @param InterviewQuestion[] $questions  面试问题列表（6-8 个）
     * @param string[]            $focusAreas 重点考察领域
     * @param string[]            $redFlags   面试中需要警惕的信号
     */
    public function __construct(
        public readonly array $questions,
        public readonly array $focusAreas,
        public readonly array $redFlags,
    ) {
    }
}
```

```php
<?php

use App\Dto\InterviewGuide;

$messages = new MessageBag(
    Message::forSystem(
        '你是技术面试官。根据候选人简历和岗位要求，设计有针对性的面试问题。'
        . "\n问题应覆盖：技术深度验证、项目经验追问、解决问题能力、团队协作。"
        . "\n特别关注候选人的"顾虑点"——设计验证性问题来确认或排除这些顾虑。"
        . "\n每个问题必须有明确的考察意图和参考答案要点。"
    ),
    Message::ofUser(
        "岗位：高级 PHP 后端工程师\n\n"
        . "候选人：{$resume->name}（{$resume->yearsExperience}年经验）\n"
        . "学历：{$resume->education} - {$resume->educationSchool}\n"
        . "技能：" . implode(', ', $resume->skills) . "\n"
        . "优势：{$score->strengths}\n"
        . "顾虑：{$score->concerns}\n\n"
        . '请设计 6 个有针对性的面试问题。'
    ),
);

$result = $anthropicPlatform->invoke('claude-sonnet', $messages, [
    'response_format' => InterviewGuide::class,
]);

/** @var InterviewGuide $guide */
$guide = $result->asObject();

echo "=== 面试指南：{$resume->name} ===\n\n";
echo "🎯 重点考察：" . implode('、', $guide->focusAreas) . "\n\n";

foreach ($guide->questions as $i => $q) {
    echo 'Q' . ($i + 1) . " [{$q->type}]：{$q->question}\n";
    echo "   考察：{$q->intent}\n";
    echo "   参考：{$q->idealAnswer}\n\n";
}

if ([] !== $guide->redFlags) {
    echo "⚠️ 注意事项：\n";
    foreach ($guide->redFlags as $flag) {
        echo "  - {$flag}\n";
    }
}
```

---

## Step 7：流式输出面试反馈

面试结束后，面试官可以用流式输出实时生成候选人评价——特别适合在面试系统 UI 中逐字显示。

```php
<?php

$messages = new MessageBag(
    Message::forSystem(
        '你是面试评估专家。根据面试记录，生成详细的候选人评估报告。'
        . '使用 Markdown 格式，包括：技术能力、沟通表达、学习潜力、团队适配度、总体评价。'
    ),
    Message::ofUser(
        "候选人：{$resume->name}\n"
        . "岗位：高级 PHP 后端工程师\n"
        . "面试时长：45 分钟\n\n"
        . "面试记录摘要：\n"
        . "Q1（技术）：关于 Symfony 的依赖注入——回答清晰，能解释 autowiring 和 service locator 的区别\n"
        . "Q2（项目）：高并发系统设计——有实际经验，处理过日均 500 万请求的系统\n"
        . "Q3（行为）：团队冲突处理——回答偏理论，缺少具体案例\n"
    ),
);

// 使用流式输出
$result = $anthropicPlatform->invoke('claude-sonnet', $messages);

echo "=== 面试评估报告（实时生成）===\n";
foreach ($result->asStream() as $chunk) {
    echo $chunk;
}
echo "\n";
```

> **💡 提示：** `asStream()` 返回一个 `Generator`，每次迭代产生一小段文本。在 Web 应用中，可以通过 SSE（Server-Sent Events）或 WebSocket 将流式输出实时推送给前端，提升用户体验。

---

## 完整示例

```php
<?php

require 'vendor/autoload.php';

use App\Dto\CandidateScore;
use App\Dto\InterviewGuide;
use App\Dto\ResumeProfile;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicPlatformFactory;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory as GeminiPlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\AI\Store\Bridge\Redis\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Indexer;
use Symfony\AI\Store\Retriever;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// === 初始化 ===

$httpClient = HttpClient::create();
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$gemini = GeminiPlatformFactory::create($_ENV['GEMINI_API_KEY'], $httpClient, eventDispatcher: $dispatcher);
$anthropic = AnthropicPlatformFactory::create($_ENV['ANTHROPIC_API_KEY'], $httpClient, eventDispatcher: $dispatcher);
$openai = OpenAiPlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);
$store = new Store($redis, 'talent-pool');
$store->setup(['vector_size' => 1536]);
$indexer = new Indexer($openai, 'text-embedding-3-small', $store);
$retriever = new Retriever($store, $openai, 'text-embedding-3-small');

// === 1. 解析简历（Gemini） ===

$resume = $gemini->invoke('gemini-2.0-flash', new MessageBag(
    Message::forSystem('你是专业的 HR 简历分析师。从简历中精准提取所有关键信息。'),
    Message::ofUser('请解析这份简历：', Document::fromFile('/path/to/resume.pdf')),
), ['response_format' => ResumeProfile::class])->asObject();

echo "✅ 解析完成：{$resume->name}（{$resume->yearsExperience}年经验）\n";

// === 2. 岗位匹配打分（Anthropic） ===

$jobReq = '高级 PHP 后端工程师，5年以上，精通 Symfony/Laravel，熟悉 MySQL/Redis';
$score = $anthropic->invoke('claude-sonnet', new MessageBag(
    Message::forSystem('你是资深技术招聘专家。根据岗位要求客观评分。'),
    Message::ofUser("岗位：{$jobReq}\n候选人：{$resume->name}，{$resume->yearsExperience}年，技能：" . implode(',', $resume->skills)),
), ['response_format' => CandidateScore::class])->asObject();

echo "📊 评分：{$score->overallScore}/100（{$score->tier}级）| {$score->recommendation}\n";

// === 3. 存入人才库（Redis） ===

$indexer->index([new TextDocument(
    id: 'candidate-' . md5($resume->email),
    content: "{$resume->name} | {$resume->yearsExperience}年 | " . implode(', ', $resume->skills),
    metadata: new Metadata([
        'name' => $resume->name, 'score' => $score->overallScore, 'tier' => $score->tier,
    ]),
)]);

echo "💾 已存入人才库\n";

// === 4. 生成面试问题（Anthropic） ===

$guide = $anthropic->invoke('claude-sonnet', new MessageBag(
    Message::forSystem('你是技术面试官。设计有针对性的面试问题。'),
    Message::ofUser("候选人：{$resume->name}，优势：{$score->strengths}，顾虑：{$score->concerns}"),
), ['response_format' => InterviewGuide::class])->asObject();

echo "📋 生成 " . count($guide->questions) . " 个面试问题\n";
```

---

## 替代实现方案

### 方案 A：全 OpenAI 方案

如果不需要 PDF 原生解析（简历已经是文本格式），可以只用 OpenAI：

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient, eventDispatcher: $dispatcher);

// 解析、打分、面试问题全部使用 OpenAI
$resume = $platform->invoke('gpt-4o', $messages, ['response_format' => ResumeProfile::class])->asObject();
$score = $platform->invoke('gpt-4o-mini', $messages, ['response_format' => CandidateScore::class])->asObject();
```

> **💡 提示：** 注意模型选择策略——解析简历需要强理解力用 `gpt-4o`，批量打分可以用更便宜的 `gpt-4o-mini`。

### 方案 B：Ollama 本地部署（隐私合规）

对于处理敏感个人数据的场景（如欧盟 GDPR 合规），可以使用 Ollama 在本地运行模型：

```php
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory as OllamaPlatformFactory;

$platform = OllamaPlatformFactory::create(
    model: 'llama3.1:70b',
    url: 'http://localhost:11434',
    httpClient: $httpClient,
    eventDispatcher: $dispatcher,
);

// 所有数据不出本地网络
$resume = $platform->invoke('llama3.1:70b', $messages, [
    'response_format' => ResumeProfile::class,
])->asObject();
```

> **🔒 安全建议：** 本地部署意味着候选人数据不经过第三方 API——完全满足数据本地化要求。70B 参数的 Llama 3.1 在简历解析任务上的效果接近 GPT-4o，但需要 GPU 支持（至少 48GB 显存）。

### 方案 C：CachePlatform 避免重复分析

同一份简历可能需要匹配多个岗位——使用缓存避免重复解析：

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\RedisAdapter;

$cachedGemini = new CachePlatform($gemini, cache: RedisAdapter::createConnection('redis://localhost'));

// 第二次解析同一份简历时直接返回缓存
$resume = $cachedGemini->invoke('gemini-2.0-flash', $messages, [
    'response_format' => ResumeProfile::class,
    'prompt_cache_key' => 'resume-parse',
])->asObject();
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Document::fromFile()` | 多模态输入——直接将 PDF/图片文件作为消息内容 |
| 嵌套 DTO | `ResumeProfile` 包含 `WorkExperience[]`，递归生成嵌套 JSON Schema |
| 多平台协作 | Gemini 解析 PDF + Anthropic 匹配打分 + OpenAI 嵌入——各司其职 |
| `PlatformSubscriber` | StructuredOutput 核心，通过事件系统注入 Schema 并反序列化 |
| `CandidateScore` DTO | 多维度评分，`@param` 注解描述直接影响评分质量 |
| `Indexer` + `Retriever` | 人才库语义检索——比关键词匹配更智能 |
| `asStream()` | 流式输出，适合在 UI 中实时显示面试评估 |
| `ResultEvent` Token 统计 | 按平台分别统计成本，用于预算控制 |
| Redis Store | 高性能向量存储，支持 KNN 搜索 |
| `CachePlatform` | 缓存简历解析结果，同一简历匹配多岗位时避免重复调用 |

## 下一步

如果你想把 YouTube 视频变成可搜索的知识库，请看 [26-youtube-video-knowledge-base.md](./26-youtube-video-knowledge-base.md)。
