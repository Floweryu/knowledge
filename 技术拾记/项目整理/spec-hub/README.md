# SpecHub 项目文档

针对两个仓库的功能、技术实现与技术难点分析，以及配套的简历素材。

分析对象：

| 仓库 | 定位 | 技术栈 | 规模 |
|------|------|--------|------|
| `spec-hub/spec-portal-service` | AI 原生需求交付平台后端 | Java / Spring Boot / LangChain4j / Redis / MySQL / ES / NebulaGraph / K8s | 769 模块、928 类 |
| `llm-translation/spechub-feishu-bot` | SpecHub 的飞书轻量运维入口 | Python / Flask / lark-oapi / httpx / SQLite | 单包 7 模块 |

## 文档索引

| 文档 | 内容 | 用途 |
|------|------|------|
| [01-项目功能与技术实现分析](01-项目功能与技术实现分析.md) | 两个仓库的产品功能、架构分层、十个核心技术实现、16 项技术难点、工程规范亮点、协作关系 | 理解项目全貌 |
| [02-简历项目描述](02-简历项目描述.md) | 精简版（两个侧重方向）、标准版（STAR 结构）、详述版（三个关键决策）、技术点速查表、面试高频追问准备 | 直接取用写简历 / 面试准备 |
| [03-技术点详解](03-技术点详解.md) | 按主题深挖：流式架构、分布式并发、AI 工程、RAG 检索、协议集成、安全设计、Python Bot、可复用设计范式 | 面试深挖时展开 |

## 核心技术点速览

**流式架构** — Redis ZSet 中转 SSE（LLM 线程只写、Poller 只读），支撑断线续接、多端订阅、多实例扩展；双层 SSE 解析（execd 帧套 Claude stream-json）；飞书 CardKit 单飞 + 快照折叠 + 序号防乱序。

**分布式与并发** — 会话级非阻塞锁（`SET NX EX` + value 比对释放，故障时保守拒绝）；交付状态机四源事件归一化 + flowId 级锁串行；Topic 合并召回并行 / 决策串行。

**AI 工程** — Plan-first 协议约束 LLM 输出可复现（Planner 独占发现权、Generator 无 Drop 权限、ID 由程序分配）；会话压缩三级 token 估算降级；tool_call 悬挂自动补合成 result；蒸馏九阶段 + outbox 中断恢复。

**RAG 检索** — 十阶段管道（BM25 + KNN 双路 RRF 融合 → Nebula 双向 BFS 图扩展 → LLM 语义筛选 → Cross-Encoder 重排）；Stage 三档降级策略（ABORT / SKIP_ON_FAILURE / FALLBACK）+ 断路器 + 规则评分兜底；自适应 Stage 同 Bean 双次复用。

**协议与集成** — MCP 动态挂载（Token 三级优先级、Client 按 workspace 复用、全链路软降级）；交付 MCP Server 让沙箱内 Claude 反向驱动后端状态机；A2A JSON-RPC 标准化对外开放。

**安全** — 沙箱内 LLM 视为不可信主体（不透明凭证换可信上下文、路径校验防遍历）；凭证 value 绝不入日志。

## 数据来源与校验

初稿基于 CodeGraph 知识库（代码图谱查询、源码片段、语义检索）与仓库内设计文档（`openspec/` 下的 proposal / design / spec，以及 `2026-08-03-knowledge-search-pipeline-design.md`、`2026-08-05-neura-topic-merge-design.md`）。

随后两个仓库已 clone 到本地并逐条校验：

| 仓库 | 本地路径 | 校验时的 commit |
|------|----------|----------------|
| spec-portal-service | `~/Workspace/spec-portal-service` | `2803a338`（main，2026-08-27） |
| spechub-feishu-bot | `~/Workspace/spechub-feishu-bot` | `c4a766e`（main，2026-08-19） |

实测规模：`spec-portal-service` 744 个 Java 文件、82,648 行；`spechub-feishu-bot` 16 个文件（含 3 个测试）。

**校验中修正的 5 处**（原稿依据的既有 wiki 已落后于代码）：

1. **TS 飞书并发控制** —— wiki 描述的 `TsMessageQueueService`（workspace 级锁）+ `TsP2pDispatcher`（user 级锁）双路径**已不存在**，重构为 `TsUserDispatcher` 统一按 `(workspaceId, userId)` 加锁，群聊私聊共享一把锁实现跨渠道互斥；锁 key 实为 `spec:ts:user:lock:`，30s TTL + 10s Watchdog 续期，虚拟线程异步执行。
2. **会话锁 key** —— 实为 `spec:chat:lock:{sessionId}`，不含 agentType。
3. **Agent 抽象方法** —— 实为 7 个而非 4 个；`streamChat` 声明为 `final`。
4. **交付节点结构** —— 实为两级：仓库级 lane `[CODING, CR, DEPLOY, API_TEST]` + flow 级阶段 `[UI_TEST]`，原稿写成扁平链且漏了 `API_TEST`。
5. **MCP Client 缓存 key** —— 实为 `{mcpServiceId}:{workspaceId}:{headersFingerprint}`，凭证指纹入 key 使 token 轮换后自动重建连接。

已确认无误的关键点：16 个核心类全部存在且路径一致；`precisionFilterStage`（LLM 精筛）确实在默认检索管道中被注释掉；蒸馏九阶段顺序与 `DegradationPolicy` 三档枚举、`RrfFusion`、embedding 2560 维、Python bot 全部命令正则与配置默认值均与文档一致。
