# Agent 模块分析报告

## 模块概述

Agent 模块是 Symfony AI 框架的核心执行引擎，负责协调 AI 平台调用、工具执行与消息处理的完整生命周期。该模块实现了一套灵活的处理器管道架构：在调用 AI 平台之前，`InputProcessor` 链负责预处理输入（注入系统提示、切换模型、加载记忆等）；在收到平台响应之后，`OutputProcessor` 链负责后处理输出（检测工具调用、循环执行工具、汇总来源等）。这种管道设计使得功能高度可组合，开发者可以按需插拔各种处理器。

工具调用（Tool Calling）是 Agent 模块最重要的特性之一，由 `AgentProcessor`（同时实现 `InputProcessorInterface` 和 `OutputProcessorInterface`）与 `Toolbox` 协作完成。当 AI 平台返回 `ToolCallResult` 时，`AgentProcessor` 会自动提取工具调用列表、调用 `Toolbox::execute()` 执行对应工具、将结果追加到消息历史，然后再次调用 Agent，直到平台返回非工具调用结果为止——这就是所谓的「迭代循环（Agent Loop）」。`FaultTolerantToolbox` 则在此基础上提供容错包装，将执行异常转换为可供 LLM 理解的错误消息。

Memory 子模块通过 `MemoryInputProcessor` 将历史对话或向量检索到的相关片段注入系统提示，从而赋予 Agent 跨会话记忆能力。MultiAgent 子模块实现了多 Agent 编排，由 `MultiAgent` 担任调度器，通过 `Handoff` 规则将请求路由到最适合的子 Agent。Bridge 子目录提供了一批开箱即用的工具实现，覆盖网页搜索、地图、天气、文件系统、相似度检索等常见场景。

---

## 架构总览（文本架构图）

```
┌──────────────────────────────────────────────────────────────────────┐
│                           AgentInterface                             │
│                   call(MessageBag, options): ResultInterface         │
└────────────────────────────┬─────────────────────────────────────────┘
                             │ implements
          ┌──────────────────┼──────────────────────┐
          │                  │                      │
    ┌─────▼──────┐   ┌───────▼────────┐   ┌────────▼────────┐
    │   Agent    │   │   MockAgent    │   │   MultiAgent    │
    │ (主实现)   │   │  (测试用 Mock) │   │  (多 Agent 编排)│
    └─────┬──────┘   └────────────────┘   └─────────────────┘
          │
          │  Agent.call() 执行流程
          │
    ┌─────▼─────────────────────────────────────────────────┐
    │  1. 构建 Input(model, MessageBag, options)             │
    │  2. 遍历 InputProcessors                              │
    │     ├─ ModelOverrideInputProcessor   (覆盖模型)        │
    │     ├─ SystemPromptInputProcessor    (注入系统提示)    │
    │     ├─ MemoryInputProcessor          (注入记忆)        │
    │     └─ AgentProcessor.processInput  (注入工具列表)     │
    │  3. platform.invoke(model, messages, options)          │
    │  4. 构建 Output(model, result, MessageBag, options)    │
    │  5. 遍历 OutputProcessors                             │
    │     └─ AgentProcessor.processOutput (处理工具调用循环) │
    │         ├─ 若 result 是 ToolCallResult:               │
    │         │   a. 执行所有 ToolCall via Toolbox          │
    │         │   b. 将结果追加到 MessageBag                │
    │         │   c. 再次调用 agent.call()                  │
    │         │   d. 循环直到非 ToolCallResult              │
    │         └─ 若 result 是 StreamResult: 注册 StreamListener
    └───────────────────────────────────────────────────────┘

    Platform 模块
    ┌──────────────────────────┐
    │   PlatformInterface      │
    │   invoke() → ResultInterface
    │   ├─ TextResult          │
    │   ├─ ToolCallResult      │
    │   └─ StreamResult        │
    └──────────────────────────┘

    Toolbox 子系统
    ┌──────────────────────────────────────────────────────┐
    │  ToolboxInterface                                    │
    │  ├─ Toolbox (标准实现，含事件分发)                    │
    │  └─ FaultTolerantToolbox (容错包装)                   │
    │                                                      │
    │  ToolFactoryInterface                                │
    │  ├─ ReflectionToolFactory  (#[AsTool] 注解解析)      │
    │  ├─ MemoryToolFactory      (编程式注册)              │
    │  └─ ChainFactory           (工厂链，依次尝试)         │
    │                                                      │
    │  ToolCallArgumentResolverInterface                   │
    │  └─ ToolCallArgumentResolver (反射 + Serializer 反序列化)
    └──────────────────────────────────────────────────────┘

    Memory 子系统
    ┌──────────────────────────────────────────────┐
    │  MemoryProviderInterface                     │
    │  ├─ StaticMemoryProvider  (静态字符串记忆)    │
    │  └─ EmbeddingProvider     (向量检索动态记忆)  │
    │  MemoryInputProcessor  (InputProcessor 实现)  │
    └──────────────────────────────────────────────┘

    MultiAgent 子系统
    ┌──────────────────────────────────────────────┐
    │  MultiAgent (AgentInterface 实现)             │
    │  ├─ orchestrator: AgentInterface              │
    │  ├─ handoffs: Handoff[]                       │
    │  └─ fallback: AgentInterface                  │
    │  Handoff (to: AgentInterface, when: string[]) │
    │  Handoff/Decision (agentName, reasoning)      │
    └──────────────────────────────────────────────┘

    Bridge 工具集
    ┌──────────────────────────────────────────────────────────────────┐
    │  Brave / SerpApi / Tavily / Ollama  →  网页搜索                  │
    │  Wikipedia                          →  百科检索                  │
    │  OpenMeteo                          →  天气查询                  │
    │  Mapbox                             →  地图 / 地理编码            │
    │  Firecrawl / Scraper                →  网页内容抓取              │
    │  Filesystem                         →  文件系统操作（沙箱化）    │
    │  SimilaritySearch                   →  向量相似度检索            │
    │  YoutubeTranscriber                 →  YouTube 字幕提取          │
    │  Clock                              →  当前时间查询              │
    └──────────────────────────────────────────────────────────────────┘
```

---

## 核心概念

### Agent Loop（迭代循环）
`Agent::call()` 本身是单次调用，但 `AgentProcessor`（作为 OutputProcessor）实现了迭代循环：只要平台返回 `ToolCallResult`，它就会执行工具并重新调用 Agent，直到收到文本或其他非工具结果。可通过 `maxToolCalls` 参数限制最大迭代次数，超出时抛出 `MaxIterationsExceededException`。

### InputProcessor 与 OutputProcessor
这两个接口定义了「在调用平台前/后对消息和结果进行变换」的扩展点。处理器可通过 `#[AsInputProcessor]`、`#[AsOutputProcessor]` 注解与 Symfony DI 容器集成，支持按 Agent 服务 ID 绑定或全局绑定，并可设置优先级。

### Toolbox（工具箱）
工具箱负责工具的注册、元数据管理和执行。工具通过 `#[AsTool]` 注解描述名称、描述和入口方法，`ReflectionToolFactory` 解析注解并配合 `Platform` 模块的 JSON Schema 工厂生成工具参数描述。`ToolCallArgumentResolver` 则利用反射和 Symfony Serializer 对 LLM 传来的 JSON 参数进行类型转换和反序列化。

### Tool Calling（工具调用）
LLM 在需要外部信息时，会返回包含一个或多个 `ToolCall` 对象的 `ToolCallResult`。`AgentProcessor` 捕获该结果，调用 `Toolbox::execute()` 逐一执行，再将结果以 `ToolCallMessage` 形式追回消息历史，最终重新请求平台。

### Memory（记忆）
`MemoryInputProcessor` 在每次调用前从一组 `MemoryProviderInterface` 加载相关记忆片段，并将它们注入系统提示。`StaticMemoryProvider` 提供固定文本记忆；`EmbeddingProvider` 利用向量存储（`Store` 模块）根据用户消息进行语义检索，实现动态长期记忆。

### Multi-Agent 多 Agent 编排
`MultiAgent` 本身也实现了 `AgentInterface`，对外表现为单一 Agent。内部，它将用户消息发送给一个「编排者（orchestrator）」Agent，请求其以结构化输出（`Decision` DTO，含 `agentName` 与 `reasoning`）的形式选择最合适的子 Agent；之后将原始消息转发给选定的子 Agent。若无匹配，则交由 `fallback` Agent 处理。

### Bridge 工具集
`Bridge/` 目录下的每个工具都标注了 `#[AsTool]`，并可直接注册到 `Toolbox`。多数工具实现了 `HasSourcesInterface`，在执行后将引用来源（`Source`）保存到 `SourceCollection`，`AgentProcessor` 在启用 `includeSources` 选项时会将这些来源附加到最终结果的元数据中。

### 与 Platform 模块的关系
Agent 模块通过 `PlatformInterface` 与 Platform 模块解耦：`Agent` 持有一个 `PlatformInterface` 实例，通过 `invoke(model, messages, options)` 发起调用，接收标准化的 `ResultInterface`（`TextResult`、`ToolCallResult`、`StreamResult`）。工具的参数 JSON Schema 由 Platform 模块的 `Contract\JsonSchema\Factory` 生成。
