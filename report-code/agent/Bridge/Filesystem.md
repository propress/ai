# Bridge/Filesystem/Filesystem.php 分析报告

## 概述

`Filesystem` 是一个提供沙箱化文件系统操作的多工具类，通过 10 个 `#[AsTool]` 注解暴露读、写、追加、复制、移动、删除、创建目录、检查存在、查询信息、列目录等操作，所有路径均经过 `PathValidator` 安全校验。

## 工具列表

| 工具名 | 方法 | 说明 |
|---|---|---|
| `filesystem_read` | `read()` | 读取文件内容（含大小限制） |
| `filesystem_write` | `write()` | 写入文件（创建或覆盖） |
| `filesystem_append` | `append()` | 追加内容到文件 |
| `filesystem_copy` | `copy()` | 复制文件 |
| `filesystem_move` | `move()` | 移动或重命名 |
| `filesystem_delete` | `delete()` | 删除文件或目录 |
| `filesystem_mkdir` | `mkdir()` | 创建目录 |
| `filesystem_exists` | `exists()` | 检查路径是否存在 |
| `filesystem_info` | `info()` | 获取文件元数据 |
| `filesystem_list` | `list()` | 列出目录内容 |

## 安全机制

- **基路径沙箱**：所有路径由 `PathValidator` 校验，必须位于配置的 `$basePath` 内。
- **写操作开关**：`$allowWrite` 控制写入/追加/复制/移动/创建目录；`$allowDelete` 控制删除。
- **扩展名过滤**：`$allowedExtensions`（白名单）和 `$deniedExtensions`（黑名单，默认拒绝 `php`、`sh`、`exe` 等危险扩展名）。
- **路径模式拒绝**：`$deniedPatterns`（默认拒绝 `.*` 隐藏文件和 `*.env*` 配置文件）。
- **文件大小限制**：`$maxReadSize`（默认 10MB），防止读取超大文件。

## 设计模式

- **门面模式（Facade）**：将多个底层操作（Symfony Filesystem + Finder）统一暴露为简洁的工具接口。
- **最小权限原则（Least Privilege）**：默认不允许删除，写操作也可单独禁用。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Bridge/Filesystem/PathValidator` | 路径安全校验 |
| `Bridge/Filesystem/Exception/PathSecurityException` | 路径违规时抛出 |
| `Bridge/Filesystem/Exception/OperationNotPermittedException` | 操作被禁用时抛出 |
| `Toolbox/Attribute/AsTool` | 10 个工具注册注解 |
