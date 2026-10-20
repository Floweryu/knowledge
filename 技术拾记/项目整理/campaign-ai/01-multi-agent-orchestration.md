# 技术亮点 01：多 Agent 编排与状态契约

## 已实现机制

Main 使用 `create_deep_agent` 构建，是公共链路的根图。`SUBAGENT_SPECS` 注册招商、选品、优惠券、秒杀、图片、页面、报名通知、广告投放 8 类子 Agent，通过同步 `CompiledSubAgent` 暴露给 Main。

模型维护 `ActivityPlan` 并决定任务委派。计划更新通过类型化操作实现，包含节点、页面、模块和状态更新；相关操作按批次校验，失败时不提交半份计划。计划顶层节点和相对顺序存在明确约束，因此这里是受业务约束的动态编排。

子 Agent 包装层执行输入投影和结果校验：消息只保留本次任务消息，业务状态通过契约传递；结果必须提供一致的 `TaskSnapshot` 与 `task_status`，不能改动其他任务的状态或未授权产物字段。该机制是应用层状态边界，不是进程安全沙箱。

## 简历表述

> 设计 Main Agent 与领域子 Agent 的分层编排机制，统一注册 8 类业务能力，通过类型化计划更新、TaskSnapshot 和产物字段校验管理任务进度与跨 Agent 状态回写。

个人只参与部分模块时，应改为“参与设计”或明确负责的子 Agent。

## 技术问题与回答

### Q1：为什么需要多 Agent，一个大 Agent 不够吗？

**回答思路：** 业务领域拥有不同的工具、操作文档、私有草稿和确认流程。把所有细节放进一个 Agent，会扩大工具选择空间，也容易把一个领域的中间状态带到另一个领域。当前方案让 Main 负责组织任务，领域 Agent 完成自己的业务闭环。

**深入追问：** 多 Agent 一定更快、更准吗？不一定。它增加委派、上下文组织与结果回收成本。是否值得拆分，要看领域边界、工具复杂度和独立验证需求；当前实现也不是全并行执行。

### Q2：为什么不用固定的 LangGraph 工作流？

**回答思路：** 用户可能只查询某个资源，或在预览阶段修改条件，不总是从头到尾完成活动。Main 的下一步由模型结合意图和当前状态决定，业务前置条件、输入格式及写入约束由代码或明确的流程规则限定。

**深入追问：** 是不是没有固定顺序？不是。ActivityPlan 顶层节点有固定相对顺序，部分业务有真实产物前置条件。动态选择与业务约束并存，不能描述成任意自主规划。

### Q3：主 Agent 和子 Agent 的职责如何划分？

**回答思路：** Main 管目标理解、活动上下文、计划和委派；子 Agent 管领域参数收集、查询、草稿、执行及结果。Main 不应重复实现子 Agent 的业务逻辑。统一注册规格描述构建器、输入投影、结果校验与允许回写字段。

**深入追问：** 子 Agent 缺参数怎么办？由领域契约表达需要澄清或失败等状态，Main 再组织交互。不能从“模型说完成了”推断成功，最终还要看结构化结果和产物。

### Q4：如何避免子 Agent 污染主状态？

**回答思路：** 包装层复制输入，缩小消息历史，按领域投影输入；返回时检查任务标识、状态一致性、其他任务是否被改动，并限制产物字段。中断中的结果只向上传递必要的消息与 interrupt 信息。

**深入追问：** 是否彻底隔离了全部数据？没有。代码会复制共享状态后再投影，并非所有字段都被最小化过滤；也没有进程级隔离。准确说法是“限制对话输入与结果合并边界”。

### Q5：TaskSnapshot 比自然语言结果多解决了什么？

**回答思路：** 它提供任务名、状态、摘要、用户消息、阻塞原因、是否可重试、错误和产物引用。失败必须带错误，等待状态必须带阻塞原因，主流程能依据契约检查结果，而不是解析一段自由文本猜状态。

**深入追问：** 它是完整任务事件账本吗？不是。`task_status` 按任务名合并最新快照，同名任务的新结果会覆盖旧结果。审计历史还需要结合 checkpoint、会话与观测记录。

### Q6：ActivityPlan 和 TaskSnapshot 是同一个东西吗？

**回答思路：** ActivityPlan 是模型维护的业务计划和进度视图，可包含页面、模块、多语言等层次；TaskSnapshot 是一次领域任务返回的语义结果。两者粒度不同。当前 Main 通过 `update_activity_plan` 显式更新计划，不能把计划上的 completed 当成外部产物存在的证明。

**深入追问：** 模型漏更新计划怎么办？这是独立的一致性风险。现有类型校验保证操作形态，不自动证明进度与外部资源一致。可用测试和评估检测；确定性对账属于可讨论的后续方案。

### Q7：为什么使用原子计划操作，而不是让模型覆盖整个 JSON？

**回答思路：** 增量操作减少误删不相关节点的风险，也能明确区分省略字段、清空字段和替换字段。代码先校验整批操作再产出新计划，父结构应先于子结构建立。

**深入追问：** 这里“原子”是什么范围？是本次计划变换的全有或全无，不是对外部业务接口、数据库和整个多 Agent 流程的分布式事务保证。

### Q8：为什么当前不让多个业务工具并行？

**回答思路：** 某些工具修改同一份状态，或执行必须有序的确认和写入。当前 Main 对 Anthropic 模型设置禁止并行工具调用，领域侧还需根据自己的状态通道处理。选择串行是为了明确状态合并和副作用顺序。

**深入追问：** 一个 reducer 能解决吗？只能解决特定状态值的合并。两个并行写操作都看到旧状态，仍可能重复创建外部资源；需要额外的业务互斥或幂等机制。

### Q9：一个子 Agent 异常，会导致整个会话丢失吗？

**回答思路：** Main 的包装层把普通异常转换为失败的 TaskSnapshot，保留可重试信息；`GraphBubbleUp` 则继续抛出，避免把中断控制流误判为业务错误。恢复依赖 checkpoint 和后续调用。

**深入追问：** 标记 retryable 是否意味着自动安全重试？不是。错误转换不保证外部系统没有发生副作用；超时后必须考虑结果未知、回查和幂等，而不是无条件重复请求。

### Q10：新增一个领域 Agent，需要改哪些地方？

**回答思路：** 定义领域输入、输出和快照契约；实现领域图与工具；声明输入投影、结果校验和产物字段；通过注册规格接入；补充该领域边界与委派集成测试。Main 保持语义编排职责。

**深入追问：** 注册是否等于所有功能自动完成？不是。用户交互、产物契约、前置条件与评估成功口径仍需明确。新增公共状态或跨模块流程要先设计边界。

## 可展开的案例

“子 Agent 返回 succeeded，但没有合法产物引用”可以串起整个回答：自然语言不能作为权威结果 → 引入结构化快照 → 领域校验 → 限定回写字段 → 评估再检查终态和产物证据。不要把这个讨论案例说成自己已经处理过的线上事故。

## 代码与测试依据

- [统一子 Agent 注册表](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagent_registry.py)：`SUBAGENT_SPECS`、`compile_subagent_graphs`。
- [Main 装配](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/main/agent.py)：`_build_subagents`、`_task_failure_result`、`build_main_agent`。
- [状态包装边界](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/state_driven_subagent.py)：`_validate_result`、`_finish_result`、`wrap_state_driven_subagent_runnable`。
- [TaskSnapshot 契约](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/schemas/state/contracts/task_snapshot.py)：`validate_status`、`merge_task_status`。
- [计划操作](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/main/activity_plan_ops.py)与[计划工具](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/main/tools/update_activity_plan.py)。
- [计划操作测试](/Users/Floweryu/Workspace/ibu_campaign_copilot/tests/agent/main/test_activity_plan_ops.py)、[Main 根图测试](/Users/Floweryu/Workspace/ibu_campaign_copilot/tests/agent/test_main_agent_runtime.py)。测试为源码证据，本次未执行。
