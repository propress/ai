# 电商智能推荐引擎

## 业务场景

你在做一个电商平台。用户浏览商品时，需要展示"猜你喜欢""相似商品""搭配推荐"。传统推荐系统需要大量用户行为数据和复杂算法。现在用 AI 嵌入 + 向量相似度快速构建推荐引擎：把商品描述向量化，用语义相似度找到相关商品，结合 HybridQuery 混合搜索提升准确度。

**典型应用：** 商品推荐、内容推荐、课程推荐、人才匹配、音乐/电影推荐

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台（嵌入模型） |
| **Store** | 商品向量存储与检索 |
| **Store HybridQuery** | 混合搜索（向量语义 + 关键词） |
| **StructuredOutput** | 结构化推荐结果 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-store symfony/ai-store-meilisearch  # 支持混合搜索
```

---

## Step 1：商品数据向量化

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\Meilisearch\Store;
use Symfony\AI\Store\Document\Document;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Indexer;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$store = new Store(/* Meilisearch 配置 */);
$indexer = new Indexer($platform, 'text-embedding-3-small', $store);

// 商品数据
$products = [
    [
        'id' => 'prod-001',
        'name' => 'MacBook Pro 14 寸 M3 芯片',
        'category' => '电脑',
        'price' => 14999,
        'description' => '搭载 M3 Pro 芯片，18GB 内存，512GB 存储，14.2 英寸 Liquid Retina XDR 显示屏，适合专业创作者和开发者。',
        'tags' => ['笔记本', '苹果', '高性能', '程序员'],
    ],
    [
        'id' => 'prod-002',
        'name' => 'Sony WH-1000XM5 降噪耳机',
        'category' => '音频',
        'price' => 2499,
        'description' => '行业领先的主动降噪，30 小时续航，轻量化设计，支持多设备连接。适合通勤和办公降噪。',
        'tags' => ['耳机', '降噪', '无线', '办公'],
    ],
    [
        'id' => 'prod-003',
        'name' => 'HHKB Professional HYBRID Type-S',
        'category' => '外设',
        'price' => 2699,
        'description' => '静电容轴体，60% 紧凑键盘，蓝牙 + USB-C，静音版。程序员和作家的终极键盘。',
        'tags' => ['键盘', '机械', '程序员', '静音'],
    ],
    [
        'id' => 'prod-004',
        'name' => 'Dell U2723QE 4K 显示器',
        'category' => '显示器',
        'price' => 3999,
        'description' => '27 英寸 4K IPS Black 面板，USB-C 90W 供电，菊花链连接。色准 Delta E<2，设计师和开发者首选。',
        'tags' => ['显示器', '4K', 'USB-C', '设计'],
    ],
    [
        'id' => 'prod-005',
        'name' => 'Logitech MX Master 3S 鼠标',
        'category' => '外设',
        'price' => 799,
        'description' => '人体工学设计，MagSpeed 电磁滚轮，多设备切换，USB-C 充电。办公效率神器。',
        'tags' => ['鼠标', '办公', '人体工学', '无线'],
    ],
    [
        'id' => 'prod-006',
        'name' => 'ThinkPad X1 Carbon Gen 11',
        'category' => '电脑',
        'price' => 11999,
        'description' => 'Intel 13 代酷睿，14 英寸 2.8K OLED，1.12kg 超轻，ThinkPad 经典键盘。商务笔记本标杆。',
        'tags' => ['笔记本', '联想', '轻薄', '商务'],
    ],
    [
        'id' => 'prod-007',
        'name' => 'AirPods Pro 2',
        'category' => '音频',
        'price' => 1899,
        'description' => 'H2 芯片主动降噪，自适应通透模式，个性化空间音频，IP54 防尘防水。',
        'tags' => ['耳机', '苹果', '降噪', '入耳式'],
    ],
    [
        'id' => 'prod-008',
        'name' => 'Keychron K3 Pro 矮轴键盘',
        'category' => '外设',
        'price' => 599,
        'description' => '75% 布局矮轴键盘，Gateron 矮轴，RGB 背光，Mac/Win 双系统兼容，蓝牙有线双模。',
        'tags' => ['键盘', '矮轴', '蓝牙', '便携'],
    ],
];

// 向量化并索引
foreach ($products as $product) {
    $doc = new Document(
        id: $product['id'],
        content: "{$product['name']}：{$product['description']}",
        metadata: new Metadata([
            'name' => $product['name'],
            'category' => $product['category'],
            'price' => $product['price'],
            'tags' => implode(',', $product['tags']),
        ]),
    );
    $indexer->index([$doc]);
}

echo "✅ 已索引 " . count($products) . " 个商品\n";
```

---

## Step 2：语义相似推荐

```php
<?php

use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Retriever;

$retriever = new Retriever($platform, 'text-embedding-3-small', $store);

// 场景 1：用户正在看 MacBook，推荐相似/搭配商品
$query = new VectorQuery(
    'MacBook Pro 程序员笔记本 高性能 开发',
    maxResults: 5,
);

$results = $retriever->retrieve($query);

echo "=== 看了 MacBook Pro 的人还看了 ===\n";
foreach ($results as $result) {
    $meta = $result->metadata;
    echo "  📦 {$meta['name']}  ¥{$meta['price']}\n";
}
```

---

## Step 3：混合搜索（HybridQuery）

结合向量语义搜索和关键词过滤，提高精准度。

```php
<?php

use Symfony\AI\Store\Query\HybridQuery;

// 用户搜索"程序员用的键盘"
// HybridQuery = 语义搜索 + 关键词搜索的融合
$query = new HybridQuery(
    text: '适合程序员使用的静音键盘',
    vectorWeight: 0.7,  // 语义权重
    textWeight: 0.3,    // 关键词权重
    maxResults: 5,
);

$results = $retriever->retrieve($query);

echo "=== 混合搜索：程序员键盘 ===\n";
foreach ($results as $result) {
    $meta = $result->metadata;
    echo "  ⌨️ {$meta['name']}  ¥{$meta['price']}  [{$meta['category']}]\n";
}
```

---

## Step 4：个性化推荐

根据用户的浏览和购买历史生成推荐。

```php
<?php

namespace App\Dto;

final class ProductRecommendation
{
    /**
     * @param string $productName  商品名称
     * @param string $reason       推荐理由
     * @param string $matchType    匹配类型（similar/complementary/upgrade/popular）
     * @param float  $confidence   推荐置信度（0.0-1.0）
     */
    public function __construct(
        public readonly string $productName,
        public readonly string $reason,
        public readonly string $matchType,
        public readonly float $confidence,
    ) {
    }
}

final class RecommendationResult
{
    /**
     * @param ProductRecommendation[] $recommendations  推荐列表
     * @param string                  $userProfile      用户画像描述
     */
    public function __construct(
        public readonly array $recommendations,
        public readonly string $userProfile,
    ) {
    }
}
```

```php
<?php

use App\Dto\RecommendationResult;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$structuredPlatform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 用户行为数据
$userHistory = [
    '浏览：MacBook Pro 14 寸',
    '购买：HHKB Professional 键盘',
    '收藏：Dell 4K 显示器',
    '搜索：程序员桌面搭建',
];

// 候选商品（从向量检索中获取）
$candidates = "可推荐商品：\n";
foreach ($products as $p) {
    $candidates .= "- {$p['name']}（¥{$p['price']}，{$p['category']}）\n";
}

$messages = new MessageBag(
    Message::forSystem(
        '你是电商推荐系统。根据用户的浏览、购买、收藏、搜索行为，推荐最合适的商品。'
        . "\n推荐类型：similar（相似）、complementary（搭配）、upgrade（升级）、popular（热门）"
    ),
    Message::ofUser(
        "用户行为：\n" . implode("\n", $userHistory) . "\n\n"
        . $candidates . "\n"
        . '请推荐 3-5 个最合适的商品。'
    ),
);

$result = $structuredPlatform->invoke('gpt-4o-mini', $messages, [
    'response_format' => RecommendationResult::class,
]);

$rec = $result->asObject();

echo "=== 个性化推荐 ===\n";
echo "👤 用户画像：{$rec->userProfile}\n\n";

foreach ($rec->recommendations as $i => $item) {
    $icon = match ($item->matchType) {
        'similar' => '🔄',
        'complementary' => '🤝',
        'upgrade' => '⬆️',
        'popular' => '🔥',
        default => '📦',
    };
    echo ($i + 1) . ". {$icon} {$item->productName}\n";
    echo "   理由：{$item->reason}\n";
    echo "   置信度：" . round($item->confidence * 100) . "%\n\n";
}
```

---

## Step 5：搭配推荐（"一起购买"）

```php
<?php

// 用户加入购物车：MacBook Pro
$currentProduct = 'MacBook Pro 14 寸 M3 芯片，程序员笔记本';

// 用向量搜索找到互补商品
$query = new VectorQuery(
    "与 {$currentProduct} 搭配使用的外设和配件",
    maxResults: 5,
);

$complements = $retriever->retrieve($query);

echo "=== 搭配购买 ===\n";
echo "📦 主商品：MacBook Pro 14 寸\n\n";
echo "🤝 推荐搭配：\n";
foreach ($complements as $item) {
    $meta = $item->metadata;
    if ('电脑' !== $meta['category']) {  // 排除同类
        echo "  + {$meta['name']}  ¥{$meta['price']}\n";
    }
}
```

---

## 完整流程

```
商品数据
    │
    ▼
[嵌入模型] → 向量化
    │
    ▼
[向量存储] → Meilisearch / Pinecone / PostgreSQL
    │
    ├──► [相似推荐] → VectorQuery（语义匹配）
    │
    ├──► [搜索推荐] → HybridQuery（语义 + 关键词）
    │
    ├──► [个性化推荐] → 用户画像 + AI 推理
    │
    └──► [搭配推荐] → 互补商品语义检索
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `text-embedding-3-small` | OpenAI 嵌入模型（向量化） |
| `Indexer` | 商品描述嵌入 + 索引 |
| `VectorQuery` | 纯语义相似度检索 |
| `HybridQuery` | 混合搜索（语义 + 关键词） |
| `vectorWeight` / `textWeight` | 控制语义和关键词的权重 |
| 个性化推荐 | 用户行为 + AI 推理 + 结构化输出 |
| 搭配推荐 | 互补商品语义检索 |

## 下一步

如果你想构建实时流式对话应用（打字机效果），请看 [33-realtime-streaming-chat.md](./33-realtime-streaming-chat.md)。
