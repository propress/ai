# Bridge/Firecrawl/Firecrawl.php 分析报告

## 概述

`Firecrawl` 集成了 [Firecrawl](https://www.firecrawl.dev/) 服务，提供网页抓取（scrape）、网站爬取（crawl）和链接映射（map）三个工具，适用于需要将网页内容转换为 Markdown 或 HTML 格式的场景。

## 工具列表

| 工具名 | 方法 | 说明 |
|---|---|---|
| `firecrawl_scrape` | `scrape()` | 抓取单个 URL 的 Markdown 和 HTML 内容 |
| `firecrawl_crawl` | `crawl()` | 递归爬取整个网站，返回所有页面的内容 |
| `firecrawl_map` | `map()` | 获取网站所有链接的 URL 列表 |

## 关键方法分析

### scrape(string $url): array{url, markdown, html}

POST 到 `/v1/scrape`，返回指定 URL 的 Markdown 和 HTML 内容。

### crawl(string $url): array

POST 到 `/v1/crawl` 启动爬取任务，然后**轮询** `/v1/crawl/{id}` 状态（`usleep(500)` 间隔），直到状态不再为 `'scraping'`，最终返回所有页面内容数组。

### map(string $url): array{url, links}

POST 到 `/v1/map`，返回网站的所有链接列表。

## 注意事项

`crawl()` 方法使用忙等待（`usleep(500)`）轮询状态，在爬取大型网站时可能造成较长阻塞，生产环境中建议配合 Symfony Messenger 异步处理。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 3 个工具注册注解 |
