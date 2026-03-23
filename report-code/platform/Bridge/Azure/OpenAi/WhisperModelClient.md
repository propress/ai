# `Azure/OpenAi/WhisperModelClient.php` 分析

## 概述

实现 `ModelClientInterface`，向 Azure OpenAI Whisper 语音识别端点发送 multipart/form-data 格式的 HTTP POST 请求，根据任务类型（转录或翻译）动态选择对应的 endpoint 路径。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `OpenAi\Whisper` 实例 |
| `request(Model, payload, options): RawHttpResult` | 根据 `options['task']` 选择 `transcriptions` 或 `translations` 路径，发送 multipart 请求 |

## 设计要点

- 构造时校验 `baseUrl` 不含协议前缀，`deployment`/`apiVersion`/`apiKey` 不为空
- `options['verbose'] = true` 时自动将 `response_format` 设为 `verbose_json`
- `task` 和 `verbose` 选项在处理后从 `$options` 中移除，避免传入请求体
- 端点格式：`https://<baseUrl>/openai/deployments/<deployment>/audio/<transcriptions|translations>`

## 关系

- 实现：`ModelClientInterface`
- 支持模型类型：`Bridge\OpenAi\Whisper`
- 使用枚举：`OpenAi\Whisper\Task`（`TRANSCRIPTION`/`TRANSLATION`）
- 被 `Azure\OpenAi\PlatformFactory` 实例化
