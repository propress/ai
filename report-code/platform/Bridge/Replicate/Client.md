# `Replicate/Client.php` 分析

## 概述

Replicate Bridge 的核心 HTTP 客户端，封装了创建预测任务并同步阻塞轮询直至任务完成的完整生命周期，是整个 Bridge 异步转同步模式的关键实现。

## 关键方法

| 方法 | 说明 |
|------|------|
| `request(model, endpoint, body): ResponseInterface` | POST 创建预测任务，然后每秒轮询一次直至终态（`succeeded`/`failed`/`canceled`） |
| `getResponse(id): ResponseInterface`（private） | GET 单次轮询请求，访问 `/v1/predictions/<id>` |

## 设计要点

- 使用 `ClockInterface::sleep(1)` 替代 `sleep(1)`，使测试中可注入 Mock 时钟，避免真实等待
- 轮询终止条件：`status` 为 `succeeded`、`failed` 或 `canceled`（使用 `!in_array` 判断）
- 请求体格式：`{'input': <body>}`，符合 Replicate API 规范
- 认证使用 `auth_bearer` 选项（Bearer Token）

## 关系

- 依赖：`HttpClientInterface`、`ClockInterface`（来自 `symfony/clock`）
- 被 `LlamaModelClient` 调用
- 被 `PlatformFactory` 通过 `new Clock()` 实例化
