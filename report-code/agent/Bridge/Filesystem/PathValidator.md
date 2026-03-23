# Bridge/Filesystem/PathValidator.php 分析报告

## 概述

`PathValidator` 是 `Filesystem` 工具的安全核心，负责将相对或绝对路径解析为经过验证的实际路径，确保所有文件操作都限制在配置的基路径内，并通过扩展名白/黑名单和路径模式过滤提供额外安全保护。

## 关键方法分析

### validate(string $path, bool $mustExist = true): string

标准文件路径校验，依次执行：
1. `resolvePath()`：解析路径并检查路径遍历（`..`）。
2. `assertWithinBasePath()`：确保解析后路径以基路径开头。
3. `assertExtensionAllowed()`：检查扩展名白/黑名单。
4. `assertNotDeniedPattern()`：用 `fnmatch()` 检查文件名是否匹配拒绝模式。

### validateDirectory(string $path, bool $mustExist = true): string

目录路径校验（跳过扩展名检查）。

### resolvePath()

- 检测 `..` 路径遍历，立即抛出 `PathSecurityException`。
- 对于已存在的路径：使用 `realpath()` 获取真实路径（解析所有符号链接）。
- 对于不存在的路径：解析父目录的真实路径，再拼接文件名。

## 设计模式

- **防御性编程（Defensive Programming）**：多层校验，每一步失败都立即抛出异常，而非延迟到操作执行时。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Bridge/Filesystem/Filesystem` | 持有并调用 `PathValidator` |
| `Bridge/Filesystem/Exception/PathSecurityException` | 路径违规时抛出 |
