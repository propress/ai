# Bridge/Ollama/Ollama.php 分析报告

## 概述

`Ollama` 集成 Ollama 网络搜索 API，提供网页搜索和网页内容抓取两个工具，支持 Bearer Token 认证和 ScopingHttpClient 两种配置方式。

## 工具列表

| 工具名 | 方法 | 说明 |
|---|---|---|
| `web_search` | `webSearch()` | 使用 Ollama 进行网页搜索 |
| `fetch_webpage` | `fetchWebPage()` | 抓取指定 URL 网页内容 |

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
| `Toolbox/Source/HasSourcesInterface` + `HasSourcesTrait` | 来源收集 |
