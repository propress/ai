# Toolbox/Source/Source.php 分析报告

## 概述

`Source` 是一个不可变值对象，表示工具执行过程中引用的外部信息来源，包含名称、引用地址（URL 或标识符）和内容摘要，用于向用户或下游系统提供结果的溯源信息。

## 关键内容

```php
final class Source
{
    public function __construct(
        private readonly string $name,       // 来源名称（如文章标题）
        private readonly string $reference,  // 引用地址（如 URL）
        private readonly string $content,    // 内容摘要
    ) {}
}
```

## 典型使用示例

- `Brave`：`new Source($result['title'], $result['url'], $result['description'])`
- `Wikipedia`：`new Source($article['title'], $wikiUrl, $article['extract'])`
- `Clock`：`new Source('Current Time', 'Clock', $now->format('Y-m-d H:i:s'))`
- `Scraper`：`new Source($title, $url, $content)`

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `SourceCollection` | 持有 `Source` 对象的集合 |
| `HasSourcesInterface` / `HasSourcesTrait` | 工具通过此接口的 `addSource()` 方法添加来源 |
| `AgentProcessor` | 在 `includeSources` 模式下将来源集合附加到最终结果元数据 |
| Bridge 工具 | Brave、Wikipedia、SerpApi、Tavily 等均在执行时创建 `Source` 对象 |
