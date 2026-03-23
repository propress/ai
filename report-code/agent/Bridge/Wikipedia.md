# Bridge/Wikipedia/Wikipedia.php 分析报告

## 概述

`Wikipedia` 集成 Wikipedia MediaWiki API，提供文章搜索和文章内容获取两个工具，支持多语言，并将文章内容注册为来源。

## 工具列表

| 工具名 | 方法 | 说明 |
|---|---|---|
| `wikipedia_search` | `search()` | 按关键词搜索文章标题列表 |
| `wikipedia_article` | `article()` | 按标题获取文章完整内容（纯文本）|

## 关键特性

- `search()` 返回标题列表，并提示使用 `wikipedia_article` 工具获取具体内容（引导两步工具调用）。
- 支持重定向（在结果前附加重定向说明）。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
| `Toolbox/Source/HasSourcesInterface` + `HasSourcesTrait` | 来源收集 |
