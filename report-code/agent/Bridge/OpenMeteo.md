# Bridge/OpenMeteo/OpenMeteo.php 分析报告

## 概述

`OpenMeteo` 集成免费的 [Open-Meteo](https://open-meteo.com/) 天气 API，提供当前天气查询和天气预报两个工具，无需 API 密钥。

## 工具列表

| 工具名 | 方法 | 说明 |
|---|---|---|
| `weather_current` | `current()` | 获取指定经纬度的当前天气 |
| `weather_forecast` | `forecast()` | 获取指定经纬度的天气预报（1-16天）|

## 关键特性

- 内置 WMO 天气代码映射表（0→'Clear' 到 99→'Thunderstorm with Hail'）。
- `forecast()` 的 `$days` 参数通过 `#[With(minimum: 1, maximum: 16)]` 约束范围。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
