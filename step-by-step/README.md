# Step-by-Step 业务场景实战指南

本目录包含基于 Symfony AI 各模块的真实业务场景教程。每个文档模拟一个完整的业务场景，从浅到深，step by step 展示如何使用对应模块解决实际问题。

## 模块对照表

| 模块 | 功能 | 涉及文档 |
|------|------|---------|
| **Platform** | AI 平台连接与调用 | 所有文档 |
| **Chat** | 对话持久化管理 | 02, 10, 11, 13, 14, 17, 23, 26, 33 |
| **Agent** | 智能体 + 工具调用 | 03, 04, 05, 08, 09, 10, 11, 12, 13, 15, 16, 17, 18, 19, 20, 22, 23, 24, 25, 27, 29, 30, 33, 34, 35 |
| **Store** | 向量存储与检索 | 04, 08, 10, 18, 24, 25, 26, 27, 29, 30, 32, 35 |
| **Store HybridQuery** | 混合搜索（语义+关键词） | 32 |
| **Store Reranking** | 语义重排序 | 24 |
| **Store Document Loader (RSS)** | RSS 订阅源加载 | 30 |
| **Store ChunkTransformer** | 文档分段 | 26 |
| **Agent MultiAgent** | 多智能体编排 | 05, 11, 18, 23 |
| **Agent Subagent** | Agent 间委托 | 23 |
| **StructuredOutput** | 结构化数据输出 | 06, 11, 12, 13, 14, 16, 18, 19, 21, 22, 24, 25, 27, 29, 30, 31, 32, 34, 35 |
| **Image Content** | 图片输入/生成 | 07, 12, 19, 21, 22 |
| **Audio Content** | 音频输入/TTS/STT | 07, 20, 21, 22, 34 |
| **Video Content** | 视频输入/分析 | 21 |
| **Document Content** | PDF/文档处理 | 07, 25, 31 |
| **Agent Memory** | 长期记忆 | 08, 10, 14, 23 |
| **Agent Bridge（搜索/维基）** | Brave、Wikipedia、Tavily 搜索 | 03, 09, 10, 16, 23, 27, 35 |
| **Agent Bridge Firecrawl** | 深度网页抓取 | 27 |
| **Agent Bridge YouTube** | YouTube 视频字幕获取 | 26 |
| **Agent Bridge Filesystem** | 文件系统操作 | 13, 17, 22, 23 |
| **Agent Bridge SimilaritySearch** | 向量相似度搜索工具 | 26, 29, 35 |
| **Agent Bridge Mapbox** | 地理编码 | 15 |
| **Agent Bridge OpenMeteo** | 天气查询 | 15 |
| **Agent Bridge Clock** | 当前时间 | 03, 09, 15, 16, 20, 23 |
| **Toolbox Events** | 工具调用生命周期事件 | 23 |
| **Platform Bridge Failover** | 多平台故障转移 | 28 |
| **Platform Bridge Cache** | 响应缓存 | 28 |
| **Platform (ElevenLabs)** | 语音 STT/TTS | 20, 22, 34 |
| **Platform (Gemini)** | 多模态视觉/音频/视频 | 20, 21, 34 |
| **Platform (Anthropic)** | Claude 模型备用 | 28 |
| **Platform (Ollama)** | 本地模型私有化部署 | 28, 29 |
| **Agent StreamListener** | 流式响应中处理工具调用 | 33 |

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

### 高级：多模块组合

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 11 | [智能邮件分类与自动回复](./11-smart-email-triage-and-auto-reply.md) | 企业邮件自动分拣与回复 | Platform + StructuredOutput + MultiAgent + Chat | ⭐⭐⭐ |
| 12 | [内容安全审核流水线](./12-content-moderation-pipeline.md) | UGC 内容多维度安全审核 | Platform + StructuredOutput + Agent + 多模态 | ⭐⭐⭐ |
| 13 | [AI 代码审查助手](./13-ai-code-review-assistant.md) | 代码质量审查 / Bug 检测 | Platform + Agent + Filesystem + StructuredOutput + Chat | ⭐⭐⭐ |
| 15 | [智能天气出行助手](./15-weather-travel-planner.md) | 旅行规划 / 天气查询 | Platform + Agent + Mapbox + OpenMeteo + Clock | ⭐⭐⭐ |
| 16 | [自动化报告生成](./16-automated-report-generation.md) | 行业周报 / 竞品监控报告 | Platform + Agent + Tavily + StructuredOutput | ⭐⭐⭐⭐ |
| 17 | [AI 智能文件管理助手](./17-ai-file-management-assistant.md) | 文件整理归档 / 内容索引 | Platform + Agent + Filesystem + Chat | ⭐⭐⭐ |

### 多模态应用

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 19 | [AI 图片生成与编辑流水线](./19-image-generation-and-editing-pipeline.md) | 电商配图生成 / 视觉评审 | Platform + Image + Agent + StructuredOutput | ⭐⭐⭐ |
| 20 | [语音交互 AI 助手](./20-voice-interactive-assistant.md) | 语音客服 / 语音导航 | Platform + Audio + ElevenLabs STT/TTS + Agent | ⭐⭐⭐ |
| 21 | [视频内容分析与总结](./21-video-content-analysis.md) | 视频审核 / 教学摘要 | Platform + Video + Gemini + StructuredOutput | ⭐⭐⭐ |
| 22 | [多模态内容生产流水线](./22-multimodal-content-creation-pipeline.md) | 营销内容自动化 | Platform + Image + Audio + Agent + Filesystem | ⭐⭐⭐⭐ |

### 数据驱动应用

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 24 | [智能客户反馈分析](./24-customer-feedback-analysis.md) | NPS/评价分析 / 趋势发现 | Platform + StructuredOutput + Store + Reranking | ⭐⭐⭐ |
| 25 | [AI 智能招聘筛选](./25-ai-recruitment-screening.md) | 简历解析 / 候选人匹配 | Platform + Document + StructuredOutput + Store | ⭐⭐⭐ |
| 26 | [YouTube 视频知识库](./26-youtube-video-knowledge-base.md) | 视频课程智能搜索 | Platform + YouTube Bridge + Store + Chat | ⭐⭐⭐ |
| 27 | [竞品情报监控系统](./27-competitor-intelligence-monitoring.md) | 竞品追踪 / 功能对比 | Platform + Firecrawl + Tavily + StructuredOutput + Store | ⭐⭐⭐⭐ |

### 架构与基础设施

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 28 | [高可用 AI 服务架构](./28-high-availability-ai-architecture.md) | 故障转移 / 缓存 / 成本优化 | Failover + Cache + 多平台 | ⭐⭐⭐ |
| 29 | [本地模型私有化部署](./29-local-model-private-deployment.md) | 数据隐私 / 离线 AI | Ollama + Agent + Store (SQLite) | ⭐⭐⭐ |
| 33 | [实时流式对话应用](./33-realtime-streaming-chat.md) | SSE 打字机效果 / 流式工具 | Platform Streaming + Agent + Chat | ⭐⭐⭐ |

### 文档与知识智能

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 30 | [RSS 新闻聚合与智能摘要](./30-rss-news-aggregation.md) | 行业监控 / 每日简报 | RSS Loader + Agent + StructuredOutput + Store | ⭐⭐⭐ |
| 31 | [智能合同审查系统](./31-contract-review-system.md) | 条款提取 / 风险识别 | Document + StructuredOutput + Agent + Store | ⭐⭐⭐⭐ |
| 32 | [电商智能推荐引擎](./32-ecommerce-recommendation-engine.md) | 商品推荐 / 搭配推荐 | Store + HybridQuery + Reranking + StructuredOutput | ⭐⭐⭐ |
| 34 | [AI 会议纪要助手](./34-meeting-minutes-assistant.md) | 录音转纪要 / 行动项提取 | Audio STT + StructuredOutput + Agent | ⭐⭐⭐ |
| 35 | [知识图谱构建与查询](./35-knowledge-graph-construction.md) | 实体关系提取 / 图谱查询 | StructuredOutput + Store + Agent + Wikipedia | ⭐⭐⭐⭐ |

### 综合：全模块整合

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 10 | [完整 RAG 聊天系统](./10-rag-chat-with-persistence.md) | 企业级智能客服平台 | 全部模块 | ⭐⭐⭐⭐ |
| 18 | [文档智能分类与路由](./18-document-classification-routing.md) | 企业文档自动处理管道 | Platform + StructuredOutput + MultiAgent + Store | ⭐⭐⭐⭐ |

### 终极：CrewAI 风格多智能体协作

| # | 文档 | 场景 | 核心模块 | 复杂度 |
|---|------|------|---------|--------|
| 23 | [PHP 版 CrewAI：协作智能体团队](./23-crewai-style-agent-team.md) | 多角色 Agent 协作 | 全部模块 + Subagent + Events + Messenger | ⭐⭐⭐⭐⭐ |

## 阅读建议

1. **完全新手**：按 01 → 02 → 03 → 04 的顺序阅读
2. **有 LLM 经验**：直接看你感兴趣的业务场景
3. **快速上手**：先看 01（理解基础 API），再看 10（理解全貌）
4. **关注特定工具**：看模块对照表，找到使用你关注的模块的文档
5. **真实项目参考**：11-18 是企业常见 LLM 应用场景
6. **多模态应用**：19-22 覆盖图片、语音、视频等多媒体场景
7. **数据驱动**：24-27 覆盖反馈分析、招聘、知识库、竞品监控
8. **架构设计**：28-29, 33 覆盖高可用、本地部署、流式对话
9. **文档智能**：30-32, 34-35 覆盖新闻聚合、合同审查、推荐、会议、知识图谱
10. **终极挑战**：23 是最高级场景 — PHP 版 CrewAI 多智能体协作团队
