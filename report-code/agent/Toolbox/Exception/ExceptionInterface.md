# Toolbox/Exception/ExceptionInterface.php 分析报告

## 概述

`Toolbox/Exception/ExceptionInterface` 继承了 Agent 根异常接口 `Agent\Exception\ExceptionInterface`，是 Toolbox 子模块所有异常类的标记接口，使调用方可以用 `catch (Toolbox\Exception\ExceptionInterface $e)` 捕获所有工具箱相关异常。

## 异常体系

```
\Throwable
└── Agent\Exception\ExceptionInterface
    └── Toolbox\Exception\ExceptionInterface
        ├── ToolException (extends InvalidArgumentException)
        ├── ToolConfigurationException (extends InvalidArgumentException → ToolException)
        ├── ToolExecutionExceptionInterface
        │   └── ToolExecutionException (extends \RuntimeException)
        └── ToolNotFoundException (extends \RuntimeException)
```

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Agent/Exception/ExceptionInterface` | 继承的父接口 |
| `ToolException` | 实现此接口（工具定义无效） |
| `ToolConfigurationException` | 实现此接口（工具配置错误） |
| `ToolExecutionExceptionInterface` | 继承此接口（工具执行失败） |
| `ToolNotFoundException` | 实现此接口（工具未找到） |
