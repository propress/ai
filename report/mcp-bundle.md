# MCP Bundle 模块分析报告

## 1. 模块概述

MCP（Model Context Protocol）是 Anthropic 提出的开放协议标准，旨在让 AI 模型能与外部工具、资源和提示词进行结构化交互。`symfony/mcp-bundle` 是该协议在 Symfony 生态中的官方集成实现，基于 `mcp/sdk` 官方 PHP SDK 构建。

该 Bundle 的核心价值在于：
- **服务端模式**：让 Symfony 应用成为 MCP 服务器，将应用中的业务逻辑暴露为 AI 可调用的工具（Tool）、可读取的资源（Resource）和可使用的提示词（Prompt）
- **客户端模式**：让 Symfony 应用作为 MCP 客户端，消费远程 MCP 服务器提供的工具能力，并集成到本地 Agent 工作流

### 1.1 包信息

| 属性 | 值 |
|------|-----|
| Composer 包名 | `symfony/mcp-bundle` |
| 类型 | `symfony-bundle` |
| 命名空间 | `Symfony\AI\McpBundle` |
| 核心依赖 | `mcp/sdk ^0.4` |
| PHP 最低版本 | 通过 framework-bundle 推断 >=8.2 |
| Symfony 版本要求 | ^7.3\|^8.0 |
| 许可证 | MIT |

### 1.2 模块文件结构

```
src/mcp-bundle/
├── composer.json
├── config/
│   ├── options.php        # Bundle 配置项定义（DefinitionConfigurator）
│   └── services.php       # 核心服务注册（Registry、Builder、Server）
└── src/
    ├── McpBundle.php              # Bundle 入口，扩展注册、属性自动配置
    ├── Command/
    │   └── McpCommand.php         # stdio 传输模式的 Console 命令
    ├── Controller/
    │   └── McpController.php      # HTTP 传输模式的控制器
    ├── DependencyInjection/
    │   └── McpPass.php            # 编译器 Pass，自动注册 MCP 服务
    ├── Profiler/
    │   ├── DataCollector.php      # Symfony Profiler 数据收集器
    │   └── TraceableRegistry.php  # 可追踪的注册表（调试用）
    └── Routing/
        └── RouteLoader.php        # 动态路由加载器
```

---

## 2. 核心接口与输入输出

### 2.1 服务端模式（输入 → 输出）

**HTTP/SSE 传输模式：**
- **输入**：HTTP POST 请求，Body 为 JSON-RPC 2.0 格式的请求体
  ```json
  {
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "search_products",
      "arguments": { "query": "iPhone", "limit": 10 }
    }
  }
  ```
- **输出**：JSON-RPC 2.0 格式响应，包含工具执行结果；或 SSE 事件流（Content-Type: text/event-stream）

**stdio 传输模式：**
- **输入**：标准输入（stdin），换行分隔的 JSON-RPC 消息
- **输出**：标准输出（stdout），换行分隔的 JSON-RPC 响应

### 2.2 客户端模式（输入 → 输出）

- **输入**：MCP 服务器 URL（HTTP 模式）或可执行命令（stdio 模式）
- **输出**：工具列表（`tools/list` 响应）、工具调用结果（`tools/call` 响应）
- **集成**：将远程 MCP 工具注册到本地 Agent Toolbox，扩展 Agent 能力

### 2.3 MCP 属性（PHP Attribute）详解

Bundle 通过 PHP 8 Attribute 自动发现并注册 MCP 能力：

#### `#[McpTool]` — 工具定义

```php
use Mcp\Capability\Attribute\McpTool;

class ProductService
{
    #[McpTool(
        name: 'search_products',
        description: '在商品数据库中搜索商品，支持关键词、分类、价格区间过滤，返回商品列表及库存信息'
    )]
    public function searchProducts(
        string $query,           // AI 按参数名和类型自动生成 JSON Schema
        string $category = '',   // 可选参数（有默认值）
        int $limit = 20,
        float $minPrice = 0.0
    ): array {
        // 业务逻辑
        return ['products' => [], 'total' => 0];
    }
}
```

**参数说明：**
- `name`：工具唯一标识符，AI 按此名称调用工具（必须全局唯一）
- `description`：**极为关键**——AI 通过此描述决定何时调用工具、如何传递参数

#### `#[McpPrompt]` — 提示词模板

```php
use Mcp\Capability\Attribute\McpPrompt;

class PromptProvider
{
    #[McpPrompt(name: 'code_review', description: '代码审查提示词模板')]
    public function codeReview(string $language, string $code): string
    {
        return "请审查以下 {$language} 代码并给出改进建议：\n{$code}";
    }
}
```

#### `#[McpResource]` — 静态资源

```php
use Mcp\Capability\Attribute\McpResource;

class DocumentationProvider
{
    #[McpResource(
        uri: 'docs://api/overview',
        name: 'API 概览文档',
        description: '项目 API 接口的完整概览文档',
        mimeType: 'text/markdown'
    )]
    public function getApiDocs(): string
    {
        return file_get_contents('/path/to/api-docs.md');
    }
}
```

#### `#[McpResourceTemplate]` — 动态资源模板

```php
use Mcp\Capability\Attribute\McpResourceTemplate;

class DatabaseProvider
{
    #[McpResourceTemplate(
        uriTemplate: 'db://{table}/{id}',
        name: '数据库记录',
        description: '按表名和 ID 获取数据库记录'
    )]
    public function getRecord(string $table, string $id): array
    {
        return $this->em->getRepository($table)->find($id);
    }
}
```

---

## 3. 参数不同带来的结果差异

### 3.1 `client_transports` 参数对比

| 参数 | `stdio: true` | `http: true` |
|------|--------------|-------------|
| **传输方式** | 标准输入/输出流 | HTTP/SSE（StreamableHTTP） |
| **注册服务** | McpCommand（Console 命令） | McpController（HTTP 控制器）+ 路由 |
| **使用场景** | 本地 CLI、IDE 插件（Cursor、Continue.dev） | Web 服务、远程调用（Claude Desktop） |
| **延迟** | 进程间通信，极低延迟（<1ms） | 网络延迟（取决于网络）|
| **安全性** | 本地进程，天然隔离，无需认证 | 需要配置 Symfony 防火墙认证 |
| **部署要求** | 直接运行 PHP CLI | 需要 Web 服务器（Nginx/Apache/FrankenPHP） |
| **并发支持** | 单会话 | 多会话并发（依赖 session.store） |
| **调试** | Console 输出 | Symfony Profiler 面板 |

两者可同时启用：
```yaml
symfony_ai_mcp:
    client_transports:
        stdio: true   # 同时支持 CLI 工具
        http: true    # 和 Web 客户端
```

### 3.2 `description` 参数的重要性对比

description 是影响 AI 工具使用质量的最关键参数：

**差的 description（AI 可能误用）：**
```php
#[McpTool(name: 'get_data', description: '获取数据')]
public function getData(string $id): array { ... }
```
- AI 不知道"数据"是什么
- 不知道 `id` 的格式要求
- 可能在不适合的场景调用

**好的 description（AI 精准调用）：**
```php
#[McpTool(
    name: 'get_order_details',
    description: '根据订单号获取订单详细信息，包括商品列表、价格、物流状态和收货地址。
    订单号格式为 ORD-YYYYMMDD-XXXXX（如 ORD-20240115-00001）。
    仅在用户询问具体订单状态时调用此工具。'
)]
public function getOrderDetails(string $orderId): array { ... }
```
- AI 清楚知道工具用途
- 知道参数格式要求
- 能准确判断调用时机

### 3.3 `session.store` 参数对比

| 存储类型 | `file`（默认） | `memory` | `cache` |
|---------|-------------|---------|--------|
| **持久化** | ✅ 文件系统持久 | ❌ 进程内存 | ✅ 依赖缓存后端 |
| **重启恢复** | ✅ 重启后会话保持 | ❌ 重启丢失 | ✅ Redis 可持久化 |
| **多进程** | ⚠️ 文件锁竞争 | ❌ 不支持 | ✅ 支持 |
| **适用场景** | 开发环境、单实例 | 测试、无状态 | 生产环境、集群部署 |
| **配置复杂度** | 低 | 极低 | 中等（需配置缓存池） |

```yaml
# 生产环境推荐：cache 存储 + Redis
symfony_ai_mcp:
    http:
        session:
            store: cache
            cache_pool: cache.app   # 使用 Redis 缓存池
            ttl: 7200               # 会话 2 小时过期
```

### 3.4 `discovery.scan_dirs` 参数

控制 MCP SDK 扫描哪些目录以自动发现带有 MCP 属性的类：

```yaml
symfony_ai_mcp:
    discovery:
        scan_dirs:
            - src/McpTool       # 只扫描专用目录（推荐生产环境）
            - src/Service       # 也扫描服务目录
        exclude_dirs:
            - src/Service/Internal  # 排除内部服务
```

- 扫描目录越少，容器构建越快
- 精确控制哪些类可被 MCP 发现，提高安全性

---

## 4. 实际应用场景

### 4.1 将 Symfony CMS 暴露为 Claude Desktop 工具

**场景描述**：编辑人员通过 Claude Desktop 直接用自然语言操作 CMS，如"创建一篇关于春节的文章"、"把首页横幅图片换成这张"。

**实现步骤**：

1. 标注 CMS 服务方法：
```php
<?php
// src/McpTool/ContentTool.php

use Mcp\Capability\Attribute\McpTool;
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

class ContentTool
{
    public function __construct(
        private readonly ArticleRepository $articles,
        private readonly EntityManagerInterface $em,
    ) {}

    #[McpTool(
        name: 'create_article',
        description: '创建新文章并保存为草稿状态。需要提供标题、正文内容（Markdown 格式）和分类标签。返回文章 ID 和预览链接。'
    )]
    public function createArticle(
        string $title,
        string $content,
        string $category,
        string $tags = ''
    ): array {
        $article = new Article($title, $content, $category);
        if ('' !== $tags) {
            $article->setTags(explode(',', $tags));
        }
        $this->em->persist($article);
        $this->em->flush();

        return [
            'id' => $article->getId(),
            'status' => 'draft',
            'preview_url' => '/preview/' . $article->getSlug(),
        ];
    }

    #[McpTool(
        name: 'list_articles',
        description: '列出文章列表，可按状态（draft/published）和分类过滤，支持分页。'
    )]
    public function listArticles(
        string $status = 'all',
        string $category = '',
        int $page = 1,
        int $limit = 20
    ): array {
        return $this->articles->findByFilters($status, $category, $page, $limit);
    }
}
```

2. Bundle 配置：
```yaml
# config/packages/mcp.yaml
symfony_ai_mcp:
    app: 'My CMS'
    version: '1.0.0'
    description: '内容管理系统 MCP 服务器'
    client_transports:
        http: true
    http:
        path: /mcp
        session:
            store: cache
```

3. Claude Desktop 配置：
```json
{
  "mcpServers": {
    "my-cms": {
      "url": "https://cms.example.com/mcp",
      "transport": "http"
    }
  }
}
```

**效果**：编辑人员对话："帮我创建一篇关于'Symfony 7.3 新特性'的技术文章，分类为'技术'，内容包括..." → Claude 自动调用 `create_article` 工具完成创建。

---

### 4.2 本地开发助手（Cursor / Continue.dev 集成）

**场景描述**：开发者在 Cursor IDE 中通过 AI 对话完成 Symfony 项目开发任务。

**实现**：使用 `stdio` 传输模式，Cursor 通过进程间通信调用工具。

```yaml
symfony_ai_mcp:
    client_transports:
        stdio: true   # 启用 Console 命令 bin/console mcp:server
```

```php
// src/McpTool/DevTool.php
class DevTool
{
    #[McpTool(
        name: 'run_tests',
        description: '运行 PHPUnit 测试套件，可指定测试文件或测试方法。返回测试结果（通过数、失败数、错误详情）。'
    )]
    public function runTests(string $filter = '', string $testsuite = ''): array
    {
        $cmd = 'vendor/bin/phpunit';
        if ('' !== $filter) {
            $cmd .= ' --filter ' . escapeshellarg($filter);
        }
        exec($cmd . ' 2>&1', $output, $exitCode);

        return ['exit_code' => $exitCode, 'output' => implode("\n", $output)];
    }

    #[McpTool(
        name: 'query_database',
        description: '执行只读 SQL 查询，返回结果集。仅支持 SELECT 语句，用于调试和数据探查。'
    )]
    public function queryDatabase(string $sql): array
    {
        // 安全检查：只允许 SELECT
        if (!str_starts_with(strtoupper(ltrim($sql)), 'SELECT')) {
            return ['error' => '只允许 SELECT 查询'];
        }
        return $this->connection->fetchAllAssociative($sql);
    }
}
```

Cursor 配置（`.cursor/mcp.json`）：
```json
{
  "mcpServers": {
    "symfony-dev": {
      "command": "php",
      "args": ["bin/console", "mcp:server"],
      "cwd": "/path/to/project"
    }
  }
}
```

---

### 4.3 跨微服务 AI 工具共享

**场景描述**：企业内多个 Symfony 微服务各自暴露 MCP 工具，中央 AI Agent 统一调用。

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  用户服务     │    │  订单服务     │    │  库存服务     │
│  /mcp        │    │  /mcp        │    │  /mcp        │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼───────┐
                    │  AI Gateway  │
                    │  (Agent)     │
                    └──────────────┘
```

中央 Agent 配置（ai-bundle）：
```yaml
symfony_ai:
    agent:
        toolbox:
            mcp_servers:
                - url: 'http://user-service/mcp'
                - url: 'http://order-service/mcp'
                - url: 'http://inventory-service/mcp'
```

---

### 4.4 企业数据查询助手

**场景描述**：业务分析师用自然语言查询企业数据，无需编写 SQL。

```php
class DataAnalyticsTool
{
    #[McpTool(
        name: 'get_sales_report',
        description: '获取销售报表数据。可按时间范围（start_date、end_date，格式 YYYY-MM-DD）、
        地区（region，如"华北"、"华南"）、产品线（product_line）过滤。
        返回销售额、订单数、客单价等指标。'
    )]
    public function getSalesReport(
        string $startDate,
        string $endDate,
        string $region = 'all',
        string $productLine = 'all'
    ): array {
        return $this->reportRepository->getSalesData($startDate, $endDate, $region, $productLine);
    }

    #[McpResource(
        uri: 'reports://kpi/current',
        name: '实时 KPI 面板',
        description: '当前实时 KPI 指标，包括日销售额、活跃用户数、转化率',
        mimeType: 'application/json'
    )]
    public function getCurrentKpi(): string
    {
        return json_encode($this->kpiService->getCurrent(), JSON_THROW_ON_ERROR);
    }

    #[McpResourceTemplate(
        uriTemplate: 'reports://monthly/{year}/{month}',
        name: '月度报表',
        description: '指定年月的月度汇总报表，year 为四位年份，month 为两位月份（01-12）'
    )]
    public function getMonthlyReport(string $year, string $month): string
    {
        return json_encode($this->reportRepository->getMonthly((int)$year, (int)$month));
    }
}
```

---

### 4.5 自动化运维助手

**场景描述**：运维人员通过自然语言执行运维任务，集成到 CI/CD 流水线。

```php
class DevOpsTool
{
    #[McpTool(
        name: 'get_deployment_status',
        description: '获取指定环境（env：dev/staging/prod）的最新部署状态，
        返回版本号、部署时间、健康检查结果。'
    )]
    public function getDeploymentStatus(string $env): array
    {
        return $this->deploymentService->getStatus($env);
    }

    #[McpTool(
        name: 'tail_logs',
        description: '获取应用日志的最新条目。service 参数指定服务名，
        level 参数过滤日志级别（error/warning/info），lines 控制返回行数（最多 100）。'
    )]
    public function tailLogs(
        string $service,
        string $level = 'error',
        int $lines = 50
    ): array {
        return $this->logService->getRecent($service, $level, min($lines, 100));
    }

    #[McpTool(
        name: 'trigger_deployment',
        description: '触发指定环境的部署流程。需要提供 env（staging/prod）和 version（git tag 或 branch）。
        注意：prod 环境需要额外确认。返回部署任务 ID 用于跟踪进度。'
    )]
    public function triggerDeployment(string $env, string $version): array
    {
        if ('prod' === $env) {
            // 生产环境额外检查
            $this->auditLog->record('deployment_requested', ['env' => $env, 'version' => $version]);
        }
        return $this->deploymentService->trigger($env, $version);
    }
}
```

---

### 4.6 AI 辅助客户支持系统

**场景描述**：客服人员通过 AI 助手快速处理客户工单，查询订单、处理退款、发送通知。

```php
class CustomerSupportTool
{
    #[McpTool(
        name: 'search_customer',
        description: '按邮箱、手机号或用户 ID 查找客户信息。
        返回客户基本信息、最近订单、账户状态。用于工单处理前的客户验证。'
    )]
    public function searchCustomer(string $identifier): array
    {
        return $this->customerService->findByIdentifier($identifier);
    }

    #[McpTool(
        name: 'process_refund',
        description: '为指定订单发起退款申请。需要订单号（order_id）、退款金额（amount，单位：分）
        和退款原因（reason）。退款将在 3-5 个工作日到账。返回退款申请单号。'
    )]
    public function processRefund(
        string $orderId,
        int $amount,
        string $reason
    ): array {
        return $this->refundService->initiate($orderId, $amount, $reason);
    }

    #[McpTool(
        name: 'send_notification',
        description: '向客户发送通知消息。channel 支持 email/sms/push，
        template 为预定义模板名（如 refund_approved、order_shipped），
        params 为模板参数键值对。'
    )]
    public function sendNotification(
        string $customerId,
        string $channel,
        string $template,
        array $params = []
    ): array {
        return $this->notificationService->send($customerId, $channel, $template, $params);
    }
}
```

---

### 4.7 多模型 IDE 扩展（Continue.dev / Cline 集成）

**场景描述**：支持任何兼容 MCP 协议的 AI 助手，通过 stdio 模式与 Symfony 项目交互。

```php
class SymfonyProjectTool
{
    #[McpTool(
        name: 'list_routes',
        description: '列出 Symfony 应用的所有路由，可按前缀过滤。
        返回路由名、路径、HTTP 方法、控制器信息。用于了解 API 接口结构。'
    )]
    public function listRoutes(string $prefix = ''): array
    {
        $routes = [];
        foreach ($this->router->getRouteCollection() as $name => $route) {
            if ('' === $prefix || str_starts_with($route->getPath(), $prefix)) {
                $routes[] = [
                    'name' => $name,
                    'path' => $route->getPath(),
                    'methods' => $route->getMethods(),
                    'defaults' => $route->getDefaults(),
                ];
            }
        }
        return $routes;
    }

    #[McpTool(
        name: 'get_entity_schema',
        description: '获取 Doctrine 实体的字段定义和关联关系。
        entity_class 为完整类名（如 App\\Entity\\User）。
        用于了解数据模型结构，辅助编写查询。'
    )]
    public function getEntitySchema(string $entityClass): array
    {
        $metadata = $this->em->getClassMetadata($entityClass);
        return [
            'fields' => $metadata->fieldMappings,
            'associations' => $metadata->associationMappings,
            'table' => $metadata->getTableName(),
        ];
    }

    #[McpResource(
        uri: 'symfony://container/services',
        name: 'DI 容器服务列表',
        description: '当前应用注册的所有公开 DI 服务列表',
        mimeType: 'application/json'
    )]
    public function getServices(): string
    {
        return json_encode($this->container->getServiceIds(), JSON_THROW_ON_ERROR);
    }
}
```

---

### 4.8 数据分析平台集成

**场景描述**：数据科学家通过自然语言指令让 AI 直接访问和分析业务数据集。

```php
class DataPlatformTool
{
    #[McpResourceTemplate(
        uriTemplate: 'data://{dataset}/{format}',
        name: '数据集访问',
        description: '访问指定数据集，dataset 为数据集名称，format 支持 csv/json/parquet。
        可用数据集：sales_daily, user_behavior, product_inventory, marketing_campaign'
    )]
    public function getDataset(string $dataset, string $format): string
    {
        $allowedDatasets = ['sales_daily', 'user_behavior', 'product_inventory', 'marketing_campaign'];
        if (!in_array($dataset, $allowedDatasets, true)) {
            throw new \InvalidArgumentException('数据集不存在：' . $dataset);
        }
        return $this->dataExporter->export($dataset, $format);
    }

    #[McpTool(
        name: 'execute_analysis',
        description: '执行预定义的数据分析任务。
        analysis_type 支持：trend_analysis（趋势分析）, cohort_analysis（用户分群）,
        funnel_analysis（漏斗分析）, ab_test_result（A/B 测试结果）。
        date_range 为分析时间范围，格式：YYYY-MM-DD,YYYY-MM-DD。
        返回分析结果和可视化数据。'
    )]
    public function executeAnalysis(
        string $analysisType,
        string $dateRange,
        array $params = []
    ): array {
        return $this->analysisEngine->run($analysisType, $dateRange, $params);
    }
}
```

---

## 5. 属性（Attribute）详解

### 5.1 `#[McpTool]` 完整参数

```php
#[McpTool(
    name: 'tool_identifier',     // 字符串，工具唯一标识符，AI 按此名称调用
    description: '...',          // 字符串，描述工具功能，影响 AI 决策（最重要）
)]
```

**方法参数自动转换为 JSON Schema**：
- `string $param` → `{"type": "string"}`
- `int $param` → `{"type": "integer"}`
- `float $param` → `{"type": "number"}`
- `bool $param` → `{"type": "boolean"}`
- `array $param` → `{"type": "array"}`
- `?string $param = null` → `{"type": "string"}` + 标记为可选
- 带默认值的参数 → 标记为可选参数

**方法返回值处理**：
- `array` → JSON 对象/数组
- `string` → 文本内容
- `bool` → 布尔结果

### 5.2 `#[McpResource]` vs `#[McpResourceTemplate]` 对比

| 特性 | `#[McpResource]` | `#[McpResourceTemplate]` |
|------|-----------------|------------------------|
| **URI 类型** | 静态固定 URI | 动态 URI 模板（含变量） |
| **URI 示例** | `docs://api/v1` | `docs://api/{version}/{section}` |
| **适用场景** | 固定内容（配置文件、静态文档） | 动态内容（按 ID 查询、分页数据） |
| **参数** | 无（URI 固定） | URI 模板变量自动解析为方法参数 |
| **缓存友好** | ✅ 内容相对稳定，可缓存 | ⚠️ 内容动态，缓存需谨慎 |

### 5.3 DI 标签映射

Bundle 通过 `McpBundle::registerMcpAttributes()` 注册属性自动配置：

| PHP Attribute | DI 标签 |
|--------------|--------|
| `McpTool::class` | `mcp.tool` |
| `McpPrompt::class` | `mcp.prompt` |
| `McpResource::class` | `mcp.resource` |
| `McpResourceTemplate::class` | `mcp.resource_template` |

---

## 6. 传输模式对比与配置

### 6.1 完整配置参考

```yaml
# config/packages/mcp.yaml
symfony_ai_mcp:
    # 服务器元信息（展示给 MCP 客户端）
    app: 'My Symfony App'           # 应用名称，默认 'app'
    version: '1.2.0'                # 版本号，默认 '0.0.1'
    description: '我的 Symfony 应用 MCP 服务器'  # 可选描述
    website_url: 'https://example.com'           # 可选网站 URL
    icons:
        - src: '/icon-192.png'
          mime_type: 'image/png'
          sizes: ['192x192']
    pagination_limit: 50            # 列表查询分页限制，默认 50
    instructions: '使用此 MCP 服务器管理我的 Symfony 应用内容'  # 可选使用说明

    # 工具发现配置
    discovery:
        scan_dirs:
            - src/McpTool           # 扫描这些目录自动发现 MCP 类
        exclude_dirs:
            - src/McpTool/Internal  # 排除这些子目录

    # 传输层配置（同时启用两种模式）
    client_transports:
        stdio: false                # 是否启用 stdio 传输（Console 命令）
        http: true                  # 是否启用 HTTP/SSE 传输（Web 控制器）

    # HTTP 传输专用配置
    http:
        path: '/_mcp'              # MCP 端点路径，默认 '/_mcp'
        session:
            store: file            # 会话存储：file | memory | cache
            directory: '%kernel.cache_dir%/mcp-sessions'  # file 模式存储目录
            cache_pool: cache.mcp.sessions  # cache 模式使用的缓存池
            prefix: 'mcp-'         # cache 模式的键前缀
            ttl: 3600              # 会话 TTL（秒），最小 1
```

### 6.2 HTTP/SSE 模式详细流程

```
Claude Desktop                    Symfony App
     │                                 │
     │  GET /_mcp                      │
     │ ──────────────────────────────► │
     │  (initialize request)           │
     │                                 │ McpController::handle()
     │  ◄────────────────────────────  │   └─ StreamableHttpTransport
     │  200 OK (server info + caps)    │      └─ Server::run()
     │                                 │
     │  POST /_mcp                     │
     │  Body: {"method":"tools/list"}  │
     │ ──────────────────────────────► │
     │                                 │ Registry 返回工具列表
     │  ◄────────────────────────────  │
     │  SSE: tools list                │
     │                                 │
     │  POST /_mcp                     │
     │  Body: {"method":"tools/call",  │
     │         "params":{"name":"..."}}│
     │ ──────────────────────────────► │
     │                                 │ 执行 #[McpTool] 方法
     │  ◄────────────────────────────  │
     │  SSE: tool result               │
```

### 6.3 Stdio 模式详细流程

```bash
# Cursor IDE 启动 MCP 服务器进程
php bin/console mcp:server

# 通过 stdin/stdout 进行 JSON-RPC 通信
# stdin:  {"jsonrpc":"2.0","id":1,"method":"tools/list"}
# stdout: {"jsonrpc":"2.0","id":1,"result":{"tools":[...]}}
```

---

## 7. 路由与控制器

### 7.1 RouteLoader

`RouteLoader` 实现 Symfony `Loader` 接口，动态注册 MCP 路由：

```php
// 路由注册逻辑
$collection->add('_mcp_endpoint', new Route(
    $this->httpPath,                    // 路径：'/_mcp'（可配置）
    ['_controller' => 'mcp.server.controller::handle'],
    methods: [
        Request::METHOD_GET,            // 初始化连接
        Request::METHOD_POST,           // 工具调用
        Request::METHOD_DELETE,         // 关闭会话
        Request::METHOD_OPTIONS,        // CORS 预检
    ]
));
```

在 `config/routes.yaml` 中启用：
```yaml
mcp:
    resource: '.'
    type: mcp
```

### 7.2 McpController

HTTP 请求处理流程：
1. 接收 Symfony `Request` 对象
2. 使用 `HttpMessageFactoryInterface` 转换为 PSR-7 `RequestInterface`
3. 创建 `StreamableHttpTransport` 实例
4. 调用 `Server::run($transport)` 执行 MCP 协议处理
5. 将 PSR-7 `ResponseInterface` 转换回 Symfony `Response`
6. 检测 `text/event-stream` 响应，启用流式输出

---

## 8. DI 标签与编译器 Pass

### 8.1 McpPass 工作流程

`McpPass` 在容器编译阶段自动将所有 MCP 服务注入到 Server Builder：

```php
public function process(ContainerBuilder $container): void
{
    // 1. 收集所有带 MCP 标签的服务
    $mcpTags = ['mcp.tool', 'mcp.prompt', 'mcp.resource', 'mcp.resource_template'];

    foreach ($mcpTags as $tag) {
        $taggedServices = $container->findTaggedServiceIds($tag);
        $allMcpServices = array_merge($allMcpServices, $taggedServices);
    }

    // 2. 创建 ServiceLocator，包含所有 MCP 服务
    $serviceLocatorRef = ServiceLocatorTagPass::register($container, $serviceReferences);

    // 3. 将 ServiceLocator 注入到 mcp.server.builder
    $container->getDefinition('mcp.server.builder')
        ->addMethodCall('setContainer', [$serviceLocatorRef]);
}
```

### 8.2 服务注册链

```
config/services.php
├── mcp.registry (Registry)           ← 能力注册表（工具、资源、提示词）
├── mcp.server.builder (Builder)      ← 服务器构建器
│   ├── setRegistry(mcp.registry)
│   ├── setSession(mcp.session.store)
│   ├── addRequestHandlers(...)
│   ├── addNotificationHandlers(...)
│   └── addLoaders(...)
└── mcp.server (Server)               ← 实际服务器实例
    └── factory: mcp.server.builder::build()
```

### 8.3 自动配置接口

Bundle 为以下接口自动添加 DI 标签：

| 接口 | DI 标签 |
|------|--------|
| `LoaderInterface` | `mcp.loader` |
| `RequestHandlerInterface` | `mcp.request_handler` |
| `NotificationHandlerInterface` | `mcp.notification_handler` |

---

## 9. Profiler 调试面板

### 9.1 组件构成

仅在 `kernel.debug = true` 时启用：

- **`TraceableRegistry`**：装饰器模式包装 `mcp.registry`，记录所有工具调用
- **`DataCollector`**：Symfony Profiler 数据收集器，聚合追踪数据

```php
// 调试环境自动注册
if ($builder->getParameter('kernel.debug')) {
    $traceableRegistry = (new Definition('mcp.traceable_registry'))
        ->setClass(TraceableRegistry::class)
        ->setArguments([new Reference('.inner')])
        ->setDecoratedService('mcp.registry')  // 装饰原始 registry
        ->addTag('kernel.reset', ['method' => 'reset']);

    $dataCollector = (new Definition(DataCollector::class))
        ->setArguments([new Reference('mcp.traceable_registry')])
        ->addTag('data_collector', ['id' => 'mcp']);
}
```

### 9.2 Profiler 面板功能

- 📊 **工具调用统计**：总调用次数、成功/失败比率
- 📝 **调用历史**：每次工具调用的时间戳、参数、返回值
- ⏱️ **性能数据**：每次工具执行耗时
- 🔍 **请求详情**：完整的 JSON-RPC 请求/响应内容

访问路径：`/_profiler` → 选择请求 → `MCP` 面板

---

## 10. 与 Agent 模块集成

### 10.1 mcp-bundle 作为 MCP 服务端 + Agent 作为客户端

```
┌─────────────────────────────────────────────────────────┐
│  Symfony App A（MCP 服务端）                              │
│  mcp-bundle → 暴露业务工具                                │
│  GET/POST /_mcp                                          │
└─────────────────────┬───────────────────────────────────┘
                      │ HTTP
┌─────────────────────▼───────────────────────────────────┐
│  Symfony App B（MCP 客户端 + AI Agent）                   │
│  ai-bundle + agent                                       │
│  ToolboxFactory::createFromMcpServer(url)                │
│  → Agent 可调用远程工具                                   │
└─────────────────────────────────────────────────────────┘
```

### 10.2 配置示例

```php
// App B: 消费 App A 的 MCP 工具
use Symfony\AI\Agent\Toolbox\ToolboxFactory;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Platform\Bridge\OpenAI\GPT;

// 创建包含远程 MCP 工具的 Toolbox
$toolbox = ToolboxFactory::createFromMcpServer(
    'https://app-a.example.com/_mcp'
);

// 创建 Agent
$agent = new Agent($platform, $toolbox);
$response = $agent->call([new UserMessage('搜索最新的 iPhone 商品')]);
```

### 10.3 工具发现流程

```
Agent 调用 AI 模型
    │
    ▼
AI 模型决定调用 MCP 工具
    │
    ▼
Agent Toolbox → HTTP 调用 /_mcp (tools/call)
    │
    ▼
McpController → Server → Registry → 执行 #[McpTool] 方法
    │
    ▼
返回结果给 AI 模型 → 最终响应用户
```

---

## 11. 安全考虑

### 11.1 HTTP 模式认证

MCP 端点需要在 Symfony 防火墙中保护：

```yaml
# config/packages/security.yaml
security:
    firewalls:
        mcp:
            pattern: ^/_mcp
            stateless: true
            # 方式 1：API Token 认证
            custom_authenticators:
                - App\Security\McpTokenAuthenticator
            # 方式 2：基于 IP 白名单（仅内网使用）
            # access_control:
            #   - { path: ^/_mcp, ips: [127.0.0.1, 10.0.0.0/8] }

    access_control:
        - { path: ^/_mcp, roles: ROLE_MCP_CLIENT }
```

### 11.2 工具权限控制

在工具方法内使用 Symfony Security 控制访问：

```php
use Symfony\Bundle\SecurityBundle\Security;

class SecureTool
{
    public function __construct(private readonly Security $security) {}

    #[McpTool(name: 'delete_user', description: '删除用户（仅管理员）')]
    public function deleteUser(int $userId): array
    {
        // 运行时权限检查
        if (!$this->security->isGranted('ROLE_ADMIN')) {
            return ['error' => '权限不足'];
        }
        // 执行删除操作
        return ['success' => true];
    }
}
```

### 11.3 输入验证

```php
#[McpTool(name: 'execute_query', description: '执行数据库查询')]
public function executeQuery(string $sql, int $limit = 100): array
{
    // 输入验证
    if ($limit > 1000) {
        $limit = 1000;  // 强制上限
    }

    // SQL 注入防护：只允许白名单查询
    if (!preg_match('/^SELECT\s+/i', ltrim($sql))) {
        return ['error' => '只允许 SELECT 查询'];
    }

    return $this->connection->fetchAllAssociative(
        $sql . ' LIMIT ' . $limit
    );
}
```

### 11.4 速率限制

```yaml
# config/packages/rate_limiter.yaml
framework:
    rate_limiter:
        mcp_api:
            policy: sliding_window
            limit: 100
            interval: '1 minute'
```

```php
// config/packages/security.yaml 中为 /_mcp 添加速率限制中间件
```

---

## 12. 会话管理深度分析

### 12.1 三种会话存储对比

**File 存储（默认）**：
```php
// 内部使用 FileSessionStore
// 会话文件存储在 %kernel.cache_dir%/mcp-sessions/
// 文件名格式：mcp-{session_id}
```

优点：零配置、开箱即用、重启后会话保持
缺点：多进程并发时文件锁竞争、不适合水平扩展

**Memory 存储**：
```php
// 内部使用 InMemorySessionStore
// 仅存储在 PHP 进程内存中
```

优点：极快、零 I/O
缺点：进程重启丢失、不支持多进程/多服务器

**Cache 存储（生产推荐）**：
```php
// 内部使用 Psr16SessionStore，包装 Symfony Cache
// 默认缓存池：cache.mcp.sessions（自动创建为 Psr16Cache(cache.app)）
```

优点：支持 Redis/Memcached 等后端、水平扩展、TTL 自动过期
缺点：需要缓存服务依赖

### 12.2 会话 TTL 设计

- 默认 TTL：3600 秒（1 小时）
- 每次 MCP 交互自动续期
- AI 助手长对话时建议增大 TTL（如 86400 秒 = 1 天）

---

## 13. 版本与兼容性

| 依赖 | 版本要求 |
|------|---------|
| PHP | >=8.2（通过 framework-bundle 推断） |
| `mcp/sdk` | ^0.4 |
| Symfony Framework Bundle | ^7.3\|^8.0 |
| Symfony Console | ^7.3\|^8.0 |
| Symfony Routing | ^7.3\|^8.0 |
| Symfony HTTP Foundation | ^7.3\|^8.0 |
| `php-http/discovery` | ^1.20 |
| `symfony/psr-http-message-bridge` | ^7.3\|^8.0 |

---

## 14. 完整示例：从零搭建 MCP 服务器

### 14.1 安装

```bash
composer require symfony/mcp-bundle
```

### 14.2 注册 Bundle

```php
// config/bundles.php
return [
    // ...
    Symfony\AI\McpBundle\McpBundle::class => ['all' => true],
];
```

### 14.3 配置

```yaml
# config/packages/mcp.yaml
symfony_ai_mcp:
    app: 'My App'
    version: '1.0.0'
    client_transports:
        http: true
    http:
        path: /mcp
        session:
            store: cache

# config/routes/mcp.yaml
mcp:
    resource: '.'
    type: mcp
```

### 14.4 创建工具

```php
// src/McpTool/HelloTool.php
<?php

namespace App\McpTool;

use Mcp\Capability\Attribute\McpTool;

class HelloTool
{
    #[McpTool(name: 'hello', description: '向指定名字的人打招呼，返回问候语')]
    public function hello(string $name): array
    {
        return ['message' => '你好，' . $name . '！'];
    }
}
```

### 14.5 测试

```bash
# 启动开发服务器
symfony server:start

# 测试 MCP 端点（初始化）
curl -X GET http://localhost:8000/mcp \
  -H "Accept: application/json, text/event-stream"

# 调用工具
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"hello","arguments":{"name":"世界"}}}'
```

---

*报告生成时间：基于 symfony/mcp-bundle 源码分析（mcp/sdk ^0.4，Symfony ^7.3）*
