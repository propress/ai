# 模板文件分析：templates/

## 模板文件列表

| 文件 | 行数 | 职责 |
|------|------|------|
| `data_collector.html.twig` | 99 行 | Profiler 主面板布局（工具栏 + 菜单 + 面板） |
| `tools.html.twig` | 79 行 | 工具列表子面板 |
| `prompts.html.twig` | 51 行 | 提示列表子面板 |
| `resources.html.twig` | 36 行 | 资源列表子面板 |
| `resource_templates.html.twig` | 36 行 | 资源模板列表子面板 |
| `icon.svg` | 13 行 | MCP 图标（SVG 格式） |

## data_collector.html.twig

### 功能概述
这是 Symfony Web Profiler 的主模板文件，继承自 `@WebProfiler/Profiler/layout.html.twig`，定义了三个核心区块：

### Block: toolbar（调试工具栏）
```twig
{% block toolbar %}
    {% if collector.totalCount > 0 %}
        {# 图标 + 能力总数 + 四类能力分别的数量 #}
    {% endif %}
{% endblock %}
```
- 只在有注册能力时显示（`totalCount > 0`）
- 显示 MCP 图标和能力总数作为主指示器
- 悬浮显示 Tools、Prompts、Resources、Resource Templates 四个分类的数量

### Block: menu（侧边栏菜单）
```twig
{% block menu %}
    <span class="label">
        <span class="icon">{{ include('@Mcp/icon.svg', { y: 16 }) }}</span>
        <strong>MCP</strong>
        <span class="count">{{ collector.totalCount }}</span>
    </span>
{% endblock %}
```
- 在 Profiler 侧边栏显示 "MCP" 菜单项和能力总数

### Block: panel（详情面板）
```twig
{% block panel %}
    {# 概览指标（四个数字） #}
    <section class="metrics">...</section>
    
    {# 四个 Tab 页 #}
    <div class="sf-tabs">
        <div class="tab">Tools</div>      → include('@Mcp/tools.html.twig')
        <div class="tab">Prompts</div>     → include('@Mcp/prompts.html.twig')
        <div class="tab">Resources</div>   → include('@Mcp/resources.html.twig')
        <div class="tab">Resource Templates</div> → include('@Mcp/resource_templates.html.twig')
    </div>
{% endblock %}
```
- 顶部显示四个指标卡片
- 使用 Symfony Profiler 的 Tab 组件分别展示四种能力
- 空的 Tab 添加 `disabled` CSS 类

### 设计模式：组合模板模式（Template Composition）
主模板通过 `include()` 引入四个子模板，每个子模板负责渲染一种能力类型。

**为什么这么做：**
- 遵循单一职责原则——每个子模板只关注一种能力
- 子模板可以独立修改，不影响其他
- 主模板只负责布局和组合

## tools.html.twig

### 功能概述
渲染工具（Tools）列表。每个工具展示为一个可展开的 Tab 页。

### 数据渲染逻辑
```
对每个 tool:
├── Tab 标题: tool.name
├── 描述: tool.description
├── 参数表格 (如果有 inputSchema.properties)
│   ├── Name (代码格式)
│   ├── Type (badge 格式，支持联合类型 join('|'))
│   ├── Required (根据 inputSchema.required 判断)
│   ├── Default (支持数组/标量)
│   └── Description
└── 完整 Schema (可复制的 JSON dump)
```

**技巧说明：**
1. **类型显示**：`prop.type is iterable ? prop.type|join('|') : (prop.type ?? 'any')` — 处理联合类型（如 `string|integer`）和缺省类型
2. **Required 判断**：`tool.inputSchema.required is defined and name in tool.inputSchema.required` — 根据 JSON Schema 的 `required` 数组判断
3. **Schema 复制按钮**：`data-clipboard-text="{{ tool.inputSchema|json_encode(constant('JSON_PRETTY_PRINT'))|e('html_attr') }}"` — 将 JSON Schema 格式化为可复制文本
4. **dump 函数**：`{{ dump(tool.inputSchema) }}` — 使用 Symfony 的 dump 组件展示完整 Schema（支持折叠/展开）

## prompts.html.twig

### 功能概述
渲染提示（Prompts）列表。

### 数据渲染逻辑
```
对每个 prompt:
├── Tab 标题: prompt.name
├── 描述: prompt.description
└── 参数表格 (如果有 arguments)
    ├── Name (代码格式)
    ├── Required (是/否标签)
    └── Description
```

比 Tools 更简单，因为 Prompt 的参数没有复杂的 JSON Schema 结构。

## resources.html.twig

### 功能概述
渲染资源（Resources）列表。

### 数据渲染逻辑
```
对每个 resource:
├── Tab 标题: resource.name
└── 属性表格
    ├── URI (代码格式)
    ├── Description (可选)
    └── MIME Type (代码格式，可选)
```

## resource_templates.html.twig

### 功能概述
渲染资源模板（Resource Templates）列表。

### 数据渲染逻辑
```
对每个 template:
├── Tab 标题: template.name
└── 属性表格
    ├── URI Template (代码格式)
    ├── Description (可选)
    └── MIME Type (代码格式，可选)
```

与 resources.html.twig 结构几乎相同，唯一区别是使用 `uriTemplate` 而非 `uri`。

## icon.svg

MCP 官方图标的 SVG 格式，使用 `currentColor` 作为描边颜色（继承父元素文字颜色），在 Profiler 工具栏和菜单中使用。

## 调用流程

```
DataCollector::getTemplate()
    → '@Mcp/data_collector.html.twig'
        │
        ├── Block: toolbar (页面底部工具栏)
        │   └── include('@Mcp/icon.svg', { y: 18 })
        │
        ├── Block: menu (Profiler 侧边栏)
        │   └── include('@Mcp/icon.svg', { y: 16 })
        │
        └── Block: panel (详情面板)
            ├── 概览指标
            └── sf-tabs
                ├── include('@Mcp/tools.html.twig')
                ├── include('@Mcp/prompts.html.twig')
                ├── include('@Mcp/resources.html.twig')
                └── include('@Mcp/resource_templates.html.twig')
```

## 在哪些场景下被调用

1. **开发环境每次页面加载**：工具栏自动渲染
2. **Profiler 面板访问**：用户点击 MCP 工具栏项
3. **仅在 debug 模式**：生产环境不可用

## 可替换/可扩展点

1. **模板覆盖**：Symfony 支持 Bundle 模板覆盖，在 `templates/bundles/McpBundle/` 目录下创建同名文件即可
2. **自定义样式**：可以在子模板中添加自定义 CSS
3. **新增 Tab**：如需展示额外信息（如调用日志、性能指标），可添加新的 Tab 和子模板

## 外部知识

### Symfony Profiler 模板系统
- 继承 `@WebProfiler/Profiler/layout.html.twig`
- 三个标准区块：`toolbar`、`menu`、`panel`
- 使用 Symfony 的 CSS 类（如 `sf-toolbar-*`、`sf-tabs`、`metric`、`badge`）
- 支持 `dump()` 函数展示复杂数据结构
- 文档：https://symfony.com/doc/current/profiler/data_collector.html#creating-a-custom-web-profiler-template
