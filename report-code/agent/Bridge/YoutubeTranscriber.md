# Bridge/Youtube/YoutubeTranscriber.php 分析报告

## 概述

`YoutubeTranscriber` 注册为 `youtube_transcript` 工具，通过 `mrmysql/youtube-transcript` 库获取 YouTube 视频字幕文本，将所有字幕行拼接为单一字符串返回。构造函数中检测依赖包是否安装，缺失时抛出 `LogicException`。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
| `Exception/LogicException` | 依赖缺失时抛出 |
