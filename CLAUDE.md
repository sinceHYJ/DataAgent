# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 项目概述

DataAgent 是基于 Spring AI Alibaba Graph 构建的企业级智能数据分析 Agent。支持 NL2SQL、Python 深度分析及多维图表报告生成。全栈架构：Spring Boot (WebFlux) 后端 + Vue 3 前端。

## 构建命令

需要 Java 17 和 Maven（推荐使用 `mvnd` 守护进程模式，或 `./mvnw`）。

```bash
make build                # 构建（跳过测试）
make test                 # 运行所有测试
make format-fix           # 自动格式化 Java 代码 (spring-javaformat)
make spotless-apply       # 应用 license 头 + 格式化 (spotless)
make format-check         # 检查代码格式
make checkstyle-check     # 运行 checkstyle
make lint                 # 运行所有 linter

# 启动后端（端口 8065）
cd data-agent-management && ./mvnw spring-boot:run

# 使用 H2 内存数据库启动（无需 MySQL）
cd data-agent-management && ./mvnw spring-boot:run -Dspring-boot.run.profiles=h2

# 启动前端
cd data-agent-frontend && npm install && npm run dev

# 运行单个测试类
./mvnw test -pl data-agent-management -Dtest=ClassName

# 运行单个测试方法
./mvnw test -pl data-agent-management -Dtest=ClassName#methodName
```

## 架构

### 后端 (`data-agent-management/`)

包路径根目录：`com.alibaba.cloud.ai.dataagent`

**分层结构** — Controller → Service（接口 + 实现） → Mapper → Entity

- `controller/` — REST 接口，统一前缀 `/api/*`
- `service/` — 业务逻辑（接口 + 实现类模式）
- `mapper/` — MyBatis Mapper 接口（基于注解，无 XML 文件）
- `entity/` — 数据库实体（Lombok）
- `dto/`、`vo/` — 请求/响应对象
- `workflow/` — 核心 StateGraph 流水线（见下文）
- `connector/` — 多数据库连接器抽象（MySQL、PostgreSQL、Oracle、SQL Server、H2、达梦、Hive）
- `prompt/` — LLM Prompt 模板
- `config/` — Spring 配置类

### StateGraph 工作流流水线

核心 NL2SQL 流水线为 `workflow/` 中的有向图：

```
START → IntentRecognition → EvidenceRecall → QueryEnhance → SchemaRecall → TableRelation
  → FeasibilityAssessment → Planner → PlanExecutor → [SQL/Python/Report 步骤]
  → ReportGenerator → END
```

- `workflow/node/` — 16 个图节点（每个节点处理一个流水线步骤）
- `workflow/dispatcher/` — 11 个边调度器（根据状态在节点间路由）
- `constant/Constant.java` — 所有状态图的 state key 定义在此，为 `public static final String`

### 核心设计模式

- **策略模式** — `DatasourceTypeHandler` 按数据库类型实现、`FusionStrategy`（RRF/加权平均）、`TextSplitter`、`CodePoolExecutorService`（Docker/本地/AI 模拟）
- **工厂模式** — `DynamicModelFactory`、`FileStorageServiceFactory`、`HybridRetrievalStrategyFactory`、`CodePoolExecutorServiceFactory`
- **注册模式** — `AiModelRegistry` 支持运行时模型热切换；通过 AOP 动态代理 `EmbeddingModel` Bean
- **SSE 流式输出** — `GraphController` 通过 Reactor `Sinks.Many<ServerSentEvent<>>` 实现流式推送

### 前端 (`data-agent-frontend/`)

Vue 3 + TypeScript + Vite + Element Plus + ECharts。代码风格规范见 `data-agent-frontend/README-CODE-STYLE.md`。

## 关键技术约束

- **WebFlux，非 Spring MVC** — Controller 返回 `Mono`/`Flux`，全程使用响应式类型
- **MyBatis 仅使用注解** — 无 XML Mapper 文件；所有 SQL 写在 `@Select`/`@Insert`/`@Update`/`@Delete` 注解中
- **构建时自动格式化** — `spring-javaformat` 和 `spotless` 插件在编译时自动执行
- **Apache 2.0 License 头** — 所有源文件必须包含，由 spotless 自动添加
- **配置前缀** — 所有自定义配置项前缀为 `spring.ai.alibaba.data-agent`
- **代码执行器** — Python 执行需要 Docker（或在配置中设置 `code-pool-executor: local` 使用本地执行）
- **端口** — 后端运行在 8065

## 提交规范

Conventional Commits 格式：`type(scope): description`（主题小写，不以大写开头）
类型：`fix`、`feat`、`refactor`、`docs`、`chore`、`perf`、`infra`、`revert`、`release`、`test`、`style`

## 数据库

- 管理库：默认 MySQL（通过环境变量 `DATA_AGENT_DATASOURCE_URL/USERNAME/PASSWORD` 配置），开发环境可用 H2
- Schema 初始化：`sql/schema.sql` + `sql/data.sql`（设置 `DATA_AGENT_DATASOURCE_SQL_INIT=always` 自动初始化）
- 向量存储：默认内存 `SimpleVectorStore`，支持 Elasticsearch/PGVector/Milvus

## 核心依赖

Spring Boot 3.4.8 | Spring AI 1.1.0 | Spring AI Alibaba 1.1.0.0 | MyBatis 3.0.4 | Druid | SpringDoc OpenAPI 2.8.8 | Docker Java 3.5.3 | Langfuse（可选）
