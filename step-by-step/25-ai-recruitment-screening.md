# AI 智能招聘筛选助手

## 项目概述

本教程构建一个**生产级 AI 智能招聘筛选系统**——从 PDF 简历解析、结构化信息提取、多维度岗位匹配打分、人才库向量存储与检索，到自动生成面试指南。你将掌握 Symfony AI 中 StructuredOutput、Document 多模态处理、Store 向量检索和 Agent 编排的实战组合用法。

与其他教程不同，本教程**不使用 OpenAI 作为主平台**。我们使用 **Google Gemini** 作为主分析引擎（其多模态能力对 PDF 解析表现出色）、**Voyage** 生成 Embedding 向量（文档语义理解能力强）、**MongoDB** 存储人才库向量数据，并展示 **Anthropic Claude** 作为替代方案和 **CachePlatform** 在生产环境中的成本优化策略。

## 业务场景

你是一家快速增长的科技公司 HR 负责人。一个热门岗位在一周内收到了 500 份简历，需要快速筛选出 Top 20 进入面试环节。传统做法是 HR 逐份阅读简历并手动打分，至少需要 2-3 天，而且容易因疲劳导致评估标准不一致。现在用 AI 自动解析简历、提取关键信息、按岗位要求多维度打分排序，还能为每位进入面试的候选人自动生成定制化面试问题。

**典型应用场景：**

- 简历 PDF 自动解析与结构化提取
- 候选人多维度评分与排序
- 岗位匹配度智能打分
- 面试问题个性化生成
- 人才库构建与语义检索
- 跨岗位人才复用与推荐
- 批量简历处理与成本监控

---

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` | AI 平台统一接口、StructuredOutput、多模态消息 |
| **Gemini Bridge** | `symfony/ai-gemini-platform` | 连接 Google Gemini（主分析引擎，PDF 多模态解析） |
| **Anthropic Bridge** | `symfony/ai-anthropic-platform` | 连接 Anthropic Claude（替代方案） |
| **Voyage Bridge** | `symfony/ai-voyage-platform` | 生成 Embedding 向量（文档语义理解） |
| **Cache Bridge** | `symfony/ai-cache-platform` | 缓存解析结果，避免重复简历重复调用 API |
| **Store** | `symfony/ai-store` | 向量存储核心（文档、向量化、检索） |
| **MongoDB Store** | `symfony/ai-mongo-db-store` | MongoDB Atlas 向量数据库持久化存储 |
| **Agent** | `symfony/ai-agent` | 智能筛选助手编排（InputProcessor / OutputProcessor） |
| **SimilaritySearch** | `symfony/ai-similarity-search-tool` | Agent 人才库语义搜索工具 |

---

## 核心概念

在开始编码前，简要了解几个关键概念：

- **StructuredOutput**：让大模型返回与 PHP 类结构匹配的 JSON，自动反序列化为强类型对象。简历信息天然适合结构化输出——姓名、学历、技能列表等字段都能精确映射到 DTO。
- **Document（多模态内容）**：`Document::fromFile()` 可以直接加载 PDF 文件发送给支持多模态的大模型（如 Gemini），无需手动 OCR 或文本提取。
- **TextDocument + Metadata**：`TextDocument` 是向量存储中的基本文档单元，携带文本内容和 `Metadata` 元数据。将候选人摘要和评分标签一起存储，方便后续过滤检索。
- **Vectorizer + DocumentIndexer**：`Vectorizer` 封装了向量化平台调用，`DocumentIndexer` 编排文档处理（分块 → 向量化 → 存储）的完整流水线。
- **Retriever**：`Retriever` 封装查询向量化 + 相似度搜索，从人才库中找到语义最匹配的候选人。
- **CachePlatform**：装饰器模式——包装任意平台实例，对相同输入返回缓存结果。批量简历解析中重复投递的简历可显著降低 API 成本。

---

## 项目实现思路

整个项目分为三个层次：

1. **基础解析层**：DTO 定义 → PDF 简历结构化解析 → 岗位匹配打分（Step 1-4）
2. **存储检索层**：人才库向量化存储 → 智能人才匹配检索（Step 5-6）
3. **智能交互层**：面试指南生成 → Agent 编排 → 批量处理 → 生产集成（Step 7-10）

每一层都在前一层的基础上递进，从"解析简历"到"理解候选人"再到"智能招聘决策"。

---

## 项目流程图

```
+--------------------+
|  简历文件 (PDF)    |
|  批量 500 份       |
+--------+-----------+
         |
         v
+--------+-----------+     +---------------------+
| CachePlatform      |     | Anthropic Claude    |
| (缓存解析结果)     |     | (替代方案，Step 9)  |
+--------+-----------+     +----------+----------+
         |                            |
         v                            v
+--------+----------------------------+----------+
|          Gemini 多模态 PDF 解析                 |
|          (StructuredOutput → ResumeProfile)     |
+--------+---------------------------------------+
         |
    +----+----+
    |         |
    v         v
+---+------+  +--------+-----------+
| 岗位匹配 |  | Voyage Embedding   |
| 打分评估 |  | 文本向量化 (Step 5)|
| (Step 4) |  +--------+-----------+
+---+------+           |
    |                  v
    v         +--------+-----------+
+---+-------+ | MongoDB Atlas      |
| Candidate |  | 人才库向量存储     |
| Score DTO |  +--------+-----------+
+---+-------+          |
    |                  v
    v         +--------+-----------+
+---+-------+ | Retriever 检索    |
| 批量排序  | | + SimilaritySearch|
| Top N     | | (Step 6-7)        |
+---+-------+ +--------+-----------+
    |                  |
    v                  v
+---+-------+ +--------+-----------+
| 面试指南  | | Agent 智能招聘助手 |
| 自动生成  | | + SystemPrompt     |
| (Step 7)  | | + SimilaritySearch |
+-----------+ | (Step 8)           |
              +--------------------+
```

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- MongoDB Atlas 或本地 MongoDB 7.0+（用于人才库向量存储）

### 安装依赖

```bash
# 核心平台 + Gemini（本教程主力 LLM，PDF 多模态解析）
composer require symfony/ai-platform symfony/ai-gemini-platform

# Anthropic Claude（替代方案）
composer require symfony/ai-anthropic-platform

# Voyage 嵌入向量（文档语义理解）
composer require symfony/ai-voyage-platform

# 缓存（生产环境降低成本）
composer require symfony/ai-cache-platform

# 向量存储 + MongoDB
composer require symfony/ai-store symfony/ai-mongo-db-store

# Agent 智能助手 + 相似度搜索工具
composer require symfony/ai-agent symfony/ai-similarity-search-tool

# Symfony 基础组件
composer require symfony/event-dispatcher symfony/http-client
```

### 启动基础设施

```bash
# 启动 MongoDB Atlas Local（人才库向量存储 + 向量检索）
docker run -d --name mongodb -p 27017:27017 \
    -e MONGO_INITDB_ROOT_USERNAME=admin \
    -e MONGO_INITDB_ROOT_PASSWORD=secret \
    mongodb/mongodb-atlas-local:8.0
```

### 设置 API 密钥

```bash
export GEMINI_API_KEY="your-gemini-key-here"
export VOYAGE_API_KEY="your-voyage-key-here"
# 可选：Anthropic Claude 替代方案
export ANTHROPIC_API_KEY="sk-ant-your-key-here"
```

> [!WARNING]
> 🔒 **安全提醒：** 永远不要将 API 密钥硬编码到源代码中或提交到版本控制系统。在生产环境中，请使用 Symfony Secrets（`bin/console secrets:set`）、环境变量注入或 Vault 等密钥管理服务。

> [!NOTE]
> 本教程使用 Google Gemini 作为主分析引擎（多模态 PDF 解析表现出色）、Voyage 生成 Embedding 向量。你只需要这两个 API 密钥即可完成核心功能。Anthropic 仅用于替代方案演示。

---

## Step 1：定义简历结构 DTO

StructuredOutput 让大模型返回与 PHP 类结构匹配的 JSON，自动反序列化为强类型对象。首先定义简历信息的嵌套 DTO 结构。

### 工作经历

```php
<?php

namespace App\Dto;

/**
 * 单段工作经历
 */
final class WorkExperience
{
    /**
     * @param string $company    公司名称
     * @param string $title      职位名称
     * @param string $startDate  入职时间（如 "2021-03"）
     * @param string $endDate    离职时间（如 "2023-06" 或 "至今"）
     * @param string $duration   工作时长（如 "2 年 3 个月"）
     * @param string $highlights 工作亮点与核心成果（量化描述优先）
     */
    public function __construct(
        public readonly string $company,
        public readonly string $title,
        public readonly string $startDate,
        public readonly string $endDate,
        public readonly string $duration,
        public readonly string $highlights,
    ) {
    }
}
```

### 简历完整结构

```php
<?php

namespace App\Dto;

/**
 * AI 解析后的完整简历信息
 */
final class ResumeProfile
{
    /**
     * @param string           $name            姓名
     * @param string           $email           邮箱
     * @param string           $phone           联系电话
     * @param int              $yearsExperience 总工作年限
     * @param string           $education       最高学历（bachelor/master/phd/other）
     * @param string           $educationSchool 毕业院校
     * @param string           $educationMajor  专业
     * @param string[]         $skills          技能列表（细分到具体技术栈）
     * @param WorkExperience[] $experience      工作经历（按时间倒序）
     * @param string[]         $certifications  证书/认证
     * @param string[]         $languages       语言能力（如 "英语-流利"）
     * @param string           $summary         个人总结（AI 生成的候选人画像）
     * @param float            $parseConfidence 解析置信度（0.0 ~ 1.0）
     */
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly string $phone,
        public readonly int $yearsExperience,
        public readonly string $education,
        public readonly string $educationSchool,
        public readonly string $educationMajor,
        public readonly array $skills,
        public readonly array $experience,
        public readonly array $certifications,
        public readonly array $languages,
        public readonly string $summary,
        public readonly float $parseConfidence,
    ) {
    }
}
```

> [!TIP]
> 📝 **DTO 设计建议：** `parseConfidence` 字段非常重要——低质量的 PDF（扫描件、图片简历）可能导致解析不准确。通过置信度字段，你可以将低置信度的解析结果送人工复核，而不是完全依赖 AI 的判断。

> [!IMPORTANT]
> 使用 StructuredOutput 时，你**必须**将 `PlatformSubscriber` 注册到 `EventDispatcher`。它负责在请求发送前将 `response_format` 选项中的 PHP 类转换为大模型能理解的 JSON Schema，并在响应返回后将 JSON 反序列化为 PHP 对象。

---

## Step 2：定义岗位匹配评分 DTO

除了简历解析，还需要定义候选人评分和面试指南的 DTO 结构。

### 候选人评分

```php
<?php

namespace App\Dto;

/**
 * AI 对候选人的多维度评分结果
 */
final class CandidateScore
{
    /**
     * @param int    $skillMatch       技能匹配度（0-100）
     * @param int    $experienceMatch  经验匹配度（0-100）
     * @param int    $educationMatch   学历匹配度（0-100）
     * @param int    $projectMatch     项目相关度（0-100）
     * @param int    $overallScore     综合评分（0-100，加权计算）
     * @param string $tier             候选人等级（A/B/C/D）
     * @param string $strengths        核心优势（具体描述）
     * @param string $concerns         顾虑点（需要面试验证的疑点）
     * @param string $recommendation   推荐意见（strong_yes/yes/maybe/no）
     * @param float  $confidence       评分置信度（0.0 ~ 1.0）
     */
    public function __construct(
        public readonly int $skillMatch,
        public readonly int $experienceMatch,
        public readonly int $educationMatch,
        public readonly int $projectMatch,
        public readonly int $overallScore,
        public readonly string $tier,
        public readonly string $strengths,
        public readonly string $concerns,
        public readonly string $recommendation,
        public readonly float $confidence,
    ) {
    }
}
```

### 面试指南

```php
<?php

namespace App\Dto;

/**
 * 单个面试问题
 */
final class InterviewQuestion
{
    /**
     * @param string $question    问题内容
     * @param string $type        类型（technical/behavioral/situational）
     * @param string $intent      考察意图
     * @param string $idealAnswer 理想回答要点
     * @param string $followUp    追问方向
     */
    public function __construct(
        public readonly string $question,
        public readonly string $type,
        public readonly string $intent,
        public readonly string $idealAnswer,
        public readonly string $followUp,
    ) {
    }
}

/**
 * 候选人面试指南
 */
final class InterviewGuide
{
    /**
     * @param InterviewQuestion[] $questions      面试问题列表（6-8 个）
     * @param string[]            $focusAreas     重点考察领域
     * @param string[]            $redFlags       注意事项/风险点
     * @param string              $interviewTips  面试官建议（如何引导候选人展示真实能力）
     * @param int                 $suggestedMinutes 建议面试时长（分钟）
     */
    public function __construct(
        public readonly array $questions,
        public readonly array $focusAreas,
        public readonly array $redFlags,
        public readonly string $interviewTips,
        public readonly int $suggestedMinutes,
    ) {
    }
}
```

> [!TIP]
> 💡 **嵌套 DTO 技巧：** `InterviewGuide` 中的 `$questions` 字段类型为 `InterviewQuestion[]`。StructuredOutput 会递归解析嵌套的 DTO 类，自动生成对应的 JSON Schema。PHPDoc 中的 `@param` 注解用于标注数组元素类型，这是 StructuredOutput 识别嵌套结构的关键。

---

## Step 3：使用 Gemini 解析 PDF 简历

Google Gemini 的多模态能力可以直接处理 PDF 文件，无需手动 OCR 或文本提取，非常适合简历解析场景。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\ResumeProfile;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 1. 初始化 EventDispatcher 并注册 PlatformSubscriber（StructuredOutput 必需）
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 2. 创建 Gemini 平台实例
$platform = PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 3. 定义简历解析系统提示词
$parsePrompt = <<<'PROMPT'
你是专业的 HR 简历分析师。从简历中精准提取所有关键信息。

解析要求：
1. **技能列表** — 细分到具体技术栈（不要只写"编程"，要写"PHP 8.x, Symfony 7, MySQL 8"）
2. **工作经历** — 按时间倒序排列，亮点用量化数据描述（如"优化接口响应时间从 2s 降到 200ms"）
3. **学历信息** — 完整提取院校、专业、学位
4. **个人总结** — 用 2-3 句话概括候选人画像，突出核心竞争力
5. **解析置信度** — 如果简历格式混乱或信息不完整，降低置信度

注意：
- 如果某字段在简历中未提及，使用合理的默认值（如 email 未提供则写 "未提供"）
- 工作年限根据工作经历起止时间计算，不要依赖候选人自述
- 不要主观美化或贬低候选人的经历
PROMPT;

// 4. 加载 PDF 简历并构建消息
$messages = new MessageBag(
    Message::forSystem($parsePrompt),
    Message::ofUser(
        '请解析这份简历：',
        Document::fromFile('/path/to/resume.pdf'),
    ),
);

// 5. 调用 Gemini 解析（StructuredOutput 自动映射为 DTO）
/** @var ResumeProfile $resume */
$resume = $platform->invoke('gemini-2.0-flash', $messages, [
    'response_format' => ResumeProfile::class,
])->asObject();

// 6. 输出解析结果
echo "=== 简历解析结果 ===\n";
echo "姓名：{$resume->name}\n";
echo "邮箱：{$resume->email} | 电话：{$resume->phone}\n";
echo "学历：{$resume->education}（{$resume->educationSchool} - {$resume->educationMajor}）\n";
echo "经验：{$resume->yearsExperience} 年\n";
echo "技能：" . implode(', ', $resume->skills) . "\n";
echo "语言：" . implode(', ', $resume->languages) . "\n";
echo "解析置信度：" . number_format($resume->parseConfidence * 100, 1) . "%\n\n";

echo "工作经历：\n";
foreach ($resume->experience as $exp) {
    echo "  📌 {$exp->company} | {$exp->title} | {$exp->startDate} ~ {$exp->endDate}（{$exp->duration}）\n";
    echo "     {$exp->highlights}\n";
}

echo "\n个人画像：{$resume->summary}\n";
```

> [!TIP]
> 💡 **PDF 解析选型：** Gemini 的多模态能力可以直接理解 PDF 的视觉布局和文本内容，无需预处理。对于纯文本 PDF，解析准确度通常在 95% 以上。对于扫描件或图片 PDF，建议先检查 `parseConfidence`，低于 0.7 时送人工复核。

> [!WARNING]
> ⚠️ **文件大小注意：** `Document::fromFile()` 会将文件内容编码为 base64 发送给 API。一份普通简历 PDF 通常 100KB-2MB，在 Gemini 的限制范围内。但如果简历附带大量作品集图片导致文件过大，建议先压缩或分离附件。

---

## Step 4：岗位匹配多维度打分

使用结构化输出对候选人进行多维度岗位匹配评分。评分提示词中明确列出了所有评分维度和枚举值，引导大模型输出规范、一致的评分结果。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\CandidateScore;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 定义岗位要求
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
权重：技能 40%、经验 30%、项目 20%、学历 10%
JOB;

// 构建评分消息
$messages = new MessageBag(
    Message::forSystem(
        '你是资深的技术招聘专家。根据岗位要求，对候选人进行客观、公正的多维度评分。'
        . "\n\n评分原则：\n"
        . "1. 只评估与岗位要求相关的技能和经验，不要因无关经历加分或扣分\n"
        . "2. 综合评分按权重加权计算：技能 40% + 经验 30% + 项目 20% + 学历 10%\n"
        . "3. 等级划分：A（85+）= 强推荐、B（70-84）= 推荐、C（55-69）= 待定、D（<55）= 不推荐\n"
        . "4. 顾虑点要具体——不要写 '经验不足'，要写 '缺少高并发系统设计的实战案例'\n"
        . "5. 保持客观，不要因候选人的性别、年龄、院校背景等非能力因素影响评分\n"
    ),
    Message::ofUser(
        "岗位要求：\n{$jobRequirements}\n\n"
        . "候选人信息：\n"
        . "姓名：{$resume->name}\n"
        . "经验：{$resume->yearsExperience} 年\n"
        . "学历：{$resume->education} - {$resume->educationSchool}（{$resume->educationMajor}）\n"
        . "技能：" . implode(', ', $resume->skills) . "\n"
        . "工作经历：\n" . implode("\n", array_map(
            fn ($exp) => "  - {$exp->company} | {$exp->title} | {$exp->duration} | {$exp->highlights}",
            $resume->experience,
        )) . "\n"
        . "个人画像：{$resume->summary}\n"
    ),
);

/** @var CandidateScore $score */
$score = $platform->invoke('gemini-2.0-flash', $messages, [
    'response_format' => CandidateScore::class,
])->asObject();

echo "=== 候选人评估：{$resume->name} ===\n";
echo "技能匹配：{$score->skillMatch}/100\n";
echo "经验匹配：{$score->experienceMatch}/100\n";
echo "学历匹配：{$score->educationMatch}/100\n";
echo "项目相关：{$score->projectMatch}/100\n";
echo "综合评分：{$score->overallScore}/100（{$score->tier} 级）\n";
echo "评分置信度：" . number_format($score->confidence * 100, 1) . "%\n";
echo "推荐意见：{$score->recommendation}\n";
echo "核心优势：{$score->strengths}\n";
echo "顾虑点：{$score->concerns}\n";
```

> [!WARNING]
> ⚠️ **AI 招聘公平性警告：** AI 评分系统可能继承训练数据中的偏见。**务必注意以下几点**：
> 1. 不要在提示词中引入年龄、性别、民族等受保护特征
> 2. 定期审计评分结果，检查是否存在系统性偏差（如对某类院校的候选人评分偏低）
> 3. AI 评分应作为辅助参考，最终录用决策必须由人工做出
> 4. 建议保留所有评分记录以备合规审计

> [!TIP]
> 📝 **评分一致性技巧：** 在系统提示词中明确列出权重和等级划分标准，可以显著提升批量评分的一致性。但即便如此，LLM 的评分仍有一定随机性——对于边界候选人（如 B/C 级交界），建议进行二次评估或人工复核。

---

## Step 5：人才库向量化存储

将候选人信息存入 MongoDB 向量数据库，为后续的跨岗位人才检索做准备。Voyage 的嵌入模型在文档语义理解方面表现出色，适合简历这类长文本向量化。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Voyage\PlatformFactory as VoyagePlatformFactory;
use Symfony\AI\Store\Bridge\MongoDb\Store;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;

// 1. 创建 Voyage 嵌入平台（文档语义理解能力强）
$voyagePlatform = VoyagePlatformFactory::create($_ENV['VOYAGE_API_KEY']);

// 2. 创建向量化工具
$vectorizer = new Vectorizer($voyagePlatform, 'voyage-3');

// 3. 创建 MongoDB 向量存储
$store = new Store(
    new \MongoDB\Client('mongodb://admin:secret@localhost:27017'),
    'recruitment',   // 数据库名
    'talent_pool',   // 集合名
);

// 4. 创建文档处理器和索引器
$processor = new DocumentProcessor($vectorizer);
$indexer = new DocumentIndexer($processor);

// 5. 将候选人转化为可检索的文档
// 假设 $resume 和 $score 来自 Step 3-4
$document = new TextDocument(
    id: 'candidate-' . md5($resume->email),
    content: sprintf(
        '候选人：%s | %d年%s开发经验 | %s（%s - %s） | 核心技能：%s | 工作亮点：%s | 个人画像：%s',
        $resume->name,
        $resume->yearsExperience,
        $resume->educationMajor,
        $resume->education,
        $resume->educationSchool,
        $resume->educationMajor,
        implode(', ', $resume->skills),
        implode('; ', array_map(fn ($e) => "{$e->company}-{$e->highlights}", $resume->experience)),
        $resume->summary,
    ),
    metadata: new Metadata([
        'name' => $resume->name,
        'email' => $resume->email,
        'years' => $resume->yearsExperience,
        'education' => $resume->education,
        'school' => $resume->educationSchool,
        'major' => $resume->educationMajor,
        'skills' => implode(',', $resume->skills),
        'score' => $score->overallScore,
        'tier' => $score->tier,
        'recommendation' => $score->recommendation,
        'indexed_at' => date('Y-m-d H:i:s'),
    ]),
);

// 6. 索引入库
$indexer->index($store, [$document]);
echo "✅ 候选人 {$resume->name} 已入人才库（评分：{$score->overallScore}，等级：{$score->tier}）\n";
```

> [!TIP]
> 💡 **向量内容设计：** `TextDocument` 的 `content` 字段是被向量化的文本，直接影响检索质量。将候选人的核心竞争力（技能、经验、亮点）组织在 content 中，而将结构化标签（评分、等级、推荐意见）放在 metadata 中用于过滤。这样语义检索和标签过滤可以协同工作。

---

## Step 6：智能人才匹配检索

当有新岗位开放时，从人才库中语义检索最匹配的候选人，无需精确关键词匹配。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Voyage\PlatformFactory as VoyagePlatformFactory;
use Symfony\AI\Store\Bridge\MongoDb\Store;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Retriever;

// 初始化向量化工具和存储
$voyagePlatform = VoyagePlatformFactory::create($_ENV['VOYAGE_API_KEY']);
$vectorizer = new Vectorizer($voyagePlatform, 'voyage-3');
$store = new Store(
    new \MongoDB\Client('mongodb://admin:secret@localhost:27017'),
    'recruitment',
    'talent_pool',
);

// 创建检索器
$retriever = new Retriever($store, $vectorizer);

// 新岗位：全栈工程师
$query = '需要一位精通 React + TypeScript 前端和 PHP Symfony 后端的全栈工程师，'
    . '有 3 年以上经验，熟悉 Docker 容器化部署和 CI/CD 流程';

$candidates = $retriever->retrieve($query);

echo "=== 全栈工程师岗位匹配候选人 ===\n\n";
foreach ($candidates as $i => $candidate) {
    $meta = $candidate->metadata;
    echo ($i + 1) . ". {$meta['name']}\n";
    echo "   经验：{$meta['years']}年 | 学历：{$meta['education']}（{$meta['school']}）\n";
    echo "   历史评分：{$meta['score']}/100（{$meta['tier']} 级）\n";
    echo "   核心技能：{$meta['skills']}\n";
    echo "   入库时间：{$meta['indexed_at']}\n\n";
}
```

> [!TIP]
> 📝 **语义检索优势：** 与传统关键词搜索不同，向量检索理解语义相似性。即使候选人简历中写的是"Vue.js"而搜索条件是"React"，如果候选人有丰富的前端框架经验，也会被合理地排在结果中——因为向量空间中"前端框架经验"的语义距离很近。

---

## Step 7：自动生成个性化面试指南

根据候选人的简历信息、评分结果和顾虑点，自动生成有针对性的面试指南，包括定制化问题、考察重点和风险提醒。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\InterviewGuide;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
    eventDispatcher: $dispatcher,
);

$messages = new MessageBag(
    Message::forSystem(
        '你是资深技术面试官。根据候选人简历和岗位要求，设计有针对性的面试问题。'
        . "\n\n设计原则：\n"
        . "1. 问题应覆盖：技术深度、项目经验验证、解决问题能力、团队协作\n"
        . "2. 针对候选人的'顾虑点'设计验证性问题\n"
        . "3. 技术问题要结合候选人的技术栈，不要问无关的技术\n"
        . "4. 行为面试题使用 STAR 框架（Situation-Task-Action-Result）\n"
        . "5. 每个问题要有明确的考察意图和理想答案要点\n"
        . "6. 提供追问方向，帮助面试官深入挖掘\n"
    ),
    Message::ofUser(
        "岗位：高级 PHP 后端工程师\n"
        . "候选人：{$resume->name}（{$resume->yearsExperience}年经验）\n"
        . "学历：{$resume->education} - {$resume->educationSchool}\n"
        . "技能：" . implode(', ', $resume->skills) . "\n"
        . "核心优势：{$score->strengths}\n"
        . "顾虑点：{$score->concerns}\n"
        . "综合评分：{$score->overallScore}/100（{$score->tier} 级）\n\n"
        . '请设计 6-8 个面试问题，并给出面试官建议。'
    ),
);

/** @var InterviewGuide $guide */
$guide = $platform->invoke('gemini-2.0-flash', $messages, [
    'response_format' => InterviewGuide::class,
])->asObject();

echo "╔══════════════════════════════════════╗\n";
echo "║    面试指南：{$resume->name}\n";
echo "╚══════════════════════════════════════╝\n\n";

echo "🎯 重点考察：" . implode('、', $guide->focusAreas) . "\n";
echo "⏱️  建议时长：{$guide->suggestedMinutes} 分钟\n\n";

foreach ($guide->questions as $i => $q) {
    $typeIcon = match ($q->type) {
        'technical' => '💻',
        'behavioral' => '🤝',
        'situational' => '🎯',
        default => '❓',
    };
    echo "Q" . ($i + 1) . " {$typeIcon} [{$q->type}]\n";
    echo "   问题：{$q->question}\n";
    echo "   考察：{$q->intent}\n";
    echo "   参考：{$q->idealAnswer}\n";
    echo "   追问：{$q->followUp}\n\n";
}

echo "💡 面试官建议：{$guide->interviewTips}\n\n";

if ([] !== $guide->redFlags) {
    echo "⚠️ 注意事项：\n";
    foreach ($guide->redFlags as $flag) {
        echo "  - {$flag}\n";
    }
}
```

> [!WARNING]
> ⚠️ **面试公平性提醒：** AI 生成的面试问题应确保对所有候选人公平。避免问题涉及：婚育状况、年龄、宗教信仰、政治倾向等受保护的个人信息。面试问题必须聚焦于岗位相关的能力和经验。建议在使用前由 HR 审核问题列表。

---

## Step 8：Agent 智能招聘助手

使用 Agent 编排完整的招聘工作流——接入人才库搜索工具，让 HR 可以用自然语言查询候选人、对比分析、生成报告。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Bridge\Voyage\PlatformFactory as VoyagePlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Store\Bridge\MongoDb\Store;
use Symfony\AI\Store\Document\Vectorizer;

// 1. 初始化平台
$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);
$voyagePlatform = VoyagePlatformFactory::create($_ENV['VOYAGE_API_KEY']);

// 2. 初始化向量存储和检索工具
$vectorizer = new Vectorizer($voyagePlatform, 'voyage-3');
$store = new Store(
    new \MongoDB\Client('mongodb://admin:secret@localhost:27017'),
    'recruitment',
    'talent_pool',
);

// 3. 创建人才搜索工具
$searchTool = new SimilaritySearch($vectorizer, $store);

// 4. 定义招聘助手系统提示词
$systemPrompt = <<<'PROMPT'
你是智能招聘助手。你可以帮助 HR 完成以下任务：

1. **人才搜索** — 根据岗位描述从人才库中搜索匹配候选人
2. **候选人对比** — 对比多位候选人的优劣势
3. **招聘建议** — 根据候选人池的情况给出招聘策略建议
4. **数据统计** — 统计人才库中的技能分布、经验分布等

使用搜索工具时，将岗位核心要求转化为语义查询，以获得最相关的结果。
回答时使用清晰的结构化格式，方便 HR 快速阅读。
PROMPT;

// 5. 创建 Agent
$agent = new Agent(
    platform: $platform,
    model: 'gemini-2.0-flash',
    inputProcessors: [new SystemPromptInputProcessor($systemPrompt)],
    outputProcessors: [],
    tools: [$searchTool],
);

// 6. HR 查询示例
$messages = new MessageBag(
    Message::ofUser(
        '我们要招一位有 Kubernetes 和微服务架构经验的后端工程师，'
        . '最好熟悉 Go 或 PHP，帮我从人才库里搜索合适的候选人，并按推荐度排序。'
    ),
);

$response = $agent->call($messages);

echo "=== 招聘助手回复 ===\n\n";
echo $response->asText() . "\n";
```

> [!TIP]
> 🏭 **Agent 扩展：** 你可以为 Agent 增加更多工具——如对接 ATS（Applicant Tracking System）API 查询候选人状态、对接日历 API 自动安排面试、对接邮件 API 发送面试邀请。每个工具只需实现对应接口并注册到 Agent 即可。

---

## Step 9：替代方案 — Anthropic Claude

如果你更倾向于使用 Anthropic Claude 作为分析引擎（Claude 在长文本理解和结构化分析方面同样出色），只需替换平台工厂即可：

```php
<?php

require 'vendor/autoload.php';

use App\Dto\ResumeProfile;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// Anthropic Claude 替代方案
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 使用 Claude 解析简历（200K 上下文窗口，非常适合长简历）
$messages = new MessageBag(
    Message::forSystem(
        '你是专业的 HR 简历分析师。从简历中精准提取所有关键信息。'
        . '技能要细分到具体技术栈。工作亮点用量化数据描述。'
    ),
    Message::ofUser(
        '请解析这份简历：',
        Document::fromFile('/path/to/resume.pdf'),
    ),
);

/** @var ResumeProfile $resume */
$resume = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => ResumeProfile::class,
])->asObject();

echo "Claude 解析结果：{$resume->name}（{$resume->yearsExperience}年经验）\n";
echo "置信度：" . number_format($resume->parseConfidence * 100, 1) . "%\n";
```

你还可以使用 **CachePlatform** 来避免重复简历的重复解析，显著降低批量处理成本：

```php
<?php

use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 创建基础 Gemini 平台
$basePlatform = PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 用 CachePlatform 包装（相同简历只解析一次）
$redis = RedisAdapter::createConnection('redis://localhost:6379');
$cache = new RedisAdapter($redis, 'recruitment', 3600);
$platform = new CachePlatform($basePlatform, $cache);
```

> [!TIP]
> 💡 **缓存策略：** 招聘场景中，同一候选人可能投递多个岗位，同一份简历会被多次解析。使用 CachePlatform 后，相同输入的解析结果直接从缓存返回，API 调用量可减少 30%-50%。建议缓存时效设为 24-72 小时，因为候选人可能更新简历。

---

## Step 10：批量处理 500 份简历（完整 CLI 示例）

以下是一个完整的批量简历处理 CLI 脚本，整合了前面所有步骤，包括错误处理、Token 用量监控和进度显示。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\CandidateScore;
use App\Dto\InterviewGuide;
use App\Dto\ResumeProfile;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Bridge\Voyage\PlatformFactory as VoyagePlatformFactory;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\AI\Store\Bridge\MongoDb\Store;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\Component\EventDispatcher\EventDispatcher;

// =============================================
// 1. 初始化所有组件
// =============================================

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// Gemini 主分析平台
$platform = PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
    eventDispatcher: $dispatcher,
);

// Voyage 嵌入平台
$voyagePlatform = VoyagePlatformFactory::create($_ENV['VOYAGE_API_KEY']);
$vectorizer = new Vectorizer($voyagePlatform, 'voyage-3');

// MongoDB 人才库
$store = new Store(
    new \MongoDB\Client('mongodb://admin:secret@localhost:27017'),
    'recruitment',
    'talent_pool',
);

$processor = new DocumentProcessor($vectorizer);
$indexer = new DocumentIndexer($processor);

// =============================================
// 2. 扫描简历目录
// =============================================

$resumeDir = '/path/to/resumes/';
$pdfFiles = glob($resumeDir . '*.pdf');
$totalFiles = count($pdfFiles);

echo "╔══════════════════════════════════════╗\n";
echo "║    AI 智能招聘筛选系统               ║\n";
echo "╚══════════════════════════════════════╝\n\n";
echo "📁 发现 {$totalFiles} 份简历待处理\n\n";

// 岗位要求
$jobRequirements = <<<'JOB'
岗位：高级 PHP 后端工程师
要求：5年+ PHP，精通 Symfony/Laravel，MySQL/Redis/ES，高并发经验，Docker/K8s
权重：技能 40%、经验 30%、项目 20%、学历 10%
JOB;

// =============================================
// 3. 批量解析与评分
// =============================================

$results = [];
$failures = [];
$totalInputTokens = 0;
$totalOutputTokens = 0;
$startTime = microtime(true);

foreach ($pdfFiles as $i => $pdfFile) {
    $fileName = basename($pdfFile);
    $progress = ($i + 1) . "/{$totalFiles}";
    echo "[{$progress}] 处理 {$fileName}...";

    try {
        // Step A: 解析简历
        $parseMessages = new MessageBag(
            Message::forSystem('你是专业 HR 简历分析师。从简历中精准提取所有关键信息。技能细分到具体技术栈。'),
            Message::ofUser('请解析这份简历：', Document::fromFile($pdfFile)),
        );

        $parseResult = $platform->invoke('gemini-2.0-flash', $parseMessages, [
            'response_format' => ResumeProfile::class,
        ]);

        /** @var ResumeProfile $resume */
        $resume = $parseResult->asObject();

        // 追踪 Token 用量
        $metadata = $parseResult->getMetadata();
        $totalInputTokens += $metadata['input_tokens'] ?? 0;
        $totalOutputTokens += $metadata['output_tokens'] ?? 0;

        // 低置信度警告
        if ($resume->parseConfidence < 0.7) {
            echo " ⚠️ 低置信度（{$resume->parseConfidence}），建议人工复核\n";
            $failures[] = ['file' => $fileName, 'reason' => '解析置信度过低: ' . $resume->parseConfidence];
            continue;
        }

        // Step B: 岗位匹配打分
        $scoreMessages = new MessageBag(
            Message::forSystem('你是技术招聘专家。客观评分，保持一致性。综合评分按权重加权。'),
            Message::ofUser(
                "岗位：\n{$jobRequirements}\n\n候选人：{$resume->name} | {$resume->yearsExperience}年 | "
                . implode(', ', $resume->skills) . "\n{$resume->summary}"
            ),
        );

        $scoreResult = $platform->invoke('gemini-2.0-flash', $scoreMessages, [
            'response_format' => CandidateScore::class,
        ]);

        /** @var CandidateScore $score */
        $score = $scoreResult->asObject();

        $totalInputTokens += $scoreResult->getMetadata()['input_tokens'] ?? 0;
        $totalOutputTokens += $scoreResult->getMetadata()['output_tokens'] ?? 0;

        // Step C: 存入人才库
        $document = new TextDocument(
            id: 'candidate-' . md5($resume->email),
            content: sprintf(
                '%s | %d年经验 | %s | 技能：%s | %s',
                $resume->name,
                $resume->yearsExperience,
                $resume->education,
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
                'indexed_at' => date('Y-m-d H:i:s'),
            ]),
        );

        $indexer->index($store, [$document]);

        $tierIcon = match ($score->tier) {
            'A' => '🟢',
            'B' => '🔵',
            'C' => '🟡',
            default => '🔴',
        };

        $results[] = [
            'name' => $resume->name,
            'score' => $score->overallScore,
            'tier' => $score->tier,
            'recommendation' => $score->recommendation,
        ];

        echo " {$tierIcon} {$resume->name} | {$score->overallScore}分（{$score->tier}级）\n";

        // 速率限制保护
        usleep(200000); // 200ms 延迟

    } catch (\Throwable $e) {
        echo " ❌ 失败：{$e->getMessage()}\n";
        $failures[] = ['file' => $fileName, 'reason' => $e->getMessage()];
    }
}

// =============================================
// 4. 输出统计报告
// =============================================

$elapsed = round(microtime(true) - $startTime, 1);

// 按评分降序排列
usort($results, fn ($a, $b) => $b['score'] <=> $a['score']);

echo "\n╔══════════════════════════════════════╗\n";
echo "║       筛选结果统计报告               ║\n";
echo "╚══════════════════════════════════════╝\n\n";

echo "📊 处理统计：\n";
echo "  总简历数：{$totalFiles}\n";
echo "  成功解析：" . count($results) . "\n";
echo "  处理失败：" . count($failures) . "\n";
echo "  总耗时：{$elapsed} 秒\n";
echo "  Token 用量：输入 {$totalInputTokens} + 输出 {$totalOutputTokens}\n\n";

// 等级分布
$tierCounts = array_count_values(array_column($results, 'tier'));
echo "📈 等级分布：\n";
foreach (['A', 'B', 'C', 'D'] as $tier) {
    $count = $tierCounts[$tier] ?? 0;
    echo "  {$tier} 级：{$count} 人\n";
}

// Top 20 候选人
echo "\n🏆 Top 20 候选人（进入面试）：\n";
$top20 = array_slice($results, 0, 20);
foreach ($top20 as $i => $r) {
    echo "  " . ($i + 1) . ". {$r['name']} | {$r['score']}分（{$r['tier']}级）| {$r['recommendation']}\n";
}

// 失败记录
if ([] !== $failures) {
    echo "\n❌ 处理失败记录：\n";
    foreach ($failures as $f) {
        echo "  - {$f['file']}: {$f['reason']}\n";
    }
}
```

> [!WARNING]
> ⚠️ **速率限制注意：** Gemini API 有速率限制（RPM/TPM）。批量处理 500 份简历时，建议在每次调用之间加入适当延迟（如 `usleep(200000)` = 200ms），或使用队列异步处理。按每份简历 2 次 API 调用（解析 + 评分）计算，500 份简历约需 1000 次调用。

> [!TIP]
> 🏭 **生产建议：** 在高流量场景下，不要在 HTTP 请求中同步处理简历。使用 Symfony Messenger 将每份简历作为消息发送到队列，由多个 Worker 并行消费。这样单份简历的处理失败不会影响其他简历，且可以通过增加 Worker 数量线性扩展吞吐量。

> [!IMPORTANT]
> ⚖️ **数据合规提醒：** 简历包含大量个人隐私信息（PII）——姓名、电话、邮箱、工作经历等。务必：1）在发送给 AI 分析前评估数据处理的合法性；2）遵守 GDPR/个人信息保护法等法规；3）向量存储中设置数据保留期限；4）候选人有权要求删除其数据。使用 AI 辅助招聘的公司可能需要进行数据保护影响评估（DPIA）。

---

## 存储方案对比

本教程使用 MongoDB Atlas 作为向量存储，以下是各方案的适用场景对比：

| 存储方案 | 适用场景 | 优势 | 限制 |
|----------|----------|------|------|
| **MongoDB Atlas** | 生产环境（本教程） | 灵活 Schema，水平扩展，Atlas Search 向量检索 | 需要 Atlas 或 Atlas Local |
| **PostgreSQL pgvector** | SQL 生态偏好 | 成熟稳定，支持混合查询，SQL 生态兼容 | 需安装 pgvector 扩展 |
| **InMemory Store** | 开发、测试、演示 | 零配置，即开即用 | 数据不持久，重启丢失 |
| **Qdrant / Milvus** | 大规模向量检索（10万+） | 专为向量设计，性能最优 | 额外运维成本 |

> [!TIP]
> 💡 **MongoDB 优势：** MongoDB 的灵活 Schema 非常适合招聘场景——不同岗位的候选人 metadata 字段可能不同（如技术岗有 GitHub 链接、设计岗有作品集 URL），无需预先定义表结构。Atlas Search 还支持全文搜索 + 向量搜索的混合查询。

---

## 生产环境建议

> [!TIP]
> 🏭 **异步处理架构：** 推荐使用 Symfony Messenger + RabbitMQ/Amazon SQS 构建异步处理管线：候选人投递简历 → 消息入队 → Worker 异步解析 + 评分 + 入库 → WebSocket 通知 HR 筛选完成。这种架构支持水平扩展，高峰期加 Worker 即可。

> [!WARNING]
> ⚠️ **AI 偏见审计：** AI 招聘系统在生产使用前，**强烈建议**进行偏见审计：1）用不同性别/年龄/院校背景的简历测试评分一致性；2）检查评分分布是否存在系统性偏差；3）定期生成公平性报告；4）设置人工复核机制——特别是对 C/D 级候选人的淘汰决策。根据多国劳动法规定，使用 AI 辅助招聘决策的企业需要向候选人披露 AI 的使用情况。

> [!TIP]
> 🏭 **监控与告警：** 生产招聘系统必须配置监控：1）简历解析成功率和平均置信度（过低说明 PDF 质量问题）；2）API 调用延迟 P99（超过 10s 需要告警）；3）每日 Token 用量和成本预算监控；4）人才库增长曲线和检索命中率；5）评分分布异常检测（如某天 A 级候选人突然增多，可能是模型版本变更导致）。

---

## 关键知识点总结

| 概念 | 类 / 组件 | 本教程中的用途 |
|------|-----------|--------------|
| **StructuredOutput** | `PlatformSubscriber` + DTO 类 | 简历解析和评分结果映射为强类型对象 |
| **Gemini Bridge** | `Bridge\Gemini\PlatformFactory` | 多模态 PDF 简历解析和岗位匹配打分 |
| **Anthropic Bridge** | `Bridge\Anthropic\PlatformFactory` | 替代分析引擎（长文本理解） |
| **Voyage Bridge** | `Bridge\Voyage\PlatformFactory` | 生成候选人文本的 Embedding 向量 |
| **CachePlatform** | `Bridge\Cache\CachePlatform` | 缓存解析结果，避免重复简历重复调用 |
| **Document** | `Message\Content\Document` | 加载 PDF 简历发送给多模态 LLM |
| **TextDocument** | `Store\Document\TextDocument` | 候选人摘要 + Metadata 元数据封装 |
| **Vectorizer** | `Store\Document\Vectorizer` | 将候选人文本转换为向量嵌入（Voyage） |
| **DocumentIndexer** | `Store\Indexer\DocumentIndexer` | 编排文档的向量化和存储流水线 |
| **MongoDB Store** | `Store\Bridge\MongoDb\Store` | 人才库向量数据持久化存储 |
| **Retriever** | `Store\Retriever` | 封装查询向量化 + 相似度搜索的检索器 |
| **Agent** | `Agent\Agent` | 智能招聘助手，编排人才搜索和分析 |
| **SimilaritySearch** | `Bridge\SimilaritySearch\SimilaritySearch` | Agent 人才库语义搜索工具 |
| **SystemPromptInputProcessor** | `Agent\InputProcessor\SystemPromptInputProcessor` | 注入招聘助手系统提示词 |
| **嵌套 DTO** | `InterviewGuide` → `InterviewQuestion` | StructuredOutput 递归解析嵌套面试指南 |

---

## 下一步

如果你想把 YouTube 视频变成可搜索的知识库，请看 [26-youtube-video-knowledge-base.md](./26-youtube-video-knowledge-base.md)。

