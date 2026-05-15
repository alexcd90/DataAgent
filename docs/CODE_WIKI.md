# DataAgent Code Wiki

本文档面向代码阅读与二次开发，聚焦仓库架构、核心模块职责、关键类/函数、依赖关系、运行方式与配置要点。

相关资料：
- 架构说明（已有文档）：[ARCHITECTURE.md](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/docs/ARCHITECTURE.md)
- 快速开始（已有文档）：[QUICK_START.md](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/docs/QUICK_START.md)
- 开发者指南（已有文档）：[DEVELOPER_GUIDE.md](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/docs/DEVELOPER_GUIDE.md)

## 1. 仓库概览

### 1.1 模块形态

- 根工程是 Maven 聚合工程（`packaging=pom`），当前只聚合后端模块 `data-agent-management`：[pom.xml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/pom.xml#L6-L16)
- 后端是 Spring Boot 应用（WebFlux）：`data-agent-management/`
- 前端是独立 npm 工程（Vue3 + Vite）：`data-agent-frontend/`
- 部署与一键演示：`docker-file/`
- CI/规范：`.github/` + `CI/` + 根 `Makefile`
- 本地技能（Skill）定义：`agent-skills/`

### 1.2 目录结构（高层）

- `data-agent-management/`
  - `src/main/java/com/alibaba/cloud/ai/dataagent/`：后端业务代码（控制器/服务/持久层/AgentRuntime/连接器等）
  - `src/main/resources/`
    - `application.yml`：默认运行配置（端口、业务库、文件存储、技能目录、Langfuse/OTel 等）[application.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/application.yml#L1-L73)
    - `sql/schema.sql`：业务库表结构基线 [schema.sql](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/sql/schema.sql#L1-L240)
    - `prompts/commonagent.md`：默认 Agent 系统提示词模板（由 `PromptLoader` 加载）
- `data-agent-frontend/`
  - `src/`：页面、组件、API services、路由等
  - `vite.config.js`：本地代理 `/api|/nl2sql|/uploads -> http://localhost:8065` [vite.config.js](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-frontend/vite.config.js#L20-L49)
- `docker-file/`
  - `docker-compose.yml`：MySQL（业务库）+ backend + frontend + 模拟数据源（MySQL/Postgres）[docker-compose.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/docker-file/docker-compose.yml#L15-L121)

## 2. 总体架构与核心概念

### 2.1 分层视角（后端）

后端主要分为以下层次（以包/职责划分）：

- API 接入层：`controller/`
  - WebFlux REST + SSE（Server-Sent Events）输出
- 业务服务层：`service/`
  - 管理侧（CRUD、配置、数据准备）与运行侧（Agent 执行、流式输出）分离
- Agent Runtime（AgentScope 集成层）：`agentscope/`
  - 模型、工具、记忆、hook、技能箱（skill box）装配
  - 运行态 registry（取消/中断）与事件发布
- 数据源连接器层：`connector/`
  - 多 DB 类型接入（MySQL/PostgreSQL/Oracle/SQLServer/Hive/H2/达梦等），提供元数据抽取与查询能力
- 持久化层：`mapper/` + `entity/`
  - MyBatis 管理业务库表（agent、session、message、datasource、knowledge 等）
- 基础设施：`config/` + `properties/` + `util/` + `observability/` 等

### 2.2 核心数据对象（业务域）

业务库中的关键实体（详见 schema）：

- 智能体：`agent`（prompt/status/api key 等）[schema.sql](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/sql/schema.sql#L4-L25)
- 数据源：`datasource` + `agent_datasource` + `agent_datasource_tables`（绑定与选表）[schema.sql](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/sql/schema.sql#L101-L240)
- 语义模型：`semantic_model`（字段业务别名/同义词/描述）[schema.sql](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/sql/schema.sql#L50-L71)
- 知识库：`agent_knowledge`、业务知识：`business_knowledge`（召回、向量化状态）[schema.sql](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/sql/schema.sql#L28-L98)
- 对话：`chat_session` + `chat_message`（消息类型、metadata）[schema.sql](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/sql/schema.sql#L191-L224)

## 3. 后端模块详解（data-agent-management）

### 3.1 启动入口与生命周期

- 应用入口：[`DataAgentApplication`](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/DataAgentApplication.java#L22-L29)
  - `@SpringBootApplication` + `@EnableScheduling`

### 3.2 API 设计（控制器一览）

以 `controller/` 包为中心，主要对外能力包括：

- Agent 管理：创建/编辑/发布/下线、API Key 管理（`AgentController`）
- 运行与流式输出：SSE 搜索/分析入口（`DataAgentController`）
- 会话与消息：会话列表、消息列表、报告/Explain（`ChatController`）
- 会话事件流：会话侧边栏更新（`SessionEventController`，SSE）
- 数据源管理：数据源 CRUD、连通性测试、表/字段/逻辑关系（`DatasourceController`）
- Agent-数据源绑定：绑定关系与 schema 初始化触发（`AgentDatasourceController`）
- 语义模型：单条/批量/Excel 导入（`SemanticModelController`）
- 模型配置：添加/更新/激活/测试/就绪检查（`ModelConfigController`）
- 知识与技能：知识库、业务知识、技能管理与绑定（`AgentKnowledgeController` / `BusinessKnowledgeController` / `SkillController` / `AgentSkillController`）
- 文件：上传/读取（`FileUploadController`）
- 异常：统一异常处理（`GlobalExceptionHandler`）

建议从 Swagger UI 对照接口：`/swagger-ui.html`（配置见 [application.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/application.yml#L4-L9)）。

### 3.3 关键链路：SSE 流式运行（/api/stream/search）

这条链路是“前端发起问题 → 运行时 Agent 执行 → 过程/结果推送 SSE → 落库会话与 explain”的主路径。

#### 3.3.1 WebFlux SSE 入口

- SSE 入口方法：[DataAgentController.streamSearch](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/controller/DataAgentController.java#L51-L100)
  - 核心动作：
    - 校验 `agentId` 并检查 `threadId(sessionId)` 归属关系：`chatSessionService.requireSessionForAgent(...)` [DataAgentController.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/controller/DataAgentController.java#L60-L62)
    - 创建 `Sinks.Many` 并调用运行服务：`agentService.graphStreamProcess(sink, request)` [DataAgentController.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/controller/DataAgentController.java#L66-L78)
    - 断开连接触发取消：`stopStreamProcessing(threadId, runtimeRequestId)` [DataAgentController.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/controller/DataAgentController.java#L87-L97)

#### 3.3.2 运行时编排：graphStreamProcess

- 运行入口：[AiAgentRuntimeServiceImpl.graphStreamProcess](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L128-L151)
  - 核心动作：
    - 初始化/补齐 `runtimeRequestId` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L207-L214)
    - 注册运行态：`runtimeRegistry.register(threadId, runtimeRequestId)` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L133-L135)
    - 通过 `AgentRuntimeEventPublisher` 把过程消息推送 SSE（并进行 streamTextTracker 记录）[AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L135-L144)
    - 异步执行 Agent：`Mono.fromCallable(...).subscribeOn(boundedElastic)` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L146-L151)
    - 结束清理状态：`doFinally -> runtimeRegistry.finish(...)` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L146-L150)

#### 3.3.3 取消机制：runtimeRegistry

- 运行态 registry：[`AgentRuntimeRegistry`](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/session/AgentRuntimeRegistry.java#L24-L102)
  - `register()`：创建/重置请求状态 [AgentRuntimeRegistry.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/session/AgentRuntimeRegistry.java#L28-L32)
  - `markRunning()`：记录运行线程 [AgentRuntimeRegistry.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/session/AgentRuntimeRegistry.java#L46-L48)
  - `markCancelled()`：标记取消并 `interrupt()` 运行线程 [AgentRuntimeRegistry.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/session/AgentRuntimeRegistry.java#L34-L44)
  - `finish()`：请求结束后清理状态 [AgentRuntimeRegistry.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/session/AgentRuntimeRegistry.java#L67-L75)

#### 3.3.4 执行核心：executeAgent（装配模型、工具、记忆、模板）

- 关键执行段落位于：[AiAgentRuntimeServiceImpl.executeAgent](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L216-L255)
  - 记忆加载：`agentScopeMemoryFactory.create(request)` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L223-L224)
  - 歧义澄清评估（可阻断执行）：`queryClarifyService.assess(...)` + `shouldBlockExecution()` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L228-L238)
  - 取 Agent 配置 + 取激活模型配置：`resolveManagedAgent(...)` + `getActiveConfigByType(CHAT)` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L239-L241)
  - 工具回调收集：`agentScopeToolkitFactory.getToolCallbacks(agentId)` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L242-L244)
  - 模型构建：`dynamicModelFactory.createChatModel(modelConfig)` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L244-L245)
  - 模板 Agent 执行：`managedAgent.run(new AgentRunContext(...))` [AiAgentRuntimeServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/service/impl/AiAgentRuntimeServiceImpl.java#L246-L254)

### 3.4 Agent 模板：CommonAgent（ReActAgent）

- 默认模板实现：[`CommonAgent`](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/template/CommonAgent.java#L30-L110)
  - 模板类型：`commonagent` [CommonAgent.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/template/CommonAgent.java#L33-L40)
  - 构建 ReActAgent：组装 `sysPrompt / model / toolkit / memory / toolExecutionContext / skillBox / hooks` [CommonAgent.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/template/CommonAgent.java#L43-L70)
  - 运行：把用户 prompt 包装为 `MsgRole.USER` 并 `block(timeout)` [CommonAgent.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/template/CommonAgent.java#L71-L78)

模板注册表：
- [`ManagedAgentRegistry.getRequired`](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/template/ManagedAgentRegistry.java#L30-L41) 默认返回 `commonagent`。

### 3.5 工具系统：Spring AI ToolCallback -> AgentScope Toolkit

- 工具收集与装配：[`AgentScopeToolkitFactory`](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/AgentScopeToolkitFactory.java#L34-L126)
  - 公共工具：从 Spring 容器中收集 `ToolCallback` 与 `ToolCallbackProvider`，并排除 MCP Server 工具（避免循环依赖/重复暴露）[AgentScopeToolkitFactory.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/AgentScopeToolkitFactory.java#L72-L86)
  - Agent 级工具：由 `AgentScopedToolCatalogService.getToolCallbacks(agentId)` 注入 [AgentScopeToolkitFactory.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/AgentScopeToolkitFactory.java#L88-L93)
  - 适配层：把 `ToolCallback` 映射为 AgentScope 工具并注册到 `Toolkit` [AgentScopeToolkitFactory.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/AgentScopeToolkitFactory.java#L51-L60)

### 3.6 歧义澄清：QueryClarifyService

在运行前对 query 做轻量规则评估，必要时阻断执行并要求补充口径：

- 评估入口：[QueryClarifyService.assess](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/QueryClarifyService.java#L59-L109)
  - 维度：时间范围、指标口径、对比对象、排序依据等 [QueryClarifyService.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/QueryClarifyService.java#L74-L97)
  - 高风险（HIGH）会返回澄清问题并阻断：`shouldBlockExecution()` [QueryClarifyService.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/QueryClarifyService.java#L211-L213)

### 3.7 Schema 初始化与向量化（连接器 + VectorStore）

从业务视角，Schema 初始化通常由“Agent 绑定数据源、选择表”触发，用于把表/列元数据与外键关系写入向量库，支撑后续 NL2SQL/检索增强。

典型实现：
- [`SchemaServiceImpl.schema`](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/schema/SchemaServiceImpl.java#L132-L193)
  - 通过 `AccessorFactory` 根据 DB 配置选择具体 `Accessor` [SchemaServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/schema/SchemaServiceImpl.java#L142-L145)
  - 清理旧 schema 向量数据，再抽取外键、表、列元数据 [SchemaServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/schema/SchemaServiceImpl.java#L146-L173)
  - 表数量较大时并行 enrich：`processTablesInParallel`（使用专用线程池）[SchemaServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/schema/SchemaServiceImpl.java#L164-L238)
  - 转为 Document 并写入向量库 [SchemaServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/schema/SchemaServiceImpl.java#L178-L187)

## 4. 关键基础设施与依赖关系

### 4.1 Maven 依赖与版本基线

- 根工程声明版本与依赖管理（Java 17、Spring Boot、Spring AI 等）：[pom.xml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/pom.xml#L22-L70)
- 后端模块核心依赖（摘选）：[data-agent-management/pom.xml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/pom.xml#L21-L220)
  - Spring WebFlux + OpenAPI（WebFlux UI）
  - MyBatis
  - Spring AI MCP Server（WebFlux）
  - AgentScope（io.agentscope）
  - 多种数据库驱动（mysql/postgresql/ojdbc/mssql/h2/hive/达梦等）
  - OpenTelemetry

### 4.2 运行时组件依赖图（概念级）

以 SSE 链路为中心，核心依赖关系可抽象为：

- `DataAgentController` -> `AgentService(AiAgentRuntimeServiceImpl)`
  - -> `AgentRuntimeRegistry`（取消与并发状态）
  - -> `ModelConfigDataService`（取激活模型配置）
    - -> `DynamicModelFactory`（按 OpenAI-compatible 协议构建 ChatModel/EmbeddingModel）
  - -> `AgentScopeToolkitFactory`（工具集合）
    - -> `AgentScopedToolCatalogService`（按 agentId 注入 agent-scoped tools）
  - -> `AgentScopeMemoryFactory` + `AgentScopeNativeSessionService`（会话记忆）
  - -> `ManagedAgentRegistry` -> `CommonAgent`（ReActAgent 模板）
  - -> `AnswerTraceExplainStore/Tracer`（可观测与 explain）
  - -> `ChatMessageService/ChatSessionService`（落库会话与 explain）

### 4.3 动态模型工厂：OpenAI-Compatible 统一接入

[`DynamicModelFactory`](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/aimodelconfig/DynamicModelFactory.java#L46-L165) 的核心意图是：

- 统一使用 `OpenAiChatModel`/`OpenAiEmbeddingModel`，通过 `baseUrl` 实现多厂商兼容 [DynamicModelFactory.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/aimodelconfig/DynamicModelFactory.java#L51-L83)
- 支持同步/异步代理（HTTP proxy + 可选 Basic Auth）[DynamicModelFactory.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/aimodelconfig/DynamicModelFactory.java#L118-L163)

### 4.4 自动配置：VectorStore 兜底 + EmbeddingModel 动态代理 + DB 专用线程池

[`DataAgentConfiguration`](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/config/DataAgentConfiguration.java#L74-L235) 提供了若干关键默认行为：

- VectorStore 兜底：当未引入其它 VectorStore starter 时，默认使用 `SimpleVectorStore` [DataAgentConfiguration.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/config/DataAgentConfiguration.java#L111-L117)
- ToolCallbackResolver：聚合 Spring Bean ToolCallback，并排除 MCP Server tools [DataAgentConfiguration.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/config/DataAgentConfiguration.java#L147-L163)
- EmbeddingModel 代理：通过 AOP TargetSource 每次动态获取 registry 中的 embeddingModel [DataAgentConfiguration.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/config/DataAgentConfiguration.java#L165-L206)
- DB 并行线程池：命名 `db-operation-*`，用于 schema 等场景的并行 DB 元数据处理 [DataAgentConfiguration.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/config/DataAgentConfiguration.java#L208-L235)

## 5. 配置与运行方式

### 5.1 后端配置要点（application.yml）

端口与 OpenAPI：
- `server.port=8065` [application.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/application.yml#L1-L3)
- Swagger UI：`/swagger-ui.html` [application.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/application.yml#L4-L9)

业务库（DataAgent 自己的管理库，不是分析数据源）：
- `spring.datasource.url/username/password` 支持环境变量覆盖：`DATA_AGENT_DATASOURCE_*` [application.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/application.yml#L10-L23)
- SQL 初始化开关：`DATA_AGENT_DATASOURCE_SQL_INIT`（默认 `never`）[application.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/application.yml#L18-L23)

技能与文件：
- 本地技能目录：`spring.ai.alibaba.data-agent.skills.local-path=./agent-skills` [application.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/application.yml#L55-L57)
- 文件存储：默认本地 `uploads`，并通过 `url-prefix=/uploads` 暴露 [application.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/resources/application.yml#L42-L48)

### 5.2 本地启动（推荐用于开发）

后端（从仓库根目录执行，使用 Maven Wrapper）：

```bash
cd /Users/mac/soft-devp/soft-project/java/git-project/DataAgent
./mvnw -pl data-agent-management spring-boot:run
```

前端：

```bash
cd /Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-frontend
npm install
npm run dev
```

前端开发代理（避免跨域）：
- `/api|/nl2sql|/uploads -> http://localhost:8065` [vite.config.js](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-frontend/vite.config.js#L27-L44)

### 5.3 Docker 一键启动（演示/部署）

Compose 入口在 `docker-file/`，并通过环境变量注入 LLM Key：

```bash
cd /Users/mac/soft-devp/soft-project/java/git-project/DataAgent/docker-file
AI_DASHSCOPE_API_KEY=你的key docker compose up --build
```

关键环境变量示例（compose）：
- 业务库连接：`DATA_AGENT_DATASOURCE_URL/USERNAME/PASSWORD` [docker-compose.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/docker-file/docker-compose.yml#L42-L49)
- 业务库初始化：`DATA_AGENT_DATASOURCE_SQL_INIT=always` [docker-compose.yml](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/docker-file/docker-compose.yml#L45-L47)

访问：
- 前端：`http://localhost:3000`
- 后端：`http://localhost:8065`

## 6. 前端模块（data-agent-frontend）

### 6.1 构建与运行

脚本与依赖概览：[package.json](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-frontend/package.json#L1-L45)

- `npm run dev`：开发服务（默认 3000）
- `npm run build`：生产构建
- `npm run lint:check` / `npm run format:check`：质量检查

### 6.2 与后端的契约

- 主要通过 `/api/**` 调用后端控制器
- 流式输出通常通过 EventSource/SSE 接收（对应后端 `TEXT_EVENT_STREAM`）
- 上传/图片等资源统一走 `/uploads/**` 代理回后端

## 7. 常见扩展点（开发落点）

- 新增一个“工具（Tool）”：
  - 以 Spring AI 的 `ToolCallback` 或 `ToolCallbackProvider` 形式注册到 Spring 容器
  - 工具会被 `AgentScopeToolkitFactory` 自动收集并映射为 AgentScope `Toolkit` [AgentScopeToolkitFactory.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/AgentScopeToolkitFactory.java#L72-L93)
- 新增一个 Agent 模板：
  - 实现 `ManagedAgent` 并声明 `getAgentType()`
  - 需要扩展 `ManagedAgentRegistry` 的选择策略（当前 `getRequired()` 固定返回 `commonagent`）[ManagedAgentRegistry.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/template/ManagedAgentRegistry.java#L35-L41)
- 替换向量库（VectorStore）：
  - 通过 Maven 引入 Spring AI 对应 VectorStore starter；未配置时会回退到 `SimpleVectorStore` [DataAgentConfiguration.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/config/DataAgentConfiguration.java#L111-L117)
- 支持更多模型厂商：
  - 在“OpenAI-compatible baseUrl”范式下通常无需新增 SDK，只需配置 `baseUrl/modelName/apiKey`（由 `DynamicModelFactory` 构建）[DynamicModelFactory.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/aimodelconfig/DynamicModelFactory.java#L54-L83)

## 8. 工程化与 CI（你会在本仓库看到的约束）

- GitHub Actions
  - 后端：构建、测试、checkstyle、format-check 等（见 `.github/workflows/*`）
  - 前端：`lint:check`、`format:check`、`unused`、`build`（见 `.github/workflows/frontend-check.yml`）
- Make 体系
  - 根目录通过 `Makefile` 统一转发到 `CI/make/*.mk`，CI 中常用 `make test / make build / make checkstyle-check`

## 9. 构建与验证（开发者常用命令）

后端（根目录执行）：

```bash
cd /Users/mac/soft-devp/soft-project/java/git-project/DataAgent
./mvnw -pl data-agent-management test
./mvnw -pl data-agent-management -DskipTests package
```

前端：

```bash
cd /Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-frontend
npm install
npm run type-check
npm run lint:check
npm run build
```

## 10. 常见问题定位（读代码时的“抓手”）

- SSE 没有输出/中断
  - 先看 SSE 过滤逻辑是否丢弃了空文本：`sink.asFlux().filter(...)` [DataAgentController.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/controller/DataAgentController.java#L79-L84)
  - 再看取消逻辑是否提前触发：`doOnCancel -> stopStreamProcessing` [DataAgentController.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/controller/DataAgentController.java#L87-L92)
  - 最后看运行端是否被 `interrupt`：`markCancelled -> runningThread.interrupt()` [AgentRuntimeRegistry.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/session/AgentRuntimeRegistry.java#L34-L44)
- 问题被“澄清阻断”
  - 入口规则：`QueryClarifyService.assess` [QueryClarifyService.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/QueryClarifyService.java#L59-L109)
  - 命中 HIGH 时会要求补充口径：`shouldBlockExecution()` [QueryClarifyService.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/agentscope/runtime/QueryClarifyService.java#L211-L213)
- Schema 初始化慢/失败
  - 并行度与线程池：`processTablesInParallel` [SchemaServiceImpl.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/service/schema/SchemaServiceImpl.java#L164-L238) + `dbOperationExecutor` [DataAgentConfiguration.java](file:///Users/mac/soft-devp/soft-project/java/git-project/DataAgent/data-agent-management/src/main/java/com/alibaba/cloud/ai/dataagent/config/DataAgentConfiguration.java#L208-L235)
