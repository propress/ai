# 文件分析：src/Profiler/DataCollector.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Profiler/DataCollector.php` |
| 完整类名 | `Symfony\AI\McpBundle\Profiler\DataCollector` |
| 继承 | `Symfony\Bundle\FrameworkBundle\DataCollector\AbstractDataCollector` |
| 实现接口 | `Symfony\Component\HttpKernel\DataCollector\LateDataCollectorInterface` |
| 修饰符 | `final` |
| 行数 | 134 行 |
| 作者 | Camille Islasse <guiziweb@gmail.com> |
| 职责 | 收集 MCP 能力数据并提供给 Symfony Web Profiler |

## 功能概述

`DataCollector` 负责从 `TraceableRegistry` 中收集所有已注册的 MCP 能力（Tools、Prompts、Resources、Resource Templates）信息，将其序列化存储，并通过 getter 方法提供给 Twig 模板渲染 Profiler 面板。

## 设计模式

### 1. 数据传输对象模式（DTO Pattern）
DataCollector 将 MCP SDK 的复杂对象（如 `Tool`、`Prompt` 等）转换为简单的关联数组，这些数组只包含展示所需的字段。

**为什么这么做：**
- Profiler 数据需要被序列化存储（用于后续请求的 Profiler 面板查看）
- MCP SDK 的对象可能包含不可序列化的字段（如 callable handler）
- 只提取展示需要的字段，减少存储开销

### 2. 延迟收集模式（Late Collection）
```php
class DataCollector extends AbstractDataCollector implements LateDataCollectorInterface
{
    public function collect(Request $request, Response $response, ?\Throwable $exception = null): void
    {
        // 空实现
    }

    public function lateCollect(): void
    {
        // 真正的数据收集在这里
    }
}
```

**为什么这么做（关键技巧）：**
- `collect()` 在请求处理过程中（kernel.response 事件后）被调用，此时 MCP 能力可能还未完全加载
- `lateCollect()` 在 Profiler 序列化数据之前调用，是最晚的收集时机
- MCP 能力的发现（Discovery）可能是延迟的——SDK 在首次请求时才扫描并注册能力
- 使用 `LateDataCollectorInterface` 确保收集到完整的能力列表

### 3. 模板方法模式（Template Method）
继承 `AbstractDataCollector` 获得了 `$this->data` 属性、`getName()` 的默认实现模板（此处覆盖了）、以及与 Profiler 集成的基础设施。

## 方法详细分析

### collect(Request $request, Response $response, ?\Throwable $exception = null): void

**输入：** Symfony 请求/响应对象和可能的异常

**输出：** 无

**说明：** 空实现。所有收集工作延迟到 `lateCollect()`。

### lateCollect(): void

**输入：** 无

**输出：** 无（修改 `$this->data`）

**逻辑流程：**

```
1. 收集 Tools
   │ foreach registry->getTools()->references as $tool
   │   → { name, description, inputSchema }
   ▼
2. 收集 Prompts
   │ foreach registry->getPrompts()->references as $prompt
   │   → { name, description, arguments: [{name, description, required}] }
   ▼
3. 收集 Resources
   │ foreach registry->getResources()->references as $resource
   │   → { uri, name, description, mimeType }
   ▼
4. 收集 Resource Templates
   │ foreach registry->getResourceTemplates()->references as $template
   │   → { uriTemplate, name, description, mimeType }
   ▼
5. 存储到 $this->data
   │ $this->data = [
   │     'tools' => [...],
   │     'prompts' => [...],
   │     'resources' => [...],
   │     'resourceTemplates' => [...]
   │ ]
```

**技巧说明：**
- Prompt 的 `arguments` 使用 `array_map` 转换为简单数组，因为 `PromptArgument` 对象可能不可序列化
- 使用 `$prompt->arguments ?? []` 处理无参数的 Prompt
- Resource 和 ResourceTemplate 字段不同：Resource 有 `uri`，ResourceTemplate 有 `uriTemplate`

### Getter 方法

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `getTools()` | `array<array{name: string, description: ?string, inputSchema: array<mixed>}>` | 工具列表 |
| `getPrompts()` | `array<array{name: string, description: ?string, arguments: array<mixed>}>` | 提示列表 |
| `getResources()` | `array<array{uri: string, name: string, description: ?string, mimeType: ?string}>` | 资源列表 |
| `getResourceTemplates()` | `array<array{uriTemplate: string, name: string, description: ?string, mimeType: ?string}>` | 资源模板列表 |
| `getTotalCount()` | `int` | 所有能力的总数 |
| `getName()` | `string` | 返回 `'mcp'`（Profiler 标识符） |
| `getTemplate()` | `string`（静态） | 返回 `'@Mcp/data_collector.html.twig'` |

### getName(): string

```php
public function getName(): string
{
    return 'mcp';
}
```

**为什么覆盖基类方法：**
- `AbstractDataCollector` 默认使用完整类名作为 name
- 覆盖为简短的 `'mcp'` 使 Profiler URL 更友好（如 `/_profiler/xxx?panel=mcp`）
- 必须与 `data_collector` 标签中的 `id` 属性匹配

### getTemplate(): string

```php
public static function getTemplate(): string
{
    return '@Mcp/data_collector.html.twig';
}
```

**说明：** 指向 Profiler 面板的 Twig 模板文件。`@Mcp` 是 Bundle 的模板命名空间。

## 数据结构

```php
$this->data = [
    'tools' => [
        [
            'name' => 'weather_forecast',
            'description' => 'Get weather forecast for a city',
            'inputSchema' => [
                'type' => 'object',
                'properties' => [
                    'city' => ['type' => 'string', 'description' => 'City name'],
                ],
                'required' => ['city'],
            ],
        ],
        // ...
    ],
    'prompts' => [
        [
            'name' => 'code_review',
            'description' => 'Review code and provide suggestions',
            'arguments' => [
                ['name' => 'language', 'description' => 'Programming language', 'required' => true],
                ['name' => 'code', 'description' => 'Code to review', 'required' => true],
            ],
        ],
    ],
    'resources' => [
        [
            'uri' => 'file:///config/app.yaml',
            'name' => 'App Config',
            'description' => 'Application configuration file',
            'mimeType' => 'application/yaml',
        ],
    ],
    'resourceTemplates' => [
        [
            'uriTemplate' => 'db://users/{id}',
            'name' => 'User Record',
            'description' => 'Database user by ID',
            'mimeType' => 'application/json',
        ],
    ],
];
```

## 调用流程

```
HTTP 请求进入 Symfony
    │
    ├── MCP 请求处理（可能注册/使用能力）
    │
    ├── kernel.response 事件
    │   └── DataCollector::collect() [空操作]
    │
    ├── Profiler 序列化前
    │   └── DataCollector::lateCollect()
    │       ├── TraceableRegistry::getTools() → Page
    │       ├── TraceableRegistry::getPrompts() → Page
    │       ├── TraceableRegistry::getResources() → Page
    │       └── TraceableRegistry::getResourceTemplates() → Page
    │       → 转换为简单数组存入 $this->data
    │
    ├── Profiler 序列化 $this->data 到存储
    │
    └── 用户访问 Profiler 面板 /_profiler/xxx?panel=mcp
        ├── DataCollector 被反序列化
        ├── Twig 渲染 data_collector.html.twig
        │   ├── collector.tools → 工具列表
        │   ├── collector.prompts → 提示列表
        │   ├── collector.resources → 资源列表
        │   ├── collector.resourceTemplates → 资源模板列表
        │   └── collector.totalCount → 总数（工具栏徽章）
        └── 显示 MCP 能力面板
```

## 在哪些场景下被调用

1. **开发环境每次请求**：Profiler 自动收集数据
2. **Profiler 面板查看**：用户点击 Web Debug Toolbar 的 MCP 图标
3. **仅在 debug 模式**：生产环境不注册此 DataCollector

## 可替换/可扩展点

1. **扩展收集数据**：可以添加更多字段（如 Tool 的调用次数、耗时统计）
2. **自定义模板**：覆盖 `@Mcp/data_collector.html.twig` 可自定义 Profiler 面板外观
3. **添加运行时追踪**：配合 TraceableRegistry 的增强，可以记录实际的 MCP 调用轨迹

## 外部知识

### Symfony Web Profiler
- Symfony 开发工具，在页面底部显示调试工具栏（Web Debug Toolbar）
- 点击工具栏项打开详细的 Profiler 面板
- 每个 DataCollector 负责收集特定领域的数据
- 数据存储在文件系统（默认 `var/cache/dev/profiler/`）
- 文档：https://symfony.com/doc/current/profiler.html

### LateDataCollectorInterface
- 提供 `lateCollect()` 方法，在所有数据收集完成后调用
- 适用于需要在请求完全处理后才能获取数据的场景
- 文档：https://symfony.com/doc/current/profiler/data_collector.html

### AbstractDataCollector
- Symfony 6.0+ 引入的基类
- 提供 `$this->data` 属性用于存储可序列化数据
- 提供 `getName()` 和 `getTemplate()` 的默认实现
- 自动处理 `__sleep()`/`__wakeup()` 序列化
