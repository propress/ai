# Whisper/ResultConverter 分析报告

解析 Whisper 转录响应，根据 `response_format` 选项返回不同类型：
- `json`/`verbose_json` → `Transcript`（包含 `Segment[]`，带时间戳）
- `text`/`srt`/`vtt` → `TextResult`（纯文本）
- 其他 → `TextResult`（纯文本回退）
