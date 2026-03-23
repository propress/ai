# 目录分析报告：templates/

## 目录基本信息

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/templates/` |
| 文件数量 | 5 |
| 模板命名空间 | `@Mcp` |
| 职责 | 为 Symfony Web Profiler 提供 MCP 能力展示的 Twig 模板 |

## 目录内容

| 文件 | 类型 | 职责 |
|------|------|------|
| `data_collector.html.twig` | 主模板 | Profiler 面板的主布局，包含 toolbar、menu、panel 三个区块 |
| `tools.html.twig` | 子模板 | 展示已注册的 MCP Tools 及其参数详情 |
| `prompts.html.twig` | 子模板 | 展示已注册的 MCP Prompts 及其参数详情 |
| `resources.html.twig` | 子模板 | 展示已注册的 MCP Resources 的 URI 和元数据 |
| `resource_templates.html.twig` | 子模板 | 展示已注册的 MCP Resource Templates 的 URI 模板和元数据 |
| `icon.svg` | SVG 图标 | MCP 面板的 icon（锁链形状的 MCP 标志） |

## 模板层次结构

```
@WebProfiler/Profiler/layout.html.twig  ← Symfony 框架提供的基础布局
    ↓ extends
data_collector.html.twig               ← MCP 面板主模板
    ├─ block toolbar                    ← 工具栏小图标
    │   └─ icon.svg
    ├─ block menu                       ← 侧边栏菜单项
    │   └─ icon.svg
    └─ block panel                      ← 主面板内容
        ├─ 概览指标区
        └─ Tab 标签页
            ├─ Tools tab → tools.html.twig
            ├─ Prompts tab → prompts.html.twig
            ├─ Resources tab → resources.html.twig
            └─ Resource Templates tab → resource_templates.html.twig
```

## 各模板详细分析

### data_collector.html.twig — 主模板

**继承：** `@WebProfiler/Profiler/layout.html.twig`

**三个 block：**

1. **`toolbar`** — 在 Symfony 调试工具栏中显示 MCP 图标和能力总数
   - 仅当 `collector.totalCount > 0` 时显示
   - 显示 MCP 图标、总数和 "capabilities" 标签
   - 悬浮面板显示 Tools、Prompts、Resources、Resource Templates 各自数量

2. **`menu`** — 在 Profiler 侧边栏显示 MCP 菜单项
   - 始终显示
   - 显示 MCP 图标、"MCP" 文字和能力总数

3. **`panel`** — 主面板内容
   - **指标概览**：4 个 metric 卡片（Tools, Prompts, Resources, Resource Templates）
   - **Tab 标签页**：4 个标签页，每个对应一种能力类型
   - 空的标签页自动标记为 `disabled`

### tools.html.twig — Tools 展示

**数据源：** `collector.tools`

**展示内容：**
- 每个 Tool 一个 tab
- Tab 标题为 Tool 名称
- 描述（如有）
- 参数表格（如有）：
  - Name（参数名）
  - Type（类型，支持多类型用 `|` 分隔）
  - Required（是否必填，红色标签）
  - Default（默认值，支持数组 JSON 序列化）
  - Description（描述）
- Full Schema 区域：显示完整的 inputSchema（使用 `dump()`）
- 复制 Schema 按钮

### prompts.html.twig — Prompts 展示

**数据源：** `collector.prompts`

**展示内容：**
- 每个 Prompt 一个 tab
- Tab 标题为 Prompt 名称
- 描述（如有）
- Arguments 表格（如有）：
  - Name
  - Required
  - Description

### resources.html.twig — Resources 展示

**数据源：** `collector.resources`

**展示内容：**
- 每个 Resource 一个 tab
- URI（code 格式）
- Description（如有）
- MIME Type（如有）

### resource_templates.html.twig — Resource Templates 展示

**数据源：** `collector.resourceTemplates`

**展示内容：**
- 每个 Resource Template 一个 tab
- URI Template（code 格式）
- Description（如有）
- MIME Type（如有）

## 设计模式

### 1. Template Method 模式（Twig 继承）
通过 Twig 的 `extends` 和 `block` 机制实现。基础布局由 Symfony 提供，MCP 只填充自己的内容。

**好处：**
- 与 Symfony Profiler 的其他面板视觉一致
- 自动获得 Profiler 的所有 UI 功能（导航、搜索等）
- 只需关注 MCP 特有的展示内容

### 2. Include 组合模式（Template Composition）
主模板通过 `include` 组合子模板：
```twig
{{ include('@Mcp/tools.html.twig') }}
```

**好处：**
- 每种能力类型的展示逻辑独立
- 子模板可以独立修改而不影响其他部分
- 减少主模板的复杂度

### 3. 条件渲染（Progressive Disclosure）
```twig
{% if collector.totalCount > 0 %}  {# toolbar 仅在有能力时显示 #}
<div class="tab {{ collector.tools is empty ? 'disabled' }}">  {# 空 tab 禁用 #}
```

**好处：**
- 没有注册任何能力时不干扰开发者
- 空的标签页灰显但仍可见，表明该功能存在但未使用

## 技巧分析

### 1. Tool Schema 的多类型显示
```twig
{{ prop.type is iterable ? prop.type|join('|') : (prop.type ?? 'any') }}
```
**为什么这么做：** JSON Schema 的 `type` 可以是字符串（如 `"string"`）或数组（如 `["string", "null"]`）。需要兼容处理。

### 2. 默认值的 JSON 序列化
```twig
{% if prop.default is iterable %}
    <code>{{ prop.default|json_encode }}</code>
{% else %}
    <code>{{ prop.default }}</code>
{% endif %}
```
**为什么这么做：** 默认值可能是数组或对象，需要 JSON 序列化才能正确显示。

### 3. `dump()` 函数显示完整 Schema
```twig
{{ dump(tool.inputSchema) }}
```
**为什么这么做：** `dump()` 是 Symfony VarDumper 的 Twig 扩展，提供可交互的变量浏览器（可展开/折叠），比 JSON 更便于调试。

## 被调用场景

- **运行时：** Symfony Web Profiler 渲染 MCP 面板时
- **前提条件：** `kernel.debug = true` 且 DataCollector 已注册
- **数据来源：** 所有数据从 `DataCollector` 的 getter 方法获取

## 可扩展性

### 可自定义
- 通过 Symfony 的模板覆盖机制覆盖任何模板
- 在应用的 `templates/bundles/McpBundle/` 目录下放置同名模板即可覆盖
- 可以修改展示样式、添加额外信息

### 可添加
- 新增 tab 展示其他 MCP 信息（如会话状态、请求日志）
- 在现有 tab 中添加更多详情（如 handler 来源、调用次数）
