# 技术亮点 02：语义契约与可靠工具执行

## 已实现机制

选品 Agent 将自然语言理解、实体解析、语义定义、报文构造、预览和写入分开。查询工具接入 HOTEL、FLIGHT、TNT、PKG_TOUR、CRUISE_SHIP、CAR_RENTAL、TRAIN 七条产品线；具体创建能力还要核对 `SPEC_BY_PRODUCT_LINE`、builder 和 E2E，不以工具数量代替验收。

模型提交业务语义，Pydantic 契约和 builder 决定字段、层级和具体报文。原始 upsert 工具从模型工具集合中过滤，写入收敛到 `upsert_selection_pools`。工具校验预览记录和活动上下文，执行层汇总逐池结果。

## 简历表述

> 构建选品 Agent 的语义契约与确定性执行链路，将产品线查询、报文构造、业务校验和写入门禁分层，支持预览反馈、部分失败处理及会话内重复创建防护。

## 技术问题与回答

### Q1：为什么不让 LLM 直接生成 SOA 请求 JSON？

**回答思路：** 请求的字段层级、枚举、操作符和跨字段约束大多是确定的，交给模型重复生成容易出现格式正确但业务错误的报文。当前模型输出语义定义，代码根据产品线契约构造请求，把不需要推理的部分收敛到可测试逻辑。

**深入追问：** 这样是不是失去 Agent 灵活性？没有。数据源选择、需求解释、条件组合和澄清仍由模型完成；接口结构与固定约束由代码处理。

### Q2：Pydantic 校验通过，为什么还要业务校验？

**回答思路：** 类型正确只说明“像一个合法对象”。某指标可能不适用于该数据源，招商数据源可能缺少规则 ID，规则型数据源可能不允许空条件。builder 和 guard 负责这些领域限制。

**深入追问：** 合法 ID 是否就是正确 ID？不是。还要核对实体类型、业务归属、权限与用户意图。查询工具和真实业务接口提供证据，模型不能只因为值是整数就认定可用。

### Q3：多条产品线为什么不共用一个包含大量可选字段的模型？

**回答思路：** 产品线请求属于不同报文族，共用大模型容易让不适用字段也通过校验。当前基类保留池身份与更新语义，各产品线定义自己的结构，再由 builder 映射报文。

**深入追问：** 所有字段都应独立吗？也不必。共性的身份、create/update 语义应统一；只有真实结构差异才拆分，避免同义规则出现多份实现。

### Q4：如何防止模型绕过预览直接写入？

**回答思路：** 第一层收窄工具面，移除原始 upsert；第二层在统一写入工具检查预览记录；第三层检查活动上下文和执行参数。这比仅在 prompt 中要求“先预览”多了一层结构性限制。

**深入追问：** 这是不是严格的审批授权？不是。当前记录按池名识别，用户回复语义由模型解释，没有把确认与完整 payload 哈希绑定。若需要强绑定，可增加草稿版本、摘要和单次确认令牌；这是改进建议。

### Q5：用户看过草稿后修改条件，原确认还有效吗？

**回答思路：** 业务上应该针对变更后的草稿重新预览。当前写入凭证主要证明池名曾经过预览，不能仅凭该机制证明“本次写入的每个字段都被确认”。技术交流时应主动说明这个边界。

**深入追问：** 更严格的设计怎么做？将规范化草稿生成稳定摘要，确认记录绑定摘要与版本；草稿变化即失效，执行前再次比较。还应处理确认过期和重放。本文没有实现这些改进。

### Q6：校验失败为什么返回 ToolMessage，而不是直接抛异常？

**回答思路：** 可修正的参数错误需要进入模型上下文，让模型知道错误字段、允许值或缺失条件，在同一循环内修正。直接终止图会损失这种反馈。真正的执行异常仍应记录并按明确边界处理。

**深入追问：** 会不会无限修复？有可能。需要区分可修复参数错误、权限错误和基础设施错误，并检查现有调用预算；如增加同类错误上限或人工接管，应作为进一步设计，不能声称所有路径都有统一重试控制。

### Q7：三只池创建了两只，重试如何处理？

**回答思路：** 执行层保留逐池结果。统一写入工具读取当前会话成功产物，对已经有成功状态和 schema ID 的新建池按池名跳过，剩余池继续处理；更新请求不能按同样规则直接跳过。

**深入追问：** 能避免所有重复创建吗？不能。远端成功但本地尚未记录、跨会话重试、并发写入，都可能绕过这层防护。端到端幂等需要业务键、上游支持或结果回查。

### Q8：为什么更新选品池比创建更容易出问题？

**回答思路：** 部分接口使用全量替换语义。以 tab 列表为例，旧 tab 带 ID 才表示更新，缺席可能表示删除。如果用户只要求改一条规则，模型却只提交那一条对应的结构，就可能删掉其他内容。

**深入追问：** 如何控制？先查询完整旧资源，保留不变项与原 ID，只修改目标字段，再预览整体结果。当前契约对 `tab_id` 与更新 `schema_id` 有说明，但不是所有全量替换风险都能靠类型校验消除。

### Q9：为什么不能简单给状态加 reducer 来开启并行？

**回答思路：** 选品有多个读改写通道。追加型 reducer 如果叠加工具内“读取历史再追加”，会重复合并历史；而两个 upsert 同时读到旧的成功产物集合，仍然可能双写外部系统。状态合并正确与副作用安全是两件事。

**深入追问：** 如何提速？优先分析只读查询、实体解析的批处理与有限并发。确认和写入保留顺序，除非已经补齐互斥、幂等和并发恢复测试。

### Q10：这条链路应该怎么测试？

**回答思路：** 分层验证：语义契约的非法字段和更新要求；builder 的报文映射；guard 的数据源与指标约束；工具的无预览拒写、部分成功重试；最后才是真实 LLM 和 SOA E2E。单测优先证明确定性边界，不依赖模型随机行为。

**深入追问：** E2E 成功一次能说明什么？只能证明该输入、该配置、该时刻的一次执行。需要多种需求、澄清和失败场景组成固定评估集，并记录重复实验结果，才能讨论整体成功率。

## 可展开的案例

选“部分成功后重复创建风险”作为讨论案例：模型重试整个池列表 → state 仍保留全部输入 → 查询成功产物并划分 pending/skipped → 明确只保护会话内新建重试 → 再讨论结果未知窗口和上游幂等。若没有亲自处理，不使用“我在线上发现并解决”的叙述。

## 代码与测试依据

- [选品工具装配](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/agent.py)：`LINE_QUERY_TOOLS`、`_strip_upsert_tools`。
- [语义契约](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/pool/spec.py)：`BasePoolSpec`、`TabSpec`、`SPEC_BY_PRODUCT_LINE`。
- [报文构造](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/pool/builder.py)、[业务约束](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/pool/guards.py)。
- [草稿工具](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/tools/draft.py)、[预览工具](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/tools/preview.py)、[写入工具](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/tools/upsert.py)。
- [执行与结果聚合](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/pool/upsertor.py)、[并发语义](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/subagents/selection/agent_state.py)。
- [选品结构测试](/Users/Floweryu/Workspace/ibu_campaign_copilot/tests/agent/subagents/selection/test_selection_changes.py)、[更新池 E2E](/Users/Floweryu/Workspace/ibu_campaign_copilot/e2e/test_selection_flight_edit_pool.py)、[空规则 E2E](/Users/Floweryu/Workspace/ibu_campaign_copilot/e2e/test_selection_empty_rules_pool.py)。测试为源码证据，本次未执行。
