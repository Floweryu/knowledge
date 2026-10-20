# pgc-rag-service 技术点与难点分析

> 分析对象：`git.dev.sh.ctripcorp.com/cct/pgc-rag-service`（Maven artifact `com.ctrip.cct:pgc-rag-search`，appId 100068080）
> 分析基准：`main` @ `f1554c2`（2026-08-25，56 commits / 7841 行 main + 523 行 test / 76 个 Java 文件）
> 验证方式：本地 `mvn compile` 通过、`mvn test` 实跑、关键行为用探针程序实测
> 成文日期：2026-08-27

---

## 1. 这是什么

一个 **面向 PGC（专业生成内容）场景的 RAG 检索服务**，Java 21 + Spring Boot 3，打包成 `war`，通过携程 SOA（Baiji）暴露两个接口：

| 接口 | 方向 | 职责 |
| :--- | :--- | :--- |
| `search` | 读 | 多路召回 → 融合 → 全局过滤 → 全局排序 → TopK |
| `embedding` | 写 | 文档分块 → 向量编码 → bulk 写入 ES |

它同时承担了 RAG 的 **写入侧（索引构建）** 和 **读取侧（检索）**，这一点和名字给人的印象不同 —— 不是纯检索服务。写入侧除 SOA 入口外还有一条 QMQ 消息触发通道（[EmbeddingConsumer.java:19](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/qmq/EmbeddingConsumer.java#L19)，topic `cct.poi.pgc.embedding.ariticle`），用于文章内容的异步入库。

存储只有一个：**Elasticsearch 8.18**。`MilvusRetrievalExecutor` 是个明确标注的空壳占位（[MilvusRetrievalExecutor.java:53](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/impl/MilvusRetrievalExecutor.java#L53) 返回 `List.of()`，且未加 `@Component`）。

```
                    ┌──────────────────────────────┐
   SOA search ─────►│  PgcRagSearchServiceImpl     │  薄入口，只做日志+委托
                    └──────────────┬───────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │  RagSearchServiceImpl        │  7 步编排 + 两级降级
                    └──────────────┬───────────────┘
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
      RetrievalCoordinator  GlobalFilterProcessor  GlobalSortProcessor
      (并行 + 单路容错)        (内存过滤)              (内存排序/兜底相似度)
              │
              ▼
      ElasticsearchRetrievalExecutor ──► RetrievalStrategyFactory
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                          BM25Strategy   KNNStrategy   HybridStrategy
                              │               │          (未完成)
                              └───────┬───────┘
                                      ▼
                        ElasticSearchClientRegistry
                        dataType → index → cluster → client
```

---

## 2. 核心设计：策略配置权完全外置给调用方

这是本仓库最值得关注的设计决策，也是它区别于一般 RAG 服务的地方。

服务**自身不决定怎么召回**。召回路数、每路查哪个索引、查哪些字段、字段权重、匹配模式、ES 层过滤与排序，全部由调用方在请求里以 `List<RetrievalStrategy>` 声明，每个策略带一个 `Map<String,String> params`：

```json
"retrievalStrategies": [
  { "type": "BM25", "params": {
      "dataType": "moments_pgc",
      "fields": "district_info_list.districtname^2.0",
      "matchType": "phrase", "sortFields": "_score:desc", "size": "5" }},
  { "type": "BM25", "params": {
      "dataType": "xhs_pgc",
      "fields": "poi_info_list.poi_name^4.0", "size": "3" }}
]
```

带来的能力是：**同一次请求可以对多个异构索引并行发起结构完全不同的查询，再融合成一个结果集**。上面的例子就是「目的地名索引」和「POI 名索引」两路并查。这对 PGC 这种数据分散在多个索引（小红书 / 朋友圈 / POI 点评）的场景很实用，且新增数据源不需要改服务代码，只要在 QConfig 的 `es-config.json` 里注册 `dataType → index → cluster` 映射即可（[ElasticSearchClientRegistry.java:125](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/es/ElasticSearchClientRegistry.java#L125)）。

代价同样明显，这是**架构层面最大的难点**：

- **契约退化为字符串字典**。`params` 是 `Map<String,String>`，所有值都要手工解析：`size` 用 `Integer.parseInt`、`enableTextFilter` 用 `Boolean.parseBoolean`、`fields` 用 `^` 拆权重、`filterCustom` 用 `,` 和 `:` 拆条件。编译期毫无保护，参数名拼错只会静默走默认值。
- **调优知识转移到了调用方**。字段权重该给多少、`matchType` 该选 `phrase` 还是 `fuzzy`、`size` 与 `topK` 的比例关系，服务端不做任何校验或建议。仓库靠 4 万字文档（`召回策略说明.md` 等）来传递这些知识，而文档与代码已经出现漂移（见 §6）。
- **服务端无法做全局优化**。因为不理解调用方意图，无法自动改写查询、无法根据 query 特征选择策略。

这是一个把灵活性换取类型安全和可控性的取舍。在调用方少（目前主要是 `activity-rag-agent`）且能紧密沟通时可行，调用方变多后会成为维护负担。

---

## 3. 容错设计：两级降级 + 业务告警通道

这是仓库里完成度最高、最值得借鉴的部分。

设计前提是：**RAG 检索的部分结果远好于无结果**。因此单路召回失败不允许影响整体。实现分三层：

**第一层，单路隔离**（[RetrievalCoordinator.java:123-133](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/RetrievalCoordinator.java#L123)）。每路策略提交到线程池独立执行，`collectRetrievalResults` 对每个 future 单独 `join()` 并 `try/catch`，失败的那一路记一条 warning、补一个空列表，其他路正常返回。

**第二层，融合降级**（[RagSearchServiceImpl.java:161-171](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/impl/RagSearchServiceImpl.java#L161)）。融合算法本身抛异常时，退化为「取结果数最多的那一路」：

```java
private List<DocEntity> fallbackToFirstNonEmptyResult(List<List<DocEntity>> strategyResults) {
    return strategyResults.stream()
        .filter(list -> list != null && !list.isEmpty())
        .max(Comparator.comparingInt(List::size))     // 方法名说 First，实现是 max
        .orElse(new ArrayList<>());
}
```

**第三层，全失败仍返回成功码**（[RagSearchServiceImpl.java:139-149](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/impl/RagSearchServiceImpl.java#L139)）。所有路都失败时，`resultCode` 仍是 200，只是 `resultList` 为空并带上告警。语义是「检索系统正常工作，只是这次没召回到」，把「系统故障」和「无结果」交给调用方通过 `warnings` 区分。

配套的 **`warnings` 字段是设计亮点**：它在响应契约里开了一条独立于 `resultCode` 的旁路，专门表达「非致命异常」。`WarningHandler` 把每个告警统一做三件事 —— 造 `WarningDTO`、打 HickWall 指标、写 Clog（[WarningHandler.java:30-50](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/impl/WarningHandler.java#L30)），保证「调用方能看到」和「监控能看到」不会脱节。这种「降级必留痕」的约束，比单纯 try/catch 兜住要成熟得多。

值得一提的是可观测性做得比较细：三套监控并行 —— CAT（`Cat.newTransaction` 埋每路策略、每次 embedding 调用、每次 ES bulk）、HickWall（`pgc.rag.*` 系列计数指标，连「boost 参数解析失败」这种配置错误都打点）、Clog（分 `RAGSearch.Retrieval` / `.Fusion` / `.Sort` 等 title）。并行任务通过 `CatAsync.wrap(executor)` 保证 trace 上下文能跨线程传递（[RetrievalCoordinator.java:56](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/RetrievalCoordinator.java#L56)）。

---

## 4. 检索层技术点

### 4.1 Nested 字段自动识别

PGC 数据大量使用 ES nested 类型（`poi_info_list`、`district_info_list`、`trunks`）。代码的处理方式是**按字段名里有没有 `.` 来自动判断**，有则自动套一层 `nested` 查询，path 取第一个 `.` 之前的部分（[AbstractRetrievalStrategyExecutor.java:447](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/impl/AbstractRetrievalStrategyExecutor.java#L447)、[:509](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/impl/AbstractRetrievalStrategyExecutor.java#L509)）。

调用方因此不用关心字段是不是 nested，写 `poi_info_list.poi_name^4.0` 就行。但这个约定有两个隐含限制：

- **多层嵌套会取错 path**。`a.b.c` 的 path 被取成 `a`，若真实 nested path 是 `a.b` 则查询报错。
- **普通 object 字段会被误判**。ES 里 `object` 类型的子字段同样写作 `parent.child`，但不能用 nested 查询。这类字段一旦出现就会失败，而失败是静默的 —— `parseFilterExpression` 的 catch 块直接 `return null`（[:96](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/impl/AbstractRetrievalStrategyExecutor.java#L96)），调用方设的过滤条件被丢弃却收不到任何提示，结果集会偏大。

另外 nested 查询没有带 `inner_hits`（全仓库无此调用），意味着**无法知道 nested 数组里究竟是哪个元素命中的**。对 KNN 检索尤其关键：向量存在 `trunks.embedding`，一篇文档有多个 chunk，检索返回的是整篇文档而非命中的那个 chunk。`convertResponseToDocList` 取的是文档级 `_source.chunk` 字段（[:549](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/impl/AbstractRetrievalStrategyExecutor.java#L549)），而写入侧从未写过这个字段 —— 只写了 `content` 和 `trunks[]`（[DocumentEmbeddingServiceImpl.java:389-442](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/impl/DocumentEmbeddingServiceImpl.java#L389)）。**所以 `DocEntity.chunk` 恒为 null，RAG 拿到的是整篇文章而不是精确片段**，这与 chunk 级检索的初衷相违。

### 4.2 KNN 用了 ES 8 top-level API

KNN 走的是 ES 8 的 top-level `knn` 子句而非 `script_score`，因此能用上 HNSW 索引（[KNNStrategyExecutor.java:113-127](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/impl/KNNStrategyExecutor.java#L113)）。查询向量支持两种来源：调用方直接传 `vector`（JSON 数组），或传文本由服务实时调 embedding 模型生成。

`numCandidates` 固定写成 `k * 2`。这个值偏小 —— ES 官方建议 `num_candidates` 应显著大于 `k`（常用 50~100 或 `k` 的 10 倍以上）以保证 HNSW 近似检索的召回率。`k=10` 时只搜 20 个候选，召回质量会有明显损失，且这个参数没有开放给调用方配置。

调试友好的一点：KNN 会把最终 DSL 序列化后打日志（[:135-139](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/impl/KNNStrategyExecutor.java#L135)），排查召回问题时很有用。

### 4.3 HYBRID 策略必然抛异常（已实测确认）

`HybridStrategyExecutor` 的 `scriptScore` 只给了 `query` 没给 `script`，`script` 部分整段被注释掉（[HybridStrategyExecutor.java:53-70](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/impl/HybridStrategyExecutor.java#L53)），类上留着 `// todo 待测试`。

ES Java client 的 builder 对必填属性有校验，实测结果：

```
MissingRequiredPropertyException: Missing required property 'ScriptScoreQuery.script'
```

**任何指定 `type: HYBRID` 的请求，该路必然失败**。所幸容错机制会把它降级成一条 warning，不会拖垮整个请求；`召回策略说明.md` 也明确标注了「⚠️ 本期暂不支持」。但 `RetrievalStrategyFactory` 仍然注册了它（[RetrievalStrategyFactory.java:73](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/RetrievalStrategyFactory.java#L73)），从 API 层面看它是「可用」的，且 `WeightedSumFusionStrategy` 还为 HYBRID 保留了 0.3 权重分支 —— 一条永远走不到的路径。

### 4.4 range 查询用字符串拼 JSON

范围查询没有用 builder，而是手工拼 JSON 再 `withJson` 解析（[AbstractRetrievalStrategyExecutor.java:180-198](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/impl/AbstractRetrievalStrategyExecutor.java#L180)）：

```java
StringBuilder jsonBuilder = new StringBuilder("{\"range\":{\"");
jsonBuilder.append(field).append("\":{");
```

推测原因是 ES 8.18 的 `RangeQuery` builder 对无类型字段较难构造。转义只处理了双引号（`value.replace("\"", "\\\"")`），字段名完全没有转义。

我实测了几个边界值（`a\b`、`a"b`、`1,2`）**都能正常解析**，所以这不是一个当前可触发的缺陷。但它是**结构性隐患**：过滤值来自调用方请求，字段名同样来自请求。一旦出现未预期字符导致 JSON 非法，`buildRangeQuery` 的 catch 直接 `return null`，过滤条件被静默丢弃，结果集悄悄变大。这类「静默失效」比抛异常更难排查。

---

## 5. 融合与排序：算法实现的正确性问题

### 5.1 WEIGHTED_SUM 惩罚了多路命中的文档（已实测确认）

公式是 `Σ(w_i × score_i) / n`，其中 `n` 是命中该文档的策略数（[WeightedSumFusionStrategy.java:87-93](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/fusion/impl/WeightedSumFusionStrategy.java#L87)）。

**除以 `n` 让这个算法的行为与融合目标相反**。多路召回的核心信号是「被多个通道同时命中 ⇒ 更可能相关」，而取平均恰好抹掉了这个信号，甚至反向惩罚。

实测（仓库自带单测 `testFuseResults_MultipleStrategies` 也固化了这个行为）：

| 文档 | 召回情况 | 融合得分 | 排名 |
| :--- | :--- | :--- | :--- |
| doc3 | 仅 KNN 命中，score 9.0 | 9.0×0.4 = **3.60** | 1 |
| doc1 | BM25(10.0) + KNN(6.0) **双路命中** | (10×0.3 + 6×0.4)/2 = **2.70** | 2 |
| doc2 | 仅 BM25 命中，score 8.0 | 8.0×0.3 = **2.40** | 3 |

双路命中的 doc1 排在只被单路命中的 doc3 之后。改成求和（不除 `n`）doc1 会得 5.40 升到第一。**这个单测把错误行为当成了正确预期锁定下来**，是比 bug 本身更需要注意的信号。

第二个问题：**权重是按 `RetrievalType` 取的硬编码常量**（KNN 0.4 / BM25 0.3 / HYBRID 0.3，[ServiceConstants.java:63-72](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/util/constant/ServiceConstants.java#L63)），不是按策略实例取的。在本仓库最典型的用法 —— **多路 BM25 查不同索引** —— 下，所有路权重都是 0.3，完全等价于不加权。而 `融合策略说明.md` 用整节篇幅（示例 3「自定义权重」）教调用方在 `params` 里传 `weight: "2.0"`，并称「系统会自动归一化权重」。我全仓库检索确认：**代码从未读取 `params` 里的 `weight`**，这个参数完全无效，文档是错的。

由于原始得分量纲还不一致（BM25 的 `_score` 无上界，KNN 相似度在 `[0,1]`），`WEIGHTED_SUM` 作为默认策略（[ResultFusionStrategyFactory.java:50](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/fusion/ResultFusionStrategyFactory.java#L50)）是个不太合适的选择。

### 5.2 RANK_FUSION 是 RRF，实现正确

`Σ 1/(K + rank_i)`，K=60（[RankFusionStrategy.java:53](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/fusion/impl/RankFusionStrategy.java#L53)），即标准 RRF。只用排名不用原始得分，天然免疫量纲不一致问题，且多路命中会累加得分 —— **没有 §5.1 的缺陷**。文档也推荐它用于 BM25+KNN 混合。

代码里叫 `CombSUM` 其实不准确（CombSUM 是得分求和，这里是 RRF），但不影响正确性。

**实践建议：显式传 `fusionRule: "RANK_FUSION"`，不要依赖默认值。**

### 5.3 兜底相似度排序会严重打乱顺序（已实测确认）

未指定 `sortEntityList` 时，不保持融合后的得分序，而是走 `DefaultSimilarityScorer` 重排（[GlobalSortProcessor.java:85-88](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/sort/GlobalSortProcessor.java#L85)）。这个 scorer 的类注释自称「占位实现 / 仅用于演示」，算法是关键词匹配率 60% + 词频 30% + 长度归一化 10%。

**致命点是分词：`query.trim().toLowerCase().split("\\s+")`** —— 按空白符切分（[DefaultSimilarityScorer.java:79](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/sort/similarity/DefaultSimilarityScorer.java#L79)）。中文 query 没有空格，整句被当作**一个关键词**做子串匹配。于是「内容里是否包含完整 query 字符串」成了主导因素。

实测 query =「黄山有什么好玩的」：

| 文档内容 | ES 得分 | 相似度 |
| :--- | :--- | :--- |
| 「黄山风景区四绝：奇松、怪石、云海、温泉，光明顶和迎客松最值得去。」 | 12.0 | **0.098** |
| 「有人问黄山有什么好玩的，我不知道。」 | 1.0 | **0.803** |

**真正有答案的文档相似度 0.098，答非所问但字面包含 query 的文档 0.803，相差 8 倍。**排序后无用文档排到第一，且这一步发生在 TopK 截取之前 —— 有价值的文档可能直接被截掉。融合和 ES 的打分成果在这里被覆盖。

对中文场景这是**当前最该修的问题**。最小改动是让兜底分支保持原有得分序（去掉相似度重排），或调用方始终显式传 `sortEntityList: [{"field": "_score", "order": "desc"}]`（样例请求里正是这么做的，可能因此掩盖了问题）。

### 5.4 过滤：字段缺失即判否

`GlobalFilterProcessor` 在字段值为 null 时直接 `return false`（[GlobalFilterProcessor.java:105-107](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/filter/GlobalFilterProcessor.java#L105)），即**字段缺失的文档一律被过滤掉**，与操作符无关。对 `!=` / `!contains` / `!in` 这些否定操作符，直觉预期通常相反（缺失应视为「不等于」而保留）。

另外过滤和排序都用一个 switch 白名单映射字段名（`id`/`source`/`locale`/`contentTime`/`content`/`chunk`/`score`/`sourceStrategy`/`sourceIndexName`），命中不了就打 warning 返回 null。加字段要同时改两处 switch —— `architecture.md` 的扩展指南也确实要求改两个地方。

值得肯定的是过滤/排序都统一按字符串取值再按需转数字（`compareValues` 先试 `Double.parseDouble` 再退化为字符串比较），逻辑简单一致；`contentTime` 这类 ISO 日期用字符串比较恰好是正确的。

---

## 6. 写入侧（embedding）技术点

写入链路：`toDocuments` → `DocumentProcessor`（清洗 + 双策略分块 + 文本去重）→ `ChunkEncoder`（批量编码 + L2 范数）→ `persistToElasticsearch`（bulk）。

### 6.1 双策略分块并存

句子分块（默认 160 字，BGE 推荐长度）和段落分块（320 字）**默认同时开启**（[ChunkingConfig.java:21-22](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/chunking/ChunkingConfig.java#L21)），两者结果合并后按文本去重（[DocumentProcessor.java:78-86](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/chunking/processor/DocumentProcessor.java#L78)）。同一段内容会以两种粒度各存一份向量，是「多粒度召回」的朴素实现 —— 换来召回覆盖，代价是索引膨胀约一倍。

长段落用滑动窗口切分，重叠 10%，并做了**智能边界处理**：向前最多找 50 个字符，落在句子结束符上断开，避免把句子切碎（[ParagraphChunkingService.java:176-186](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/chunking/impl/ParagraphChunkingService.java#L176)）。这个细节做得不错。

一处从 Python 移植遗留的痕迹：`sentence_endings` 默认值是 `"r'[。！？；.!?;]'"` —— 带着 Python 原始字符串字面量的 `r'` 和引号。`SentenceChunkingService` 只能靠 `substring(2, length-1)` 把包装剥掉才能得到正则（[SentenceChunkingService.java:54](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/chunking/impl/SentenceChunkingService.java#L54)）。调用方若通过契约传入正常正则（如 `[。！？]`），会被切掉首尾两字符变成 `。！`，行为完全错乱。

### 6.2 Embedding 调用的工程化较完整

`AbstractEmbeddingService` 是这次 `dev-embedding-refactor` 重构的产物，质量明显高于仓库其他部分：

- **批量请求**：按模型 `batchSize`（默认 32）分批，一批一次 HTTP。
- **空文本不发请求**，用占位保持返回顺序与输入严格一一对应（[:110-137](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/embedding/impl/AbstractEmbeddingService.java#L110)）。
- **按 `index` 字段回填**：不依赖 OpenAI 响应的数组顺序，优先用 `data[i].index` 映射回输入下标，并用 `filled[]` 校验无遗漏（[:292-308](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/embedding/impl/AbstractEmbeddingService.java#L292)）。这是很到位的细节 —— 顺序错乱会导致向量与文本错配，且难以发现。
- **429 指数退避**，优先采纳响应头 `Retry-After`（[:206-224](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/embedding/impl/AbstractEmbeddingService.java#L206)）。
- **配置热更新**：`EmbeddingConfig` 监听 QConfig `embedding-config.properties` 变更实时重建参数（[EmbeddingConfig.java:46-51](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/embedding/impl/EmbeddingConfig.java#L46)）。两个模型（Qwen3-Embedding-4B / bge-base-zh-v1.5）共用一份 `OpenAICompatEmbeddingService` 实现，差异全在配置，抽象干净。

两个遗留问题：

**每次请求新建 HttpClient**（[:259-263](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/embedding/impl/AbstractEmbeddingService.java#L259)）。`createHttpClient()` 在 `requestWithRetry` 的每次循环里被调用，连接池无法复用，每次都要重新 TCP + TLS 握手。这条路径在 KNN 检索的同步链路上（文本转向量），直接影响检索延迟。`HttpClient` 是线程安全的，应提为字段。

**重试放大**。默认 Qwen 5 次重试、基数 5000ms，退避 `baseDelay * 2^retryCount`。最坏情况单批耗时 5+10+20+40+80 = 155 秒。实跑单测时观察到 `testEmbedText_Qwen` 因 401 耗了 **311 秒** 才失败 —— 而 401 是不可恢复的认证错误，重试毫无意义。当前实现对所有非 429 异常一律重试（[:236-253](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/embedding/impl/AbstractEmbeddingService.java#L236)），4xx 应当直接失败。这条路径在检索链路上，意味着**模型端认证失效时 KNN 召回会挂几分钟而不是快速失败**。

### 6.3 写入侧与查询侧的向量维度不一致（重要）

这是一个会直接影响召回质量的不对称：

| 侧 | 调用 | 截断维度 |
| :--- | :--- | :--- |
| 写入 | `embedTexts(texts, null)`（[ChunkEncoder.java:93](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/chunking/processor/ChunkEncoder.java#L93)） | **null，不截断** |
| 查询 | `embedText(queryText, dims)`（[KNNStrategyExecutor.java:185-186](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/retrieval/strategy/impl/KNNStrategyExecutor.java#L185)） | 取 QConfig `<model>-truncation-dims` |

写入侧硬编码传 `null`，查询侧读配置。只要 QConfig 里配了 `truncation-dims`，**查询向量维度就会小于索引里的向量维度**，ES KNN 会因维度不匹配直接报错，整条 KNN 召回失效（会被降级成 warning，所以表现为「KNN 静默不出结果」而非报错）。

`ChunkingConfig` 里其实有个 `embeddingTruncationDims` 字段（默认 0），但**全仓库没有任何地方读取它** —— 我确认过没有 `getEmbeddingTruncationDims()` 调用。看起来是重构中途留下的未接通线头。

截断实现本身是正确的：硬截断后重新做 L2 归一化（[AbstractEmbeddingService.java:326-346](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/embedding/impl/AbstractEmbeddingService.java#L326)），符合 Matryoshka 表示学习的降维用法。

### 6.4 索引路由硬编码 + 文档 ID 覆盖风险

写入的目标索引是三元判断（[DocumentEmbeddingServiceImpl.java:144](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/impl/DocumentEmbeddingServiceImpl.java#L144)）：

```java
String indexName = "trip_trends_xhs".equalsIgnoreCase(key) ? "embedding_xhs" : "embedding_test";
```

读取侧已经通过 QConfig 做了 `dataType → index` 的配置化映射，写入侧却把索引名写死在代码里，而且默认落到名为 `embedding_test` 的索引 —— 任何新 source 的数据都会进「test」索引。这是明显的设计不对称，应该复用 registry 的映射。

bulk 用 `source_id` 作为文档 `_id`（[:323](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/impl/DocumentEmbeddingServiceImpl.java#L323)）。这带来幂等性（重复投递不会产生重复文档），但也意味着**同一 `source_id` 的重复投递会整篇覆盖**。因为 `trunks` 是整体重写的，若两次投递的分块配置不同，旧向量会被完全替换 —— 在这个场景下是期望行为，但需要调用方明确知道。

`content_createtime` 的时间归一化做得比较周全：依次尝试 ISO 8601 带时区 → 纯数字（10 位按秒、其余按毫秒）→ 空格/T 分隔的日期时间，全部失败则**不写该字段**而非写入脏值（[:456-485](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/impl/DocumentEmbeddingServiceImpl.java#L456)），避免了 ES 动态 mapping 被污染。

---

## 7. 参数校验与健壮性

`validateRequest` 只检查三项（[RagSearchServiceImpl.java:111-115](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/impl/RagSearchServiceImpl.java#L111)）：

```java
return request != null && request.getQuery() != null && request.getIntent() != null;
```

`intent` 被强制要求非空，但**全仓库除这一行外再无任何地方使用它** —— 我确认过没有其他 `getIntent()` 调用。它是个纯粹的形式门槛。同样未被使用的契约字段还有：`extQueryList`（扩展 query）、`slotList`（槽位识别结果）、`needExtraInfo`、`DocEntity.referenceInfo`。契约按更完整的设计定义，实现只落地了一部分。

**校验缺失导致两个可触发的 NPE**（我用探针程序实测确认，契约里 `topK` 是 `Integer`、`retrievalStrategies` 是 `List`，默认值均为 null）：

| 场景 | 位置 | 结果 |
| :--- | :--- | :--- |
| 不传 `topK` | `applyTopK(sortedResults, request.getTopK())`，形参是 `int`，自动拆箱 | `NullPointerException` |
| 不传 `retrievalStrategies` | `for (RetrievalStrategy s : strategies)` 遍历 null | `NullPointerException` |

两者都会被最外层 `catch (Exception e)` 兜住转成 500（[:98-100](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/service/impl/RagSearchServiceImpl.java#L98)），所以不会导致进程问题，但**语义错了**：这是调用方参数缺失，应该返回 400 + 明确提示，而非 500「内部错误」。让调用方去排查一个本该由参数校验拦下的问题，是不必要的成本。

另有一个未使用的 `validator` 包（`AbstractValidator` / `IValidate` / `Validation` / `ValidatorFactory`，共 4 个文件），是搭好但没接上的校验框架 —— 参数校验的位置本该在这里。

`EnvUtil.isLocal()` 的实现值得注意（[EnvUtil.java:28-30](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/util/EnvUtil.java#L28)）：

```java
public static Boolean isLocal(){
    return Env.PRO != getCurrentEnv();
}
```

**「非 PRO 即 local」**。这导致 FAT / UAT 环境下 `ElasticSearchClientRegistry` 会把 ES 地址强制改写成 `localhost:8070`（[ElasticSearchClientRegistry.java:96-101](../../pgc-rag-service/src/main/java/com/ctrip/cct/pgcragsearch/es/ElasticSearchClientRegistry.java#L96)）—— 这是本地 SSH 隧道端口。方法名 `isLocal` 与实际语义（`isNotProd`）不符，测试环境要连真实 ES 会被这段逻辑挡住。

---

## 8. 工程化现状

**实测基线**（本地 `mvn test`，2026-08-27）：

```
Tests run: 25, Failures: 0, Errors: 2
  编译：BUILD SUCCESS
  GlobalFilterProcessorTest          6 通过
  WeightedSumFusionStrategyTest      6 通过（但固化了 §5.1 的错误行为）
  DefaultSimilarityScorerTest       11 通过
  OpenAICompatEmbeddingServiceTest   2 失败（401，耗时 381 秒）
```

两个失败的 embedding 测试是**依赖真实模型服务的集成测试**，token 硬编码为 `sk-test-token`，没有 mock、没有 `@Disabled`、没有 profile 隔离。后果是 `mvn test` 在任何 CI 环境都会失败，且要耗掉 6 分钟才失败 —— 这会让 `only_allow_merge_if_pipeline_succeeds` 之类的门禁形同虚设。

测试覆盖集中在三个纯函数式组件（融合、过滤、相似度），**核心编排链路完全没有测试**：`RagSearchServiceImpl` 的 7 步流程、`RetrievalCoordinator` 的并行容错、`AbstractRetrievalStrategyExecutor` 的 600 行查询构建（仓库最大的类）、整个写入链路，都是零覆盖。`AppSpringBootConfigurationTests` 里 0 个 `@Test`。这解释了为什么 §5.1、§6.3 这类逻辑缺陷能留存下来。

**文档与代码的漂移**是另一个风险点。`soa/相关文档/` 下有约 22 万字符的中文文档（含 4 万字的召回策略说明、5.8 万字的过滤排序说明），加 `docs/` 下的架构与性能文档。密度很高，但已确认的漂移：

- `融合策略说明.md` 教的 `weight` 参数代码从不读取（§5.1）
- `HybridStrategyExecutor` 的参数表列了 `filterLocale`/`filterSource`，而召回策略文档另一处明确说这些参数「已移除」
- `docs/performance.md` 的性能数字（P95 178ms、QPS 1200）没有说明测量条件，且其中「Hybrid 52ms」一栏对应的是一条必然抛异常的代码路径 —— 这些数字应视为设计目标而非实测基线
- `docs/` 下混入了 3 篇与本服务无关的 MCP 服务手册（地理信息、天气预报）

代码内部的注释质量倒是相当好 —— 每个类都有职责说明、设计说明、算法公式、使用示例，`@see` 交叉引用完整。分层和命名也清晰（SOA → Service → Coordinator → Executor → Strategy，工厂 + 策略模式贯彻一致，`Collections.unmodifiableMap` + `EnumMap` 保证注册表线程安全）。**问题不在代码组织，而在算法正确性和参数校验。**

`git-ai` 出现在作者列表里（2 commits），主力是 shiyangliu（22）和 x_zhang53（19）。开发节奏上，2025-09 建仓做骨架，2026-01 密集补 KNN/embedding/重试（提交信息多为「fix」「log」「向量编码」），2026-08 做 embedding 重构 —— 最近一次重构的代码质量明显高于早期。

---

## 9. 综合评价

**架构骨架是好的**：分层清晰、策略/工厂模式贯彻到底、扩展点明确（加召回策略只需实现接口 + 注册）、容错与降级设计成熟（两级降级 + 告警旁路 + 三套监控，降级必留痕）。`AbstractEmbeddingService` 的批量/重试/顺序保证做得专业。这些是可以直接借鉴的部分。

**问题集中在算法正确性和输入校验**，而且几个问题都在关键路径上：

| 严重度 | 问题 | 影响 |
| :--- | :--- | :--- |
| 高 | 中文兜底相似度排序（§5.3） | 不传 `sortEntityList` 时排序被打乱，好文档可能被 TopK 截掉。实测相关文档 0.098 vs 无关文档 0.803 |
| 高 | 写入/查询向量维度不对称（§6.3） | QConfig 配了 `truncation-dims` 就会让 KNN 召回静默全失效 |
| 高 | `WEIGHTED_SUM` 除以命中数（§5.1） | 默认融合策略反向惩罚多路命中，且被单测固化 |
| 中 | `DocEntity.chunk` 恒为 null（§4.1） | 返回整篇文档而非命中片段，与 chunk 级检索初衷相违 |
| 中 | 4xx 也重试（§6.2） | 认证失效时 KNN 召回挂数分钟而非快速失败（实测 311 秒） |
| 中 | 缺参数校验导致 NPE→500（§7） | 调用方参数问题被报成服务端内部错误 |
| 中 | 集成测试无隔离（§8） | `mvn test` 必失败且耗时 6 分钟，CI 门禁失效 |
| 低 | HYBRID 必然抛异常（§4.3） | 有容错兜底 + 文档已标注，但 API 层面仍「可用」 |
| 低 | 写入索引名硬编码（§6.4） | 新 source 默认落 `embedding_test` |
| 低 | `isLocal()` 实为 `isNotProd()`（§7） | FAT/UAT 的 ES 地址被改写成 localhost |

**如果只做三件事**：修中文分词导致的兜底排序（§5.3，或让兜底分支保持得分序）、打通写入侧截断维度（§6.3）、`WEIGHTED_SUM` 去掉除以 `n` 并同步修单测（§5.1）。

**作为调用方的自保姿势**：显式传 `fusionRule: "RANK_FUSION"`、显式传 `sortEntityList: [{"field":"_score","order":"desc"}]`、显式传 `topK` 和 `retrievalStrategies`、不要用 `HYBRID`、每次都检查响应里的 `warnings`。

**定位判断**：这是一个**骨架成熟但算法层未经充分验证**的服务。适合作为多索引并行召回的架构参考，但直接复用其融合与排序实现需要先修上述问题。核心检索链路缺测试是这些问题能长期留存的根本原因 —— 补测试的收益会高于继续加功能。

---

## 附：速查

| 项 | 值 |
| :--- | :--- |
| 仓库 | `cct/pgc-rag-service`，本地 `/Users/Floweryu/Workspace/pgc-rag-service` |
| Maven | `com.ctrip.cct:pgc-rag-search:0.0.1-SNAPSHOT`，`war` 包 |
| appId | 100068080 |
| SOA | `PgcRagSearch`，namespace `http://soa.ctrip.com/36651` |
| 契约 jar | `com.ctrip.cct.pgc:pgc-rag-search:0.0.4`（外部生成，非本仓库） |
| 技术栈 | Java 21 / Spring Boot 3 / ES 8.18 / QMQ 6.3.14 / QConfig / CAT / HickWall / Clog |
| QConfig 文件 | `es-config.json`、`embedding-config.properties` |
| Embedding 模型 | `bge-base-zh-v1.5`（默认）、`qwen3-embedding-4b` |
| 并行线程池 | 固定 10（`RETRIEVAL_THREAD_POOL_SIZE`） |
| 融合默认 | `WEIGHTED_SUM`（建议改用 `RANK_FUSION`） |
| ES 写入索引 | `embedding_xhs` / `embedding_test`（硬编码） |
| 向量字段 | `trunks.embedding`（nested，`dense_vector`） |
| Clog title | `RAGSearch` / `.Retrieval` / `.Fusion` / `.Filter` / `.Sort` / `.ES` |
| HickWall 前缀 | `pgc.rag.*` |
| CAT type | `RetrievalStrategy` / `EmbeddingCall` / `EsBulkWrite` |
| 关联仓库 | `cct/activity-rag-agent`（Agent 侧调用方） |
