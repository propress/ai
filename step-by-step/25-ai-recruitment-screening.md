# AI 智能招聘筛选助手

## 业务场景

你是一家快速增长公司的 HR 负责人。一个热门岗位收到了 500 份简历，需要快速筛选出 Top 20 进入面试。传统做法：HR 逐份看简历，至少要 2-3 天。现在用 AI 自动解析简历、提取关键信息、按岗位要求打分排序，还能为每位候选人自动生成面试问题。

**典型应用：** 简历自动解析与打分、候选人匹配排序、面试问题生成、人才库构建与检索

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **StructuredOutput** | 简历信息结构化提取 |
| **Store** | 人才库向量存储，候选人相似度匹配 |
| **Agent** | 编排筛选与面试准备流程 |
| **Document Content** | PDF 简历文件处理 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-store
```

---

## Step 1：定义简历结构

```php
<?php

namespace App\Dto;

final class WorkExperience
{
    /**
     * @param string $company    公司名称
     * @param string $title      职位
     * @param string $duration   工作时长（如 "2 年 3 个月"）
     * @param string $highlights 工作亮点
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
     * @param int              $yearsExperience 工作年限
     * @param string           $education       最高学历（bachelor/master/phd/other）
     * @param string           $educationSchool 毕业院校
     * @param string           $educationMajor  专业
     * @param string[]         $skills          技能列表
     * @param WorkExperience[] $experience      工作经历
     * @param string[]         $certifications  证书/认证
     * @param string           $summary         个人总结
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

---

## Step 2：AI 解析简历

```php
<?php

require 'vendor/autoload.php';

use App\Dto\ResumeProfile;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 解析 PDF 简历
$messages = new MessageBag(
    Message::forSystem(
        '你是专业的 HR 简历分析师。从简历中精准提取所有关键信息。'
        . '技能要细分到具体技术栈（不要只写"编程"，要写"PHP, Python, MySQL"）。'
    ),
    Message::ofUser(
        '请解析这份简历：',
        Document::fromFile('/path/to/resume.pdf'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => ResumeProfile::class,
]);

$resume = $result->asObject();

echo "=== 简历解析 ===\n";
echo "姓名：{$resume->name}\n";
echo "学历：{$resume->education}（{$resume->educationSchool} - {$resume->educationMajor}）\n";
echo "经验：{$resume->yearsExperience} 年\n";
echo "技能：" . implode(', ', $resume->skills) . "\n";
echo "经历：\n";
foreach ($resume->experience as $exp) {
    echo "  📌 {$exp->company} | {$exp->title} | {$exp->duration}\n";
    echo "     {$exp->highlights}\n";
}
```

---

## Step 3：岗位匹配打分

```php
<?php

namespace App\Dto;

final class CandidateScore
{
    /**
     * @param int    $skillMatch       技能匹配度（0-100）
     * @param int    $experienceMatch  经验匹配度（0-100）
     * @param int    $educationMatch   学历匹配度（0-100）
     * @param int    $overallScore     综合评分（0-100）
     * @param string $tier             候选人等级（A/B/C/D）
     * @param string $strengths        优势
     * @param string $concerns         顾虑
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
        '你是资深的技术招聘专家。根据岗位要求，对候选人的简历进行客观评分。'
        . '注意：只评估与岗位要求相关的技能和经验，不要因无关经历加分或扣分。'
    ),
    Message::ofUser(
        "岗位要求：\n{$jobRequirements}\n\n"
        . "候选人信息：\n"
        . "姓名：{$resume->name}\n"
        . "经验：{$resume->yearsExperience} 年\n"
        . "学历：{$resume->education} - {$resume->educationSchool}\n"
        . "技能：" . implode(', ', $resume->skills) . "\n"
        . "经历摘要：{$resume->summary}\n"
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => CandidateScore::class,
]);

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

---

## Step 4：人才库向量存储

```php
<?php

use Symfony\AI\Store\Document\Document as StoreDocument;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Indexer;

// 将候选人存入人才库
$document = new StoreDocument(
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
    ]),
);

$indexer->index([$document]);
echo "✅ 候选人已入人才库\n";
```

---

## Step 5：智能人才匹配检索

当有新岗位开放时，从人才库中找到最匹配的候选人。

```php
<?php

use Symfony\AI\Store\Query\VectorQuery;

// 新岗位：前端 + 后端全栈工程师
$query = new VectorQuery(
    'React TypeScript Node.js PHP 全栈开发 3年以上经验',
    maxResults: 10,
);

$candidates = $retriever->retrieve($query);

echo "=== 全栈岗位匹配候选人 ===\n";
foreach ($candidates as $i => $candidate) {
    $meta = $candidate->metadata;
    echo ($i + 1) . ". {$meta['name']} | {$meta['years']}年 | "
        . "{$meta['education']} | 评分：{$meta['score']} | {$meta['tier']}级\n";
}
```

---

## Step 6：自动生成面试问题

```php
<?php

namespace App\Dto;

final class InterviewQuestion
{
    /**
     * @param string $question  问题
     * @param string $type      类型（technical/behavioral/situational）
     * @param string $intent    考察意图
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
     * @param InterviewQuestion[] $questions       面试问题列表
     * @param string[]            $focusAreas      重点考察领域
     * @param string[]            $redFlags        注意事项/风险点
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
        . '问题应覆盖：技术深度、项目经验验证、解决问题能力、团队协作。'
        . '根据候选人的"顾虑点"设计验证性问题。'
    ),
    Message::ofUser(
        "岗位：高级 PHP 后端工程师\n"
        . "候选人：{$resume->name}（{$resume->yearsExperience}年经验）\n"
        . "技能：" . implode(', ', $resume->skills) . "\n"
        . "优势：{$score->strengths}\n"
        . "顾虑：{$score->concerns}\n\n"
        . '请设计 6 个面试问题。'
    ),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => InterviewGuide::class,
]);

$guide = $result->asObject();

echo "=== 面试指南：{$resume->name} ===\n\n";
echo "🎯 重点考察：" . implode('、', $guide->focusAreas) . "\n\n";

foreach ($guide->questions as $i => $q) {
    echo "Q" . ($i + 1) . " [{$q->type}]：{$q->question}\n";
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

## 完整流程

```
简历（PDF/文本）
    │
    ▼
[AI 解析] → ResumeProfile 结构化数据
    │
    ▼
[岗位匹配打分] → CandidateScore（A/B/C/D）
    │
    ├──► [向量化存入人才库] → Store
    │
    ├──► [批量排序] → Top N 候选人
    │
    └──► [生成面试指南] → InterviewGuide
              ├─ 定制面试题
              ├─ 考察重点
              └─ 风险提醒
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Document::fromFile()` | 直接读取 PDF 简历 |
| 嵌套 DTO | `ResumeProfile` 包含 `WorkExperience[]` |
| 多级评分 | `CandidateScore` 从多维度量化评估 |
| 人才库 | 向量存储，支持语义检索匹配候选人 |
| 面试指南 | `InterviewGuide` 根据候选人弱点定制问题 |

## 下一步

如果你想把 YouTube 视频变成可搜索的知识库，请看 [26-youtube-video-knowledge-base.md](./26-youtube-video-knowledge-base.md)。
