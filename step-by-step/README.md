# Step-by-Step 业务场景实战指南

本目录包含基于 Symfony AI 各模块的真实业务场景教程。每个文档模拟一个完整的业务场景，从浅到深，step by step 展示如何使用对应模块解决实际问题。

## 模块对照表

| 模块 | 功能 | 涉及文档 |
|------|------|---------|
| **Platform** | AI 平台连接与调用 | 所有文档 |
| **Chat** | 对话持久化管理 | 02, 10, 11, 13, 14, 17 |
| **Agent** | 智能体 + 工具调用 | 03, 04, 05, 08, 09, 10, 11, 12, 13, 15, 16, 17, 18 |
| **Store** | 向量存储与检索 | 04, 08, 10, 18 |
| **Agent MultiAgent** | 多智能体编排 | 05, 11, 18 |
| **StructuredOutput** | 结构化数据输出 | 06, 11, 12, 13, 14, 16, 18 |
| **多模态 Content** | 图片/音频/PDF 处理 | 07, 12 |
| **Agent Memory** | 长期记忆 | 08, 10, 14 |
| **Agent Bridge（搜索/维基）** | Brave、Wikipedia、Tavily 搜索 | 03, 09, 10, 16 |
| **Agent Bridge Filesystem** | 文件系统操作 | 13, 17 |
| **Agent Bridge Mapbox** | 地理编码 | 15 |
| **Agent Bridge OpenMeteo** | 天气查询 | 15 |
| **Agent Bridge Clock** | 当前时间 | 03, 09, 15, 16 |

## 教程列表（从浅到深）

### 入门：单一模块

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 01 | [基础问答聊天机器人](./01-basic-chatbot.md) | 产品官网 AI 客服 | Platform | ⭐ |
| 02 | [多轮对话与会话持久化](./02-multi-turn-conversation.md) | 在线客服（跨请求保持对话） | Platform + Chat | ⭐⭐ |
| 06 | [结构化数据提取](./06-structured-data-extraction.md) | 表单自动填充 / 简历解析 | Platform + StructuredOutput | ⭐⭐ |
| 07 | [多模态内容理解](./07-multimodal-content-understanding.md) | 图片审核 / 文档OCR / 语音转写 | Platform + Content 类 | ⭐⭐ |

### 进阶：双模块组合

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 03 | [工具增强型 AI 助手](./03-tool-augmented-assistant.md) | 运维助手（查时间/搜百科/查订单） | Platform + Agent | ⭐⭐ |
| 04 | [RAG 知识库问答](./04-rag-knowledge-base.md) | 企业文档问答系统 | Platform + Agent + Store | ⭐⭐⭐ |
| 05 | [多智能体客服路由](./05-customer-service-multi-agent.md) | 多部门客服自动分发 | Platform + Agent MultiAgent | ⭐⭐⭐ |
| 08 | [带长期记忆的个性化助手](./08-agent-with-memory.md) | 健身教练 / 学习辅导 | Platform + Agent + Store | ⭐⭐⭐ |
| 09 | [联网研究助手](./09-web-research-assistant.md) | 市场调研 / 竞品分析 | Platform + Agent + 工具桥接 | ⭐⭐⭐ |
| 14 | [多语言内容翻译与本地化](./14-multilingual-translation.md) | 产品国际化 / 术语一致性 | Platform + Chat + StructuredOutput + Memory | ⭐⭐⭐ |

### 高级：多模块随机组合

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 11 | [智能邮件分类与自动回复](./11-smart-email-triage-and-auto-reply.md) | 企业邮件自动分拣与回复 | Platform + StructuredOutput + MultiAgent + Chat | ⭐⭐⭐ |
| 12 | [内容安全审核流水线](./12-content-moderation-pipeline.md) | UGC 内容多维度安全审核 | Platform + StructuredOutput + Agent + 多模态 | ⭐⭐⭐ |
| 13 | [AI 代码审查助手](./13-ai-code-review-assistant.md) | 代码质量审查 / Bug 检测 | Platform + Agent + Filesystem + StructuredOutput + Chat | ⭐⭐⭐ |
| 15 | [智能天气出行助手](./15-weather-travel-planner.md) | 旅行规划 / 天气查询 | Platform + Agent + Mapbox + OpenMeteo + Clock | ⭐⭐⭐ |
| 16 | [自动化报告生成](./16-automated-report-generation.md) | 行业周报 / 竞品监控报告 | Platform + Agent + Tavily + StructuredOutput | ⭐⭐⭐⭐ |
| 17 | [AI 智能文件管理助手](./17-ai-file-management-assistant.md) | 文件整理归档 / 内容索引 | Platform + Agent + Filesystem + Chat | ⭐⭐⭐ |

### 综合：全模块整合

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 10 | [完整 RAG 聊天系统](./10-rag-chat-with-persistence.md) | 企业级智能客服平台 | 全部模块 | ⭐⭐⭐⭐ |
| 18 | [文档智能分类与路由](./18-document-classification-routing.md) | 企业文档自动处理管道 | Platform + StructuredOutput + MultiAgent + Store | ⭐⭐⭐⭐ |

## 阅读建议

1. **完全新手**：按 01 → 02 → 03 → 04 的顺序阅读
2. **有 LLM 经验**：直接看你感兴趣的业务场景
3. **快速上手**：先看 01（理解基础 API），再看 10（理解全貌）
4. **关注特定工具**：看模块对照表，找到使用你关注的模块的文档
5. **真实项目参考**：11-18 是从 GitHub/行业调研中提炼的 2025 年热门 LLM 应用场景
