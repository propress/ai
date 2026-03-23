# 文件分析报告：src/Profiler/DataCollector.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Profiler/DataCollector.php` |
| 命名空间 | `Symfony\AI\McpBundle\Profiler` |
| 类名 | `DataCollector` |
| 继承 | `Symfony\Bundle\FrameworkBundle\DataCollector\AbstractDataCollector` |
| 实现接口 | `LateDataCollectorInterface` |
| 类型 | `final class` |
| 作者 | Camille Islasse \<guiziweb@gmail.com\> |
| 职责 | 为 Symfony Web Profiler 收集和提供 MCP 服务器能力数据 |

## 类签名

```php
final class DataCollector extends AbstractDataCollector implements LateDataCollectorInterface
{
    public function __construct(private readonly TraceableRegistry $registry)

    public function collect(Request $request, Response $response, ?\Throwable $exception = null): void
    public function lateCollect(): void

    /** @return array<array{name: string, description: ?string, inputSchema: array<mixed>}> */
    public function getTools(): array
    /** @return array<array{name: string, description: ?string, arguments: array<mixed>}> */
    public function getPrompts(): array
    /** @return array<array{uri: string, name: string, description: ?string, mimeType: ?string}> */
    public function getResources(): array
    /** @return array<array{uriTemplate: string, name: string, description: ?string, mimeType: ?string}> */
    public function getResourceTemplates(): array
    public function getTotalCount(): int
    public function getName(): string
    public static function getTemplate(): string
}
```

## 设计模式

### 1. Data Collector 模式（Symfony Profiler 集成）
这是 Symfony Profiler 的标准扩展模式。DataCollector 收集请求相关的数据，并在 Web Profiler 面板中展示。

```
请求处理过程
    ↓ collect() — 在请求完成后调用（此处为空实现）
    ↓ lateCollect() — 在序列化前调用（真正的数据收集）
Profiler 面板
    ↓ getTools(), getPrompts(), etc. — 模板读取数据
    ↓ data_collector.html.twig 渲染
Web Profiler 界面
```

**好处：**
- 与 Symfony Web Profiler 完全集成
- 可视化展示 MCP 服务器的所有能力
- 开发时便于调试和验证注册的 MCP 能力

### 2. Late Data Collection（延迟数据收集）
实现 `LateDataCollectorInterface`，将数据收集推迟到 `lateCollect()` 阶段。

**好处：**
- 确保所有 MCP 能力都已注册完毕后再收集
- 避免在请求处理过程中进行不必要的数据复制
- 符合 Symfony Profiler 的性能最佳实践

### 3. 数据规范化模式（Data Normalization）
`lateCollect()` 将 SDK 的对象转换为简单的 PHP 数组。

**好处：**
- 数据可以安全序列化到 Profiler 存储
- 避免序列化复杂对象（可能包含闭包等不可序列化的内容）
- 数据格式固定，Twig 模板可以可靠地渲染

## 方法详细分析

### `collect(Request $request, Response $response, ?\Throwable $exception = null): void`

**输入：**
- `$request`：当前 HTTP 请求
- `$response`：HTTP 响应
- `$exception`：请求中抛出的异常（如有）

**输出：** 无

**说明：** 空实现。MCP 能力数据不依赖于具体的请求/响应，因此在 `collect()` 阶段不做任何事。

### `lateCollect(): void`

**输入：** 无（从 `$this->registry` 读取数据）

**输出：** 无（数据写入 `$this->data`）

**逻辑流程：**
```
lateCollect()
│
├─ 收集 Tools
│   └─ registry->getTools()->references
│       └─ 遍历每个 tool
│           └─ 提取 name, description, inputSchema
│
├─ 收集 Prompts
│   └─ registry->getPrompts()->references
│       └─ 遍历每个 prompt
│           └─ 提取 name, description
│           └─ 处理 arguments（使用 array_map）
│               └─ 提取每个 arg 的 name, description, required
│
├─ 收集 Resources
│   └─ registry->getResources()->references
│       └─ 遍历每个 resource
│           └─ 提取 uri, name, description, mimeType
│
├─ 收集 Resource Templates
│   └─ registry->getResourceTemplates()->references
│       └─ 遍历每个 template
│           └─ 提取 uriTemplate, name, description, mimeType
│
└─ 写入 $this->data
    └─ ['tools' => [...], 'prompts' => [...], 'resources' => [...], 'resourceTemplates' => [...]]
```

### 数据读取方法

| 方法 | 返回类型 | 数据结构 |
|------|----------|----------|
| `getTools()` | `array` | `[{name, description, inputSchema}, ...]` |
| `getPrompts()` | `array` | `[{name, description, arguments: [{name, description, required}]}, ...]` |
| `getResources()` | `array` | `[{uri, name, description, mimeType}, ...]` |
| `getResourceTemplates()` | `array` | `[{uriTemplate, name, description, mimeType}, ...]` |
| `getTotalCount()` | `int` | 所有四种能力的总数 |

### `getName(): string`

**返回：** `'mcp'`

**说明：** 数据收集器的短名称，用于在 Profiler 中标识。对应 `data_collector` 标签的 `id` 属性。

### `getTemplate(): string`

**返回：** `'@Mcp/data_collector.html.twig'`

**说明：** 指定 Web Profiler 面板使用的 Twig 模板。`@Mcp` 命名空间对应 Bundle 的 `templates/` 目录。

## 技巧分析

### 1. `LateDataCollectorInterface` 的使用
```php
final class DataCollector extends AbstractDataCollector implements LateDataCollectorInterface
```
**为什么这么做：**
- MCP 能力可能在请求处理过程中动态注册
- `lateCollect()` 在 Profiler 序列化数据之前才调用
- 确保收集到最完整的数据

### 2. 空的 `collect()` 方法
```php
public function collect(Request $request, Response $response, ?\Throwable $exception = null): void
{
}
```
**为什么这么做：**
- MCP 能力数据是全局的，不与特定请求关联
- 在 `collect()` 阶段可能还有能力在注册中
- 所有工作推迟到 `lateCollect()` 更安全

### 3. Prompt Arguments 的 array_map 处理
```php
'arguments' => array_map(static fn ($arg) => [
    'name' => $arg->name,
    'description' => $arg->description,
    'required' => $arg->required,
], $prompt->arguments ?? []),
```
**为什么这么做：**
- Prompt 的 arguments 是嵌套的对象数组
- 需要递归地规范化为简单数组
- 使用 `?? []` 处理 arguments 可能为 null 的情况
- 使用 `static fn` 因为不需要访问 `$this`

### 4. `data_collector` 标签包含 `id`
```php
->addTag('data_collector', ['id' => 'mcp'])
```
**为什么这么做：**
- `id` 用于在 Profiler URL 中标识面板
- 也用于 Twig 模板中的 `collector` 变量名
- 必须与 `getName()` 返回值一致

### 5. 数据序列化安全
所有收集的数据都是标量值和数组，没有对象引用。

**为什么这么做：**
- Symfony Profiler 需要序列化数据到文件/数据库
- MCP SDK 的对象可能包含不可序列化的属性（闭包、资源句柄等）
- 只提取需要展示的数据，避免序列化错误

## 被调用场景

1. **注册时机：** `McpBundle::loadExtension()` → 仅在 `kernel.debug = true` 时注册
2. **Symfony Profiler 自动调用** `collect()` 和 `lateCollect()`
3. **Web Profiler 面板渲染时** 调用 getter 方法

## 与 Twig 模板的关系

DataCollector 的数据通过 `collector` 变量在 Twig 模板中可用：

```twig
{# data_collector.html.twig #}
{{ collector.totalCount }}
{{ collector.tools|length }}
{% for tool in collector.tools %}
    {{ tool.name }}
{% endfor %}
```

## 可扩展性

### 可添加的数据
- 请求追踪数据（每次 MCP 调用的耗时、参数等）
- 错误统计
- 能力使用频率统计

### 可自定义模板
- 修改 `templates/data_collector.html.twig` 及子模板可自定义显示样式
- 通过覆盖 `@Mcp` 命名空间下的模板实现自定义

### 不可扩展
- DataCollector 是 `final` 的，不能通过继承扩展
- 数据收集逻辑固定为四种 MCP 能力类型
