# Bridge/Clock/Clock.php 分析报告

## 概述

`Clock` 是一个简单的时间查询工具，通过 `#[AsTool('clock', ...)]` 注册，返回当前日期和时间的格式化字符串，支持自定义时区，同时将当前时间注册为来源。

## 关键方法分析

### __invoke(): string

- 使用 `ClockInterface::now()` 获取当前时间（`ClockInterface` 来自 Symfony Clock 组件，便于测试时注入固定时间）。
- 若配置了 `$timezone`，转换时区。
- 创建并添加 `Source('Current Time', 'Clock', '...')` 来源记录。
- 返回格式：`'Current date is YYYY-MM-DD (YYYY-MM-DD) and the time is HH:MM:SS (HH:MM:SS).'`

## 设计亮点

- 使用 Symfony `ClockInterface` 而非 `new \DateTime()`，使工具在测试中可注入模拟时钟（如 `MockClock`），实现确定性测试。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
| `Toolbox/Source/HasSourcesInterface` + `HasSourcesTrait` | 来源收集 |
