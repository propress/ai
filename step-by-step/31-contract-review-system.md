# 智能合同审查系统

## 业务场景

你是一家企业的法务部门。每个月要审查几十份合同：供应商合同、客户合同、保密协议、租赁合同等。法务人员需要逐条检查：条款是否完整、风险条款是否存在、关键日期和金额是否正确。现在用 AI 自动提取合同关键信息、识别风险条款、对比公司标准模板、给出修改建议。

**典型应用：** 合同条款审查、风险条款识别、合规检查、合同信息提取、供应商协议审核

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Document Content** | PDF 合同文件输入 |
| **StructuredOutput** | 提取合同关键信息为结构化对象 |
| **Agent** | 编排多阶段审查流程 |
| **Store** | 存储历史合同条款，支持对比检索 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai  # 或 anthropic
composer require symfony/ai-agent
composer require symfony/ai-store
```

---

## Step 1：定义合同信息结构

```php
<?php

namespace App\Dto;

final class ContractParty
{
    /**
     * @param string $name    当事方名称
     * @param string $role    角色（甲方/乙方/丙方）
     * @param string $address 地址
     * @param string $contact 联系人
     */
    public function __construct(
        public readonly string $name,
        public readonly string $role,
        public readonly string $address,
        public readonly string $contact,
    ) {
    }
}

final class ContractInfo
{
    /**
     * @param string          $contractType    合同类型（sales/service/nda/lease/employment/other）
     * @param ContractParty[] $parties         当事方
     * @param string          $effectiveDate   生效日期
     * @param string          $expirationDate  到期日期
     * @param string          $totalAmount     合同总金额
     * @param string          $paymentTerms    付款条件
     * @param string          $terminationClause 终止条款摘要
     * @param string          $governingLaw    适用法律
     * @param string[]        $keyObligations  关键义务
     */
    public function __construct(
        public readonly string $contractType,
        public readonly array $parties,
        public readonly string $effectiveDate,
        public readonly string $expirationDate,
        public readonly string $totalAmount,
        public readonly string $paymentTerms,
        public readonly string $terminationClause,
        public readonly string $governingLaw,
        public readonly array $keyObligations,
    ) {
    }
}
```

---

## Step 2：合同信息提取

```php
<?php

require 'vendor/autoload.php';

use App\Dto\ContractInfo;
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

// 上传合同 PDF
$messages = new MessageBag(
    Message::forSystem(
        '你是资深法务专家。从合同文件中精确提取所有关键信息。'
        . '金额要包含币种，日期使用 YYYY-MM-DD 格式。'
        . '如果某项信息在合同中未找到，标注为"未约定"。'
    ),
    Message::ofUser(
        '请提取以下合同的关键信息：',
        Document::fromFile('/path/to/contract.pdf'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => ContractInfo::class,
]);

$contract = $result->asObject();

echo "=== 合同信息提取 ===\n";
echo "类型：{$contract->contractType}\n";
echo "金额：{$contract->totalAmount}\n";
echo "有效期：{$contract->effectiveDate} ~ {$contract->expirationDate}\n";
echo "付款条件：{$contract->paymentTerms}\n";
echo "适用法律：{$contract->governingLaw}\n\n";

echo "当事方：\n";
foreach ($contract->parties as $party) {
    echo "  {$party->role}：{$party->name}\n";
}

echo "\n关键义务：\n";
foreach ($contract->keyObligations as $obligation) {
    echo "  📌 {$obligation}\n";
}
```

---

## Step 3：风险条款审查

```php
<?php

namespace App\Dto;

final class RiskClause
{
    /**
     * @param string $clause      条款内容
     * @param string $riskLevel   风险等级（critical/high/medium/low）
     * @param string $riskType    风险类型（financial/legal/operational/compliance）
     * @param string $explanation 风险说明
     * @param string $suggestion  修改建议
     */
    public function __construct(
        public readonly string $clause,
        public readonly string $riskLevel,
        public readonly string $riskType,
        public readonly string $explanation,
        public readonly string $suggestion,
    ) {
    }
}

final class ContractReview
{
    /**
     * @param string       $overallRisk       整体风险（high/medium/low）
     * @param RiskClause[] $riskClauses       风险条款列表
     * @param string[]     $missingClauses    缺失条款
     * @param string[]     $positivePoints    有利条款
     * @param string       $legalOpinion      法律意见摘要
     * @param bool         $recommendApproval 是否建议签署
     */
    public function __construct(
        public readonly string $overallRisk,
        public readonly array $riskClauses,
        public readonly array $missingClauses,
        public readonly array $positivePoints,
        public readonly string $legalOpinion,
        public readonly bool $recommendApproval,
    ) {
    }
}
```

```php
<?php

use App\Dto\ContractReview;

$messages = new MessageBag(
    Message::forSystem(
        "你是企业法务审查专家。站在我方（甲方）立场审查合同。重点关注：\n"
        . "1. 无限责任/连带责任条款\n"
        . "2. 不合理的违约金比例\n"
        . "3. 模糊的验收标准\n"
        . "4. 缺少知识产权归属\n"
        . "5. 缺少保密条款\n"
        . "6. 不利的争议解决方式\n"
        . "7. 自动续约/难以终止的条款\n"
        . "8. 缺少不可抗力条款\n"
    ),
    Message::ofUser(
        '请审查以下合同并识别风险条款：',
        Document::fromFile('/path/to/contract.pdf'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => ContractReview::class,
]);

$review = $result->asObject();

$riskIcon = match ($review->overallRisk) {
    'high' => '🔴',
    'medium' => '🟡',
    'low' => '🟢',
    default => '⚪',
};

echo "{$riskIcon} 整体风险：{$review->overallRisk}\n";
echo "建议签署：" . ($review->recommendApproval ? '✅ 是' : '❌ 否') . "\n\n";

// 风险条款
if ([] !== $review->riskClauses) {
    echo "⚠️ 风险条款：\n";
    foreach ($review->riskClauses as $risk) {
        $icon = match ($risk->riskLevel) {
            'critical' => '🔴',
            'high' => '🟠',
            'medium' => '🟡',
            'low' => '🟢',
            default => '⚪',
        };
        echo "  {$icon} [{$risk->riskType}] {$risk->explanation}\n";
        echo "     原文：{$risk->clause}\n";
        echo "     建议：{$risk->suggestion}\n\n";
    }
}

// 缺失条款
if ([] !== $review->missingClauses) {
    echo "❗ 缺失条款：\n";
    foreach ($review->missingClauses as $missing) {
        echo "  ⛔ {$missing}\n";
    }
}

// 有利条款
if ([] !== $review->positivePoints) {
    echo "\n✅ 有利条款：\n";
    foreach ($review->positivePoints as $positive) {
        echo "  ✔ {$positive}\n";
    }
}

echo "\n📋 法律意见：\n{$review->legalOpinion}\n";
```

---

## Step 4：合同模板对比

将合同与公司标准模板对比，找出偏差。

```php
<?php

namespace App\Dto;

final class TemplateDeviation
{
    /**
     * @param string $clause         条款名称
     * @param string $templateVersion 模板版本
     * @param string $contractVersion 合同版本
     * @param string $deviationType  偏差类型（missing/modified/added/weaker）
     * @param string $impact         影响说明
     */
    public function __construct(
        public readonly string $clause,
        public readonly string $templateVersion,
        public readonly string $contractVersion,
        public readonly string $deviationType,
        public readonly string $impact,
    ) {
    }
}
```

```php
<?php

// 公司标准合同模板要点
$standardTemplate = <<<'TEMPLATE'
我方标准合同模板要求：
1. 违约金不超过合同总额的 20%
2. 知识产权归我方所有
3. 保密期限至少 3 年
4. 验收周期不超过 15 个工作日
5. 争议解决方式为我方所在地仲裁
6. 不可抗力条款必须包含
7. 付款在验收后 30 日内
8. 终止通知期至少 30 天
TEMPLATE;

$messages = new MessageBag(
    Message::forSystem('对比合同与公司标准模板，找出所有偏差。'),
    Message::ofUser(
        "公司标准模板：\n{$standardTemplate}\n\n"
        . "合同关键条款：\n"
        . "金额：{$contract->totalAmount}\n"
        . "付款：{$contract->paymentTerms}\n"
        . "终止：{$contract->terminationClause}\n"
        . "义务：" . implode('; ', $contract->keyObligations)
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
echo "=== 模板偏差分析 ===\n" . $result->asText() . "\n";
```

---

## Step 5：合同知识库

积累已审查的合同条款，辅助未来审查。

```php
<?php

use Symfony\AI\Store\Document\Document as StoreDocument;
use Symfony\AI\Store\Document\Metadata;

// 将风险条款存入知识库
foreach ($review->riskClauses as $risk) {
    $doc = new StoreDocument(
        id: 'risk-' . md5($risk->clause),
        content: "风险条款：{$risk->clause}\n说明：{$risk->explanation}\n建议：{$risk->suggestion}",
        metadata: new Metadata([
            'risk_level' => $risk->riskLevel,
            'risk_type' => $risk->riskType,
            'contract_type' => $contract->contractType,
            'date' => date('Y-m-d'),
        ]),
    );
    $indexer->index([$doc]);
}

echo "✅ 风险条款已入库，可辅助未来审查\n";

// 未来审查新合同时，检索历史类似风险
// $query = new VectorQuery('违约金条款', maxResults: 5);
// $historicalRisks = $retriever->retrieve($query);
```

---

## 完整流程

```
合同 PDF
    │
    ▼
[AI 信息提取] → ContractInfo
    │             ├─ 当事方
    │             ├─ 金额 & 日期
    │             └─ 关键义务
    ▼
[AI 风险审查] → ContractReview
    │             ├─ 风险条款
    │             ├─ 缺失条款
    │             └─ 签署建议
    ▼
[模板对比] → 偏差分析
    │
    ├──► [存入知识库] → 积累审查经验
    │
    └──► 输出审查报告 → 法务确认
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Document::fromFile()` | 直接输入 PDF 合同文件 |
| 嵌套 DTO | `ContractInfo` 含 `ContractParty[]` |
| 风险评估 | `ContractReview` 含 `RiskClause[]` |
| 模板对比 | 合同 vs 标准模板偏差分析 |
| 知识库积累 | 历史风险条款存入 Store |

## 下一步

如果你想用 AI 构建电商智能推荐引擎，请看 [32-ecommerce-recommendation-engine.md](./32-ecommerce-recommendation-engine.md)。
