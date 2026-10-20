# 技术亮点 06：会话级 Agent 评估体系

## 已实现机制

评估脚本按 Session 汇总多个 Trace，构建 Session Bundle，再按指标与 Agent 范围投影证据。确定性指标与语义指标采用不同路径：任务成功依据终态和产物，计数与耗时由代码计算；推理、字段推断、采纳等语义维度使用 LLM Judge。

代码生成固定 Evaluation Units，Judge 返回逐单元判断，程序验证是否遗漏、重复或返回未知单元，再计算指标。超出请求大小限制时，将证据分片扫描再汇总；评分请求共享并发上限，指标失败进行隔离，结果生成总览、明细和失败案例报告。

该脚本评估已有会话，不会重新执行 Campaign Agent。重复评估同一会话能分析 Judge 波动，但不能证明 Agent 多次真实执行的稳定性。

## 简历表述

> 构建 Session 级离线评估流程，结合终态产物校验与 LLM Judge，支持固定评分单元、覆盖校验、长会话证据分片及失败案例报告，为 Agent 质量迭代提供可追溯依据。

## 技术问题与回答

### Q1：为什么要评估完整 Session，而不是单条回答？

**回答思路：** 一项业务可能跨澄清、预览、修改、确认和恢复多个阶段。单条回答看不到之前的需求与后面的真实结果，也可能把一次暂停算成失败。Session Bundle 统一组织多 Trace 证据，再按评分问题裁剪。

**深入追问：** Session 里的所有任务能混在一起算吗？不能。需区分 Agent 范围和逻辑任务，防止一个任务的产物被拿来证明另一个任务成功。

### Q2：怎么判断任务成功，模型说成功算吗？

**回答思路：** 当前确定性判定要求终态为 succeeded、没有执行错误，并命中该 Agent 的有效产物证据。只包含 interrupt、缺终态快照或没有配置产物契约时，保留不可评估状态。

**深入追问：** 有产物 ID 就证明业务完全正确吗？不是。它证明记录中存在符合契约的执行产物，不证明内容正确或资源现在仍然有效。语义质量需单独评估，必要时还要外部回查。

### Q3：为什么既用规则，又用 LLM Judge？

**回答思路：** 轮次、调用次数、时间戳、状态与产物存在性适合代码计算。是否理解用户条件、解释是否合理、用户是否要求内容修改等需要语义判断。将两者分开，可以减少不必要的模型评分和主观波动。

**深入追问：** Judge 能替代人工验收吗？不能。仍需人工标注样本校准，分析一致率与分歧类型。当前有评估实现，不等于已经证明 Judge 与人工高度一致。

### Q4：为什么不让 Judge 直接返回一个 0～100 分？

**回答思路：** 总分很难发现漏评或分母漂移。代码先生成固定评分单元，Judge 只逐项给出 verdict 与证据，再由代码计算分子、分母和分数。不同指标使用适合自己的计算方式，并非都简单平均。

**深入追问：** 能解决全部评分偏差吗？不能。它约束统计范围，无法自动纠正模型对单个单元的错误判断，因此还需要 rubric、证据要求和人工校准。

### Q5：怎么检测 Judge 漏看了部分问题？

**回答思路：** 将返回的 unit_id 与固定清单比较，拒绝重复 ID、缺失 ID 和未知 ID。部分非法或空结果会触发批次拆分等处理，不能把不完整输出直接当作完整成绩。

**深入追问：** 所有 ID 都返回了就代表看过全部证据吗？不代表。覆盖校验只证明输出覆盖了单元。证据扫描、引用校验和抽检进一步提高可信度，但仍不能保证每个推理判断都正确。

### Q6：长会话超过 Judge 输入限制怎么办？

**回答思路：** 先按指标和 Agent 投影，去掉无关或重复内容；仍超限时拆成带稳定定位的证据片段，逐片扫描相关事实，再汇总到固定单元。如果汇总请求仍超限，尝试拆分单元批次；单单元仍无法容纳时明确失败。

**深入追问：** 为什么不直接截断？截断可能丢掉用户后续修改或最终结果，造成系统性偏差。分片也有代价：调用变多、成本上升，并可能在汇总时损失关系，需要保留证据定位与失败信息。

### Q7：not_evaluable 应该算失败，还是直接忽略？

**回答思路：** 当前任务成功口径把缺乏充分证据或尚未终结的任务排除出可评估分母，而不是强行判成功或失败。但报告必须同时看可评估覆盖率与排除原因。

**深入追问：** 会不会通过排除难例抬高成绩？会有这个风险。因此比较实验时要固定筛选口径，报告样本总数、可评估数和失败数；不能只展示剩余样本上的高分。

### Q8：怎样避免某个评估指标失败拖垮所有实验？

**回答思路：** 多指标共享 Semaphore 控制实际并发，每个指标单独捕获异常，其他指标可继续。失败信息通过记录和报告暴露，不能把缺失指标视为零分或成功。

**深入追问：** 并发越高越好吗？不是。要考虑模型服务限流、单请求大小、重试放大和成本预算。并发上限应按实际服务能力测量调整。

### Q9：怎么比较两个版本，证明改动有效？

**回答思路：** 先固定任务集、模型与配置，分别运行被测 Agent 得到新会话，再用相同评估口径分析成功、语义质量、成本和耗时。判断收益应同时展示样本量与失败案例。

**深入追问：** 对同一份历史 Session 多跑几次 Judge，算 A/B 吗？只能分析评估器波动。评估脚本不重跑业务 Agent，不能用它替代两个业务版本的执行对照。

### Q10：耗时 P95 和用户等待时间应该怎么统计？

**回答思路：** 代码从 observation 时间戳提取原始样本；子任务执行耗时需要区分执行片段与用户等待，端到端墙钟耗时则可能包含等待。P95 应从符合统一口径的原始样本求分位数。

**深入追问：** 能平均多个 Session 的 P95 得到总体 P95 吗？不能。分位数不可这样平均。还要报告样本规模、是否只统计完成任务，避免因排除失败和超时任务美化延迟。

## 建议补齐的实验材料

以下是技术交流前建议准备的证据，不是已验证的线上成果：

| 材料 | 至少记录什么 |
|---|---|
| 固定评估集 | 任务类型、难度、预期结果、确认和失败分支 |
| 实验版本 | 代码、模型、prompt、工具与配置版本 |
| 成功率报告 | 样本总数、可评估数、成功数、失败类型 |
| Judge 校准 | 人工标签、Judge 标签、分歧原因 |
| 成本报告 | 每任务模型调用、输入输出与缓存 usage |
| 耗时报告 | 执行耗时与墙钟耗时、P50/P95、失败与超时口径 |

## 代码与测试依据

- [兼容 CLI 入口](/Users/Floweryu/Workspace/ibu_campaign_copilot/scripts/langfuse_session_evaluation.py)、[实现说明](/Users/Floweryu/Workspace/ibu_campaign_copilot/scripts/langfuse_eval/TECHNICAL_IMPLEMENTATION.md)。
- [Session Bundle](/Users/Floweryu/Workspace/ibu_campaign_copilot/scripts/langfuse_eval/bundle.py)、[评分单元与投影](/Users/Floweryu/Workspace/ibu_campaign_copilot/scripts/langfuse_eval/units.py)。
- [确定性任务成功判定](/Users/Floweryu/Workspace/ibu_campaign_copilot/scripts/langfuse_eval/task_success.py)：`_task_verdict`。
- [覆盖校验与计分](/Users/Floweryu/Workspace/ibu_campaign_copilot/scripts/langfuse_eval/judge_support.py)：`_validate_unit_coverage`、`_reduce_unit_results`。
- [分片与汇总](/Users/Floweryu/Workspace/ibu_campaign_copilot/scripts/langfuse_eval/judge_runtime.py)、[指标并发与隔离](/Users/Floweryu/Workspace/ibu_campaign_copilot/scripts/langfuse_eval/judge.py)。
- [时间指标](/Users/Floweryu/Workspace/ibu_campaign_copilot/scripts/langfuse_eval/timing.py)、[评估测试](/Users/Floweryu/Workspace/ibu_campaign_copilot/tests/scripts/test_langfuse_session_evaluation.py)。测试为源码证据，本次未执行。
