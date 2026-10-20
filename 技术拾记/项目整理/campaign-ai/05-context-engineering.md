# 技术亮点 05：上下文工程与成本归因

## 已实现机制

Main 使用 CompositeBackend：默认 StateBackend 存放会话相关内容，`/skills/` 路由指向 Main 的 Skills 目录，Skills 写操作被禁止。领域子 Agent 通过自己的工具与参考文档获取业务规则。

Main 的 middleware 每次调用模型时从当前状态派生上下文，将静态提示与动态状态分块。当前时间按 Asia/Shanghai 的十分钟窗口取整；Anthropic 客户端才附加对应的 cache_control 标记。此前“确认活动类型后注入 effective_knowledge 快照”的描述不再代表本次核对的实现。

运行结束后，系统调度 Langfuse 子 Agent Token 归因，按 observation 树识别领域执行消耗和根 Agent 自身消耗。这个机制提供分析依据，不等于已经取得特定节省比例。

## 简历表述

> 实现 Skills 按需读取、权威状态动态注入与稳定提示前缀组织，结合模型兼容的缓存控制和子 Agent Token 归因，为上下文优化与成本分析提供依据。

## 技术问题与回答

### Q1：Context Engineering 与写一个好 prompt 有什么区别？

**回答思路：** 不只设计指令文本，还决定模型在某次调用能看到哪些知识、工具、历史、业务状态，以及它们何时刷新和如何持久化。项目分别管理稳定策略、Skills 资料、本次任务消息和动态活动上下文。

**深入追问：** 上下文越多越好吗？不是。无关信息会增加成本和干扰，缺失必要信息会导致反复查询或错误推断，需要依据任务裁剪。

### Q2：为什么不把所有业务文档直接放进 system prompt？

**回答思路：** 不同产品线所需规则不同，全量注入会增加每次请求负担。Skills 目录提供可发现的领域入口，模型按需求读取具体资料。代价是多一次或多次工具调用，也存在漏读风险。

**深入追问：** 怎么保证关键规则被执行？关键写入规则仍要落在类型契约、builder 或工具门禁中，不能只依赖文档被读到。Skills 负责操作指导，代码守住确定性边界。

### Q3：这个项目算 RAG 吗？

**回答思路：** 本专题核实的是文件式 Skills 按需读取与状态注入。仅凭这些不能证明有向量索引、召回、重排、引用评估等完整 RAG 链路。简历应使用与实现一致的“领域知识按需读取”或“上下文工程”。

**深入追问：** 什么时候需要向量检索？当资料规模、检索需求和更新方式使目录导航不足时再考虑；需要评估召回、权限过滤和知识版本，而不是为了技术关键词追加组件。

### Q4：为什么业务状态必须每次调用时注入？

**回答思路：** 本轮工具可能创建资源、切换活动或改变计划。只依赖历史消息会让模型继续使用旧 ID。middleware 从当前 state 派生模型视图，减少模型自行重建业务状态的负担。

**深入追问：** state 一定等于外部最新事实吗？不是。它是当前运行的权威输入视图，外部资源可能被其他人修改。涉及更新仍需通过真实查询校准，缓存和 state 不能替代外部一致性策略。

### Q5：稳定前缀和动态内容为什么要分开？

**回答思路：** 变化较少的指令放前面，有利于复用相同前缀；实时状态放后面，保持业务新鲜度。当前实现对 Anthropic 客户端设置缓存标记，不向所有模型无差别发送厂商参数。

**深入追问：** 设置 cache_control 就一定命中吗？不能保证。还受实际请求前缀、模型服务、窗口和返回 usage 口径影响。必须看实际缓存读写统计，不能从代码标记推导成本收益。

### Q6：时间为什么按十分钟取整，有什么风险？

**回答思路：** 每秒变化的时间文本会使后续前缀更容易变化，取整可减少无业务意义的变动。当前代码将分钟向下对齐到十分钟，并在文本中说明窗口语义。

**深入追问：** 能用这个时间做秒杀截止校验吗？不应把粗粒度模型上下文当精确时间源。敏感的截止判断需要实际业务校验。窗口时间最多落后接近十分钟；缓存 TTL 与时间取整窗口也不是同一个概念。

### Q7：为什么子 Agent 只拿本次任务消息？

**回答思路：** 主对话可能包含多个领域、旧活动和已取消需求，全部传入会放大干扰。当前包装层缩小消息历史，同时保留原始 brief 和经过投影的业务状态，兼顾当前任务与必要事实。

**深入追问：** 会不会丢失用户先前要求？会有这个风险。委派描述和结构化状态必须覆盖仍然有效的条件，输入投影需要测试。不能靠压缩消息数量本身证明信息完整。

### Q8：read_reference 为什么采用整文件读取？

**回答思路：** 工具不暴露 offset/limit，避免模型围绕一个规则文件反复分段读取；同时限制在子 Agent 目录内，并设置文件大小上限。它适合内容规模可控的规则文档。

**深入追问：** 大文档怎么办？整文件策略有上下文成本。可先拆成有明确主题的小文件，再通过索引导航。不能简单放大上限；本项目这个工具的上限为 100 KiB，也不意味着所有文件读取工具都共享该策略。

### Q9：把外部内容标成“不可信数据”，就能防止 Prompt Injection 吗？

**回答思路：** 这种标记帮助模型区分数据与指令，但不是安全证明。实际边界还包括工具权限、文件访问范围、结构化输入校验和写入门禁。Main 对 Skills 路由禁止写入，也不能推导为整个运行时拥有完整沙箱。

**深入追问：** 如何验证？建议设计恶意文档、越权字段、伪造 ID 和绕过确认等测试。是否全部具备，需要逐项核对测试，本文不宣称已完成完整攻击评估。

### Q10：如何知道优化该先改哪个 Agent？

**回答思路：** 按 observation 树统计领域子树中的 GENERATION usage，再将不属于子任务的调用归给根 Agent。这样能区分成本来自 Main 反复规划、某个子 Agent 多次试错还是大工具结果。

**深入追问：** Token 就等于费用吗？不是。还要区分模型、输入输出、缓存读写和实际计费。比较优化前后时需要固定任务集、记录版本，同时观察质量和延迟，避免只靠减少调用次数宣称优化成功。

## 可展开的案例

以“同一规则被反复读取”组织案例：定位重复工具调用 → 分析文档导航和工具读取接口 → 改善文件粒度或读取方式 → 对比工具调用数、输入量和成功率。项目已提供整文件读取工具，但具体收益必须通过实验补齐。

## 代码与测试依据

- [Main Backend 与 Skills 装配](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/main/agent.py)：`build_main_agent`、`SKILLS_ROOT`、`_FS_PERMISSIONS`。
- [动态上下文与时间窗口](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/main/middleware.py)：`build_main_context`、`_system_message_with_context`、`inject_main_context`。
- [子任务消息投影](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/state_driven_subagent.py)：`prepare_input`。
- [领域参考文件读取](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/tools/read_reference.py)、[领域 Agent 工厂](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/agent/skill_agent.py)。
- [Token 归因](/Users/Floweryu/Workspace/ibu_campaign_copilot/campaign/observability/langfuse_subagent_stats.py)：`_sum_descendant_usage`、`_sum_root_own_usage`。
- [Main middleware 测试](/Users/Floweryu/Workspace/ibu_campaign_copilot/tests/agent/main/test_middleware.py)、[参考文件读取测试](/Users/Floweryu/Workspace/ibu_campaign_copilot/tests/agent/test_read_reference.py)。测试为源码证据，本次未执行。
