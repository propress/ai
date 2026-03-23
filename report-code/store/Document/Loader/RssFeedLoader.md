# RssFeedLoader 源码分析报告

## 基本信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `src/store/src/Document/Loader/RssFeedLoader.php` |
| **命名空间** | `Symfony\AI\Store\Document\Loader` |
| **类声明** | `final class RssFeedLoader implements LoaderInterface` |
| **作者** | Niklas Grießer \<niklas@griesser.me\> |
| **职责** | 从 RSS 订阅源加载文档，每个 RSS 条目转化为一个 TextDocument |

---

## 类常量

| 常量 | 值 | 用途 |
|------|----|------|
| `OPTION_UUID_NAMESPACE` | `'uuid_namespace'` | 运行时覆盖 UUID v5 命名空间 |

---

## 构造函数

```php
public function __construct(
    private readonly HttpClientInterface $httpClient,
    private readonly string $uuidNamespace = '6ba7b810-9dad-11d1-80b4-00c04fd430c8',
)
```

### 参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$httpClient` | `HttpClientInterface` | （必需） | Symfony HTTP 客户端，用于获取 RSS 订阅内容 |
| `$uuidNamespace` | `string` | `'6ba7b810-...'` | UUID v5 的命名空间，默认为 RFC 4122 定义的 URL 命名空间 |

### 设计要点

- `HttpClientInterface` 通过依赖注入传入，而非内部创建，便于测试时使用 MockHttpClient
- 默认 UUID 命名空间为 RFC 4122 标准 URL 命名空间（`6ba7b810-9dad-11d1-80b4-00c04fd430c8`），适合基于 URL 生成确定性 ID

---

## 方法签名与行为

### `load(?string $source = null, array $options = []): iterable`

**输入：**
- `$source`：RSS 订阅源 URL（不可为 `null`）
- `$options`：`array{uuid_namespace?: string}` — 可选的 UUID 命名空间覆盖

**输出：**
- `iterable<TextDocument>`：每个 RSS 条目对应一个文档

**异常：**
- `RuntimeException`：`symfony/dom-crawler` 未安装
- `InvalidArgumentException`：`$source` 为 `null`
- `RuntimeException`：HTTP 请求失败

**核心流程：**

```
1. 检查 symfony/dom-crawler 依赖
2. 验证 source 非空
3. 解析 UUID 命名空间（支持运行时覆盖）
4. 通过 HttpClient 发起 GET 请求获取 RSS XML
5. 使用 DomCrawler 解析 XML
6. 遍历所有 rss/channel/item 节点
7. 对每个条目：
   a. 提取 guid、link、title、pubDate、description
   b. 提取可选字段：dc:creator（作者）、content:encoded（全文）
   c. 生成文档 ID（UUID 策略见下文）
   d. 创建 RssItem 值对象
   e. yield TextDocument（内容为 RssItem.toString()，元数据包含所有字段）
```

### ID 生成策略（三级判断）

```php
$guid = $node->filterXpath('node()/guid')->count() > 0
    ? $node->filterXpath('node()/guid')->text()
    : null;
$link = $node->filterXpath('node()/link')->text();

$id = null !== $guid && Uuid::isValid($guid)
    ? Uuid::fromString($guid)       // 1. guid 是有效 UUID → 直接使用
    : Uuid::v5($uuidNamespace, $guid ?? $link);  // 2. 否则使用 UUID v5
```

| 条件 | ID 生成方式 | 确定性 |
|------|------------|--------|
| `guid` 存在且为有效 UUID | 直接使用 `guid` 值 | ✅ 确定性 |
| `guid` 存在但非 UUID | UUID v5（命名空间 + guid） | ✅ 确定性 |
| `guid` 不存在 | UUID v5（命名空间 + link） | ✅ 确定性 |

**关键设计：** 所有 ID 生成策略都是**确定性**的——对同一 RSS 条目，无论加载多少次都产出相同的文档 ID。这对于增量更新和去重至关重要。

---

## RSS 条目字段提取

| 字段 | XPath | 必需 | 说明 |
|------|-------|------|------|
| `guid` | `node()/guid` | 否 | 全局唯一标识符 |
| `link` | `node()/link` | 是 | 条目链接 |
| `title` | `//title` | 是 | 条目标题 |
| `pubDate` | `//pubDate` | 是 | 发布日期 |
| `description` | `//description` | 是 | 摘要描述 |
| `dc:creator` | `node()/dc:creator` | 否 | 作者（Dublin Core 命名空间） |
| `content:encoded` | `node()/content:encoded` | 否 | 全文内容（Content 命名空间） |

---

## HTTP 请求配置

```php
$xml = $this->httpClient->request('GET', $source, [
    'headers' => [
        'Accept' => 'application/rss+xml,application/xml,text/xml',
    ],
])->getContent();
```

- 使用标准 RSS MIME 类型组合请求
- 异常通过 `ExceptionInterface` 统一捕获并转换为内部 `RuntimeException`

---

## 设计模式

### 1. 值对象辅助（Value Object Helper）
使用 `RssItem` 值对象结构化 RSS 条目数据，提供 `toString()` 和 `toArray()` 方法，实现了数据表示与文档创建的分离。

### 2. 确定性 ID 生成（Deterministic ID Generation）
利用 UUID v5 基于命名空间和名称生成确定性 ID，确保相同的 RSS 条目始终映射到相同的文档 ID。

### 3. 依赖注入（Dependency Injection）
`HttpClientInterface` 通过构造函数注入，遵循控制反转原则：
- 生产环境使用真实 HTTP 客户端
- 测试环境使用 `MockHttpClient`

---

## 使用场景

| 场景 | 说明 |
|------|------|
| **新闻聚合** | 从多个 RSS 源收集新闻文章 |
| **博客监控** | 追踪技术博客更新并建立知识库 |
| **内容索引** | 为 RSS 内容建立向量搜索索引 |
| **定期同步** | 确定性 ID 支持增量更新（避免重复导入） |

### 典型用法示例

```php
use Symfony\Component\HttpClient\HttpClient;

$loader = new RssFeedLoader(HttpClient::create());

// 加载 RSS 源
foreach ($loader->load('https://example.com/feed.rss') as $doc) {
    echo $doc->getId();                        // 确定性 UUID
    echo $doc->getContent();                   // 格式化的条目文本
    echo $doc->getMetadata()['title'];         // 条目标题
    echo $doc->getMetadata()['date'];          // 发布日期
    echo $doc->getMetadata()['link'];          // 原文链接
}

// 自定义 UUID 命名空间
$loader = new RssFeedLoader(
    HttpClient::create(),
    uuidNamespace: 'a1b2c3d4-e5f6-7890-abcd-ef1234567890',
);
```

---

## 外部依赖

| 依赖 | 类/接口 | 用途 | 必需？ |
|------|---------|------|--------|
| `symfony/dom-crawler` | `Symfony\Component\DomCrawler\Crawler` | XML/HTML DOM 解析 | ✅ |
| `symfony/http-client` | `Symfony\Contracts\HttpClient\HttpClientInterface` | HTTP 请求 | ✅（构造注入） |
| `symfony/uid` | `Symfony\Component\Uid\Uuid` | UUID 生成（v5、验证） | ✅ |
| 内部 | `Rss\RssItem` | RSS 条目值对象 | — |
| 内部 | `TextDocument` | 产出的文档类型 | — |
| 内部 | `Metadata` | 文档元数据容器 | — |
| 内部 | `LoaderInterface` | 实现的接口 | — |

---

## 可扩展点

1. **UUID 命名空间自定义**：通过构造函数参数或运行时选项调整 ID 生成行为
2. **HttpClient 装饰**：通过装饰 HttpClient 添加缓存、重试、认证等能力
3. **RssItem 格式化**：`RssItem.toString()` 的输出格式可在值对象中修改

### 潜在扩展方向
- 支持 Atom 订阅格式
- 条目过滤（按日期、关键词）
- 内容清理（HTML → 纯文本）
- 分页/增量加载（基于日期的增量获取）

---

## 技巧和设计理由

### 为什么使用 UUID v5 而非 UUID v4？
UUID v4 是随机的——每次加载相同 RSS 条目会产生不同 ID。UUID v5 是确定性的，基于命名空间和名称（guid 或 link）计算，相同输入始终产生相同输出。这对于向量数据库的更新和去重至关重要。

### 为什么默认命名空间是 `6ba7b810-...`？
这是 RFC 4122 定义的标准 URL 命名空间 UUID。由于 RSS 条目通常以 URL 作为标识，使用 URL 命名空间在语义上最为合适。

### 为什么先检查 guid 是否为有效 UUID？
某些 RSS 源的 `<guid>` 已经是 UUID 格式（如由内容管理系统生成）。直接使用这些 UUID 可以保持与原始系统的 ID 一致性。

### `node()/` vs `//` XPath 前缀的区别
- `node()/guid`：仅搜索当前节点的直接子节点
- `//title`：搜索当前节点下所有层级的 `<title>` 元素

使用 `node()/` 处理可能存在命名空间的元素（如 `dc:creator`、`content:encoded`），而 `//` 用于标准 RSS 元素。

### 异常转换
HTTP 异常（`ExceptionInterface`）被包装为内部 `RuntimeException`，包含源 URL 和原始错误信息，便于调试同时保持异常类型一致性。
