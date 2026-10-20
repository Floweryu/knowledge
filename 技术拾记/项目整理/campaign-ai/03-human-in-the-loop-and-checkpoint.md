# 技术亮点 03：人工确认与持久化恢复

## 已实现机制

项目通过 LangGraph interrupt 与 AG-UI 交互表达部分业务确认流程，通过根图 PostgreSQL checkpoint 保存恢复所需的状态。嵌套子图编译时使用 `checkpointer=None`，在根图调用上下文中继承父 checkpoint 机制。

确认并非全部使用相同实现：选品预览使用 interrupt；Main prompt 也定义了普通对话中的阶段确认。分析时需要区分业务确认与图中断，不能把所有确认步骤都说成持久化 interrupt。

## 简历表述

> 基于 LangGraph interrupt 与 PostgreSQL checkpoint 实现业务预览确认和跨请求续跑，处理嵌套子图状态隔离、中断结果边界及恢复时的副作用风险。

## 技术问题与回答

### Q1：为什么不能在 HTTP 请求里直接等用户确认？

**回答思路：** 用户可能长时间不回复、刷新页面或从另一个请求返回。确认点应转化为可恢复的业务状态，而不是占用一个请求等待。interrupt 将待确认内容返回，后续请求依据同一会话上下文恢复。

**深入追问：** checkpoint 是否等于业务数据库？不是。它保存图状态与执行信息，不代替外部活动、选品池等系统的业务数据。

### Q2：interrupt 恢复时从哪继续，为什么要关心？

**回答思路：** 本项目按“包含 interrupt 的节点会重新执行”的语义设计。中断前如果发送不可去重消息或调用写接口，恢复时可能再次发生。选品预览因此把展示内容直接放在 interrupt payload 中，而不是先发一遍聊天气泡。

**深入追问：** 把写接口移到 interrupt 后就绝对安全了吗？没有。写入完成后、本地结果持久化前仍可能失败，恢复或重试时还需考虑幂等与回查。

### Q3：为什么只有一个根 checkpoint？

**回答思路：** 当前 Main 是唯一根图，嵌套任务沿父执行上下文恢复，避免再建立外层主图复制业务状态，也避免子 Agent 通过独立持久状态串到下一次任务。

**深入追问：** `checkpointer=None` 是否意味着子图没有恢复能力？不能单独下结论。在本项目的嵌套执行环境中依赖父 checkpoint；如果把子图脱离父图独立运行，需重新配置和验证。

### Q4：如何区分同一任务恢复和同一个 Agent 的新任务？

**回答思路：** 恢复必须关联原线程和嵌套执行上下文；新委派则应以新任务输入启动，不沿用上一任务的私有草稿。包装层保留配置传递，测试覆盖二次调用和连续中断场景。

**深入追问：** 只保留 thread_id 够吗？thread_id 是根会话标识，嵌套 checkpoint 上下文也要正确传递。不能随意把子任务改成另一个独立线程再期望原确认点能恢复。

### Q5：中断时为什么不能立即把子 Agent 状态合并回 Main？

**回答思路：** 这时子任务尚未完成，可能只是草稿或等待输入。包装层对带 `__interrupt__` 的结果只保留必要消息和中断信息，不把未验证的业务产物当成最终结果。

**深入追问：** 草稿放在哪里？由相应子图和 checkpoint 管理。共享产物的发布必须符合该领域完成与回写契约，不能拿用户可见预览替代真实执行结果。

### Q6：怎样处理用户确认、修改和取消？

**回答思路：** 选品预览恢复后，将原始回复交给模型在当前上下文中解释，再决定执行、修改后再预览或取消。代码校验结构性条件，模型负责自然语言语义。

**深入追问：** “再看看”会不会被误判确认？这是模型语义判断风险。当前预览凭证不是严格确认令牌。对高风险操作，可使用结构化确认动作并绑定草稿版本，但需明确这是增强方案。

### Q7：数据库断连如何影响恢复？

**回答思路：** 当前 PostgreSQL Saver 使用连接池、连接检查、连接归还清理以及特定写入操作的 OperationalError 重试，并关闭 pipeline 以适配代理场景。源码中记录了单连接并发访问与代理兼容问题。

**深入追问：** 重试能保证业务原子性吗？不能。Saver 重试是 checkpoint 存储层容错，不把外部 SOA 写入与 checkpoint 组成同一事务。

### Q8：checkpoint 保存了，是否就能保证不重复执行？

**回答思路：** 不能。考虑远端写成功、本地还没记录成功就异常的窗口。checkpoint 能恢复已持久化的图状态，但不知道未持久化的外部结果。必须借助业务幂等键、执行记录或远端查询来缩小不确定性。

**深入追问：** 项目当前具备哪一层防护？选品有会话成功产物去重；没有据此证明跨会话和所有外部接口的 Exactly-once。

### Q9：恢复时的权限和身份从哪里来？

**回答思路：** 项目通过请求级 ContextVar 传播 SSO 身份，原始 cookie 不应作为业务状态持久化进 checkpoint。恢复执行需要当前请求在正确的身份上下文中进入链路。

**深入追问：** thread_id 就是访问授权吗？不是。它是定位键，仍需验证会话所属和资源权限。ContextVar 能解决上下文隔离，但不自动构成完整授权检查；本文没有据此认定所有入口都已验证权限。

### Q10：恢复功能如何设计测试？

**回答思路：** 覆盖首次 interrupt、同线程 resume、连续两次 interrupt、同子 Agent 新任务状态清空，以及恢复时输入契约的变化。单独测试消息去重与业务副作用次数。

**深入追问：** 单测能证明生产重启恢复吗？不能。还需要真实 PostgreSQL、进程重启、外部接口结果未知等故障场景的集成验证。本次只确认测试源码存在。

## 可展开的案例

可用“恢复时预览重复展示”讲清执行语义：interrupt 前发送消息 → resume 重跑节点 → 重复副作用 → 把用户可见预览放入 interrupt → 测试首次展示与恢复后的消息数量。应以自己的实际经历补充日志和验证结果。

## 代码与测试依据

- [checkpoint 初始化与连接池](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/graph_runtime.py)：`_boot_runtime`、`_postgres_checkpointer`、`deepagent_lifespan`。
- [选品预览中断](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/tools/preview.py)：`preview_selection_pools`。
- [中断结果过滤](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/state_driven_subagent.py)：`_finish_result`。
- [Main 确认策略](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/main/prompt.py)：区分普通对话确认和工具内确认。
- [SSO 请求上下文](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/utils/sso_context.py)、[入口身份绑定](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/middleware/sso_auth_middleware.py)。
- [嵌套恢复与状态隔离测试](/Users/Floweryu/Workspace/ibu_campaign_copilot/tests/agent/test_subagent_checkpoint_namespace.py)、[恢复请求解析测试](/Users/Floweryu/Workspace/ibu_campaign_copilot/tests/deepagent/test_request_parser_resume.py)。测试为源码证据，本次未执行。
