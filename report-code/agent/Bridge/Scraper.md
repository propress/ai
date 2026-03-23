# Bridge/Scraper/Scraper.php 分析报告

## 概述

`Scraper` 使用 Symfony HttpClient + DomCrawler 抓取网页的可见文本和标题，注册为 `scraper` 工具，并将结果作为来源收集。构造函数中检测 `DomCrawler` 是否可用，缺失时抛出 `RuntimeException`。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
| `Toolbox/Source/HasSourcesInterface` + `HasSourcesTrait` | 来源收集 |
