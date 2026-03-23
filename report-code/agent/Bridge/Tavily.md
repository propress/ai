# Bridge/Tavily/Tavily.php 分析报告

## 概述

`Tavily` 集成 [Tavily](https://tavily.com/) 搜索和内容提取服务，提供网页搜索和批量 URL 内容提取两个工具，并收集结果来源。

## 工具列表

| 工具名 | 方法 | 说明 |
|---|---|---|
| `tavily_search` | `search()` | 互联网搜索，返回 JSON 字符串 |
| `tavily_extract` | `extract()` | 批量抓取多个 URL 的内容 |

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
| `Toolbox/Source/HasSourcesInterface` + `HasSourcesTrait` | 来源收集 |
