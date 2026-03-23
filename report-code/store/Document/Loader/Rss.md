# Rss 子目录概览

## 目录信息

| 属性 | 值 |
|------|-----|
| **目录路径** | `src/store/src/Document/Loader/Rss/` |
| **命名空间** | `Symfony\AI\Store\Document\Loader\Rss` |
| **包含文件** | `RssItem.php` |
| **职责** | 为 RSS 订阅加载功能提供辅助值对象 |

---

## 目录结构

```
src/store/src/Document/Loader/Rss/
└── RssItem.php      # RSS 条目值对象
```

---

## 文件说明

### RssItem.php

| 属性 | 值 |
|------|-----|
| **类** | `final class RssItem` |
| **模式** | 值对象（Value Object） |
| **作者** | Niklas Grießer |
| **详细报告** | [RssItem.md](Rss/RssItem.md) |

`RssItem` 是一个不可变的值对象，封装 RSS 订阅条目的所有字段：

- `$id`（Uuid）— 条目唯一标识
- `$title`（string）— 标题
- `$link`（string）— 原文链接
- `$date`（\DateTimeImmutable）— 发布日期
- `$description`（string）— 摘要描述
- `$author`（?string）— 作者（可选）
- `$content`（?string）— 完整内容（可选）

提供两种数据输出方式：
- `toString()`：人类可读的格式化文本，用作 `TextDocument` 的内容
- `toArray()`：结构化数组，用作 `TextDocument` 的元数据

---

## 与 RssFeedLoader 的关系

```
RssFeedLoader (Loader/)
    │
    ├── 依赖 HttpClientInterface (获取 RSS XML)
    ├── 依赖 DomCrawler (解析 XML)
    │
    └── 使用 RssItem (Loader/Rss/)
        ├── toString() → TextDocument.content
        └── toArray()  → TextDocument.metadata (via spread operator)
```

`Rss/` 子目录的存在体现了**按功能组织代码**的原则：
- 主加载器 `RssFeedLoader` 位于 `Loader/` 顶层（与其他加载器并列）
- 辅助类 `RssItem` 位于 `Loader/Rss/` 子目录（表明其专属于 RSS 功能）

---

## 设计决策

### 为什么创建独立子目录？
当某个加载器需要辅助类时，将其放入同名子目录可以：
1. 保持 `Loader/` 目录的整洁（只包含加载器类）
2. 明确标识辅助类的归属关系
3. 为将来可能增加的 RSS 相关类（如 `AtomItem`、`FeedParser` 等）预留空间

### 为什么 RssItem 不是内部类？
PHP 不支持内部类（inner class）。独立文件和子目录是 PHP 中组织紧耦合类的标准方式。

### 命名空间与目录的对应
遵循 PSR-4 自动加载标准：
- 命名空间 `Symfony\AI\Store\Document\Loader\Rss`
- 目录路径 `src/store/src/Document/Loader/Rss/`
