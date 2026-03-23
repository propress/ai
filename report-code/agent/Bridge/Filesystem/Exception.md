# Bridge/Filesystem/Exception/ 分析报告

## 概述

Filesystem Bridge 拥有独立的异常体系，所有异常均实现本目录下的 `ExceptionInterface`，与 Agent 根异常体系并列（不继承 `Agent\Exception\ExceptionInterface`，而是直接继承自 PHP 原生类）。

## 异常类

### ExceptionInterface

标记接口，继承自 `\Throwable`，是 Filesystem Bridge 所有异常的根接口。

### RuntimeException

基础运行时异常类，继承自 PHP `\RuntimeException` 并实现 `ExceptionInterface`。注意：此类直接实现了 Agent 根级的 `Agent\Exception\ExceptionInterface`。

### PathSecurityException

继承自 `RuntimeException`，在以下情况抛出：
- 路径包含 `..`（路径遍历攻击）
- 路径超出基路径沙箱
- 文件不存在（`mustExist = true` 时）
- 扩展名不在白名单或在黑名单中
- 文件名匹配拒绝模式
- 文件超出最大读取大小

### OperationNotPermittedException

继承自 `RuntimeException`，在以下情况抛出：
- 调用写操作（write/append/copy/move/mkdir）但 `$allowWrite = false`
- 调用删除操作但 `$allowDelete = false`

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Bridge/Filesystem/Filesystem` | 抛出 `PathSecurityException` 和 `OperationNotPermittedException` |
| `Bridge/Filesystem/PathValidator` | 抛出 `PathSecurityException` |
