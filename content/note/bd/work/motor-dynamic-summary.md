---
draft: "true"
---

# Motor Dynamic Summary：算法设计（给算法工程师版）

## 1. 目标任务拆解

系统整体可以理解为“检索 + 内容理解 + 结构化特征/参配 + 多分数融合”的服务化封装：

- 摘要：query 与内容（title/docs/asr/ocr/正文）→ abstract + no_answer_score
- 相关性：query 与内容 → rel_score（可基于 chunk/全文/summary 多视角）
- 重排：query 与候选 summary → bge_ranker_score
- 规划/参配：query → select/where（结构化）与参数召回（markdown/prompt parts）

## 2. Mermaid 视图

### 2.1 整体算法流水线

```mermaid

flowchart TB

Q[Query] --> P[Query Plan / Params\nLLM+规则]

P --> R1[候选召回\n搜索/向量/KG/多模态]

R1 --> C[候选内容构建\nFeatureContext]

C --> S1[SP 摘要/可回答性\nabstract + no_answer_score]

C --> S2[相关性\nrel_score]

C --> S3[Rerank\nbge_ranker_score]

S1 --> F[融合/过滤/排序\noverall_score]

S2 --> F

S3 --> F

F --> O[TopK 输出\n摘要/特征/结构化结果]

```

### 2.2 决策打分融合（overall_score）

```mermaid

flowchart LR

A[answer_score\n=1-no_answer_score] --> G{answer_score < 0.3?}

B[bge_ranker_score] --> G

G -->|是| L1[overall=max【0,answer-0.2】+bge*0.2]

G -->|否| L2[overall=0.3+min【0.1,answer-0.3】+bge*0.6]

```

### 2.3 schema_retrieve 融合（字段相关性）

```mermaid

flowchart LR

Q[question_query] --> BM25[BM25【query, schema_relevance】]

Q --> SP[SP 可回答性\n【对 schema_relevance 询问】]

BM25 --> N1[min-max normalize]

SP --> N2[min-max normalize]

N1 --> SUM[rank_score = bm25_norm + answer_norm]

N2 --> SUM

SUM --> TOPK[TopK schema_select 拼接进 prompt]

```

## 3. 统一数据结构（算法侧接口）

- FeatureContext：承载“一个候选内容”的所有可用信号（文本、多模态字段、分数与特征）。
- thrift 定义：[FeatureContext](file:///cloudide/workspace/motor_dynamic_summary/idls/idl/motor_dynamic_summary.thrift#L48-L70)
- 典型生产：内容云 join + SP 摘要回填，见 [feature_join_handle.py](file:///cloudide/workspace/motor_dynamic_summary/handler/feature_join_handle.py#L23-L155)
- ReadingSummary：用于“多 query、多候选”的决策打分与选择。
- overall_score 由 answer_score 与 bge_ranker_score 融合，见 [decision_handle.py](file:///cloudide/workspace/motor_dynamic_summary/handler/decision_handle.py#L27-L88)

## 4. 核心分数定义与融合

### 3.1 Answer score（可回答性/摘要模型输出）

- 来源：SP 摘要模型输出 SPSummaryModelResult.no_answer_score
- 定义：answer_score = 1 - no_answer_score
- 使用场景：
- ReadingComprehension 的候选摘要与答案分数（决策/过滤）
- schema_retrieve 的相关性近似（见 5.1）

### 3.2 Rerank score（BGE/语义重排）

- 来源：rerank_relevance_score_feature(all_queries, feature_contexts)
- 作用：对候选的 summary（或 content）做 query-候选语义一致性打分，进入 overall。

### 3.3 Overall score（决策融合规则）

实现为分段函数（保证 answer_score 很低时不被 rerank 过度抬高）：

- 若 answer_score < 0.3：overall = max(0, answer-0.2) + bge * 0.2
- 否则：overall = 0.3 + min(0.1, answer-0.3) + bge * 0.6

代码：[decision_handle.py](file:///cloudide/workspace/motor_dynamic_summary/handler/decision_handle.py#L68-L77)

## 5. 规划与参配（Plan / ParamsRecall）

### 5.1 schema_retrieve（字段检索：answer_score + BM25）

在参数规划中，为了把“可能相关的字段 schema”拼进 prompt，使用两路信号融合：

- answer_score：用 SP 模型判断 query 是否能从 schema_relevance 被回答
- bm25_score：query 与 schema_relevance 的 BM25
- rank_score = norm(answer_score) + norm(bm25_score)

代码入口：[plan.py:schema_retrieve](file:///cloudide/workspace/motor_dynamic_summary/llm/plan.py#L288-L381)

### 5.2 qwen 规划输出 → 结构化 params

- 输入：拼接 select_schema/where_schema + query（并注入 DA 的标准车系名可选）
- 模型：qwen_8b_plan（HTTP 推理）
- 解析：parse_multiple_params(qwen_output)
- 映射：process_select_params 将中文字段映射到内部 params_name，并展开聚合属性 include_property

入口：[plan.py:car_base_plan_qwen](file:///cloudide/workspace/motor_dynamic_summary/llm/plan.py#L104-L149)

### 5.3 ParamsRecall（两套版本）

- base/llm_v1：传统召回 + 澄清/改写 + markdown 输出（见 [params_recall_handle.py](file:///cloudide/workspace/motor_dynamic_summary/handler/params_recall_handle.py)）
- qwen_v1：大模型抽取 car_model_text + select_property_list，再驱动参配召回（见 [params_recall_handle_pro.py](file:///cloudide/workspace/motor_dynamic_summary/handler/params_recall_handle_pro.py#L713-L760)）
- need_prompt=true：走 KGRAG 拼接 params_text/oneshot/control/prompt_parts，便于上游直接喂给生成模型
- KGRAG 实现：[parameter_llm_model.py](file:///cloudide/workspace/motor_dynamic_summary/car_series_feature/parameter_llm_model.py)

## 6. ChunkRecall（向量召回 + query-match + 生成摘要/打分）

流程（简化）：

1) query → embedding_encode

2) VikingDB recall → keyword 主题簇

3) Redis mget 拉取缓存 doc（按 keywords-gid 键）

4) LLM 做“keyword ↔ query(含 rewrite)”语义匹配，产出 query2docs

5) 对每个 query：生成摘要 + 相关性 + 最终排序，去重后返回 topk

入口：[chunk_recall_handle.py:chunk_recall_process](file:///cloudide/workspace/motor_dynamic_summary/handler/chunk_recall_handle.py#L181-L220)

算法侧可优化点：

- match 阶段可替换为轻量 cross-encoder 或规则/实体约束，降低 LLM 成本
- 排序阶段显式融合：answer_score、rel_score、freshness、站点黑名单等（当前有部分规则散落在 handler）

## 7. KG/Tp Recall（图谱观点召回）

- 输入：series_list + ents（观点/属性词列表）
- 下游：kg_recall_proxy
- 输出：优点/缺点/一句话总结 + 节点观点明细（opinion_name/polarity/basis/score）

入口：[kg_recall_handle.py](file:///cloudide/workspace/motor_dynamic_summary/handler/kg_recall_handle.py#L80-L185)

## 8. 训练/评估与线上诊断建议

- 分数校准：answer_score 与 bge_ranker_score 的分布漂移要监控（按场景、query 类别、内容类型分桶）
- 离线评估：
- Decision：Top1 命中率/Pairwise accuracy/Calibration
- Recall：NDCG@k/Recall@k/去重率/站点黑名单覆盖
- Plan：字段召回覆盖率（select/where），where 约束正确率（可参考 plan_checker）
- 线上可观测字段（建议写入 extra_info）：模型版本、prompt 版本、关键中间输出（plan xml、match pairs、topk 分数列表）。

## 9. 面试怎么讲这套服务

如果面试官让我概括 `motor_dynamic_summary`，我不会把它讲成一个“摘要服务”，而会讲成**选车问答里的算法能力中台**。它对上承接 `select_car_feature_server` 这类业务编排服务，对下封装内容云、摘要模型、相关性模型、reranker、向量库、Redis 和 LLM 网关，输出的不是单一能力，而是一组可复用的结构化算法接口：摘要、可回答性、字段检索、参配召回、ChunkRecall、KGRecall 和决策打分。

再往下一层讲，这个服务解决的是“业务服务不应该自己拼模型能力”这个问题。业务方真正关心的是：给你一个 query 和一批候选内容，你能不能给我可排序的分数；给你一个选车 query，你能不能把相关 schema、参配字段和 prompt 片段准备好；给你一组车系和观点词，你能不能返回可直接用在回答里的优缺点依据。`motor_dynamic_summary` 把这些问题收敛成 RPC，不让上层服务直接碰一堆异构下游。

我认为这套设计最值得讲的地方有三个。

- 第一，它没有把问题粗暴地做成“一个大模型直接端到端生成”。摘要、相关性、重排、字段检索、参配召回被拆成多个可观测模块，代价是系统复杂一些，但收益是每个环节都能独立评估和替换。
- 第二，它的很多打分不是单模型真理，而是**多信号融合**。最典型的是 `overall_score`，本质上是在回答“一个候选既要能答，也要和 query 真相关”。只看 `bge_ranker_score` 会把语义相近但答不上来的内容排太前，只看 `answer_score` 又容易把模板化可回答内容抬高，所以这里用了分段融合，而且对低 `answer_score` 做了抑制。
- 第三，它把 Plan 也做成了检索问题。`schema_retrieve` 不是把所有字段一股脑塞进 prompt，而是先用 `answer_score + BM25` 做字段相关性召回，再把 TopK schema 塞给规划模型。这一步直接决定了 Plan 的上下文质量，很多后续 XML / params 错误其实都不是生成能力问题，而是字段候选池一开始就给偏了。

如果面试官继续追问“你个人觉得最像算法贡献的点是什么”，我会重点讲两个。

- 一个是 `schema_retrieve`。这里把 QA 可回答性信号和 BM25 字面信号融合起来，本质是在做字段级 schema linking，只不过不是经典 Text2SQL 那种表列检索，而是面向选车属性体系的中文字段召回。这个设计的好处是对主观描述和短 query 都更稳，`BM25` 保字面命中，`answer_score` 保语义可回答性。
- 另一个是 `overall_score`。它不是 learned fusion，而是人工设计的分段函数，背后假设很明确：可回答性太差的候选，再高的 rerank 也不能被抬到太前。这个规则看起来“土”，但线上系统里往往比一个没校准好的 learned scorer 更稳，因为分布漂移时更容易解释和治理。

从系统边界上看，它和 `select_car_feature_server` 的关系也很适合面试里主动讲清楚。前者更像业务主链路，负责 QueryAnalysis、候选车系召回、排序编排和协议返回；后者更像算法能力底座，负责 Plan、参配、内容理解、分数和 prompt 组件。把这层职责边界主动讲出来，面试官一般会直接知道你不是只会堆模型，而是在看系统怎么拆。

## 10. 面试高频追问

### Q1：`motor_dynamic_summary` 和上层业务服务为什么要拆开，不能全放在一个服务里？

因为这里封装的是跨场景复用的算法原子能力，而不是某一个业务流程。摘要、相关性、rerank、参数召回、ChunkRecall、KGRecall 这些能力不只服务选车 dialog，也服务搜索、问答、图文内容理解。如果全塞在业务服务里，复用性差，模型和 prompt 升级也会把业务服务一起拖着发版。

### Q2：`overall_score` 为什么不直接学一个融合模型，而要写分段规则？

这里更像一个校准问题，不只是拟合问题。我们最担心的是低 `answer_score` 的候选被高 `bge_ranker_score` 误抬，所以先用规则把“不可回答”这件事压住。在线上真实系统里，这种可解释的 hard bias 往往比一个离线分高、线上一漂就失控的 learned fusion 更稳。后面当然可以做 learned weights，但前提是先把 label 和线上监控做扎实。

### Q3：`answer_score = 1 - no_answer_score` 这个定义为什么合理？

因为下游 SP 模型天然给的是“答不上来”的概率或倾向，而业务排序真正需要的是“这个候选能不能回答 query”。两者是单调可逆的，直接取 `1 - no_answer_score` 最简单，也便于和其他越大越好的分数融合。这里的重点不是数学形式，而是把“可回答性”显式做成一个独立维度。

### Q4：为什么 `schema_retrieve` 里要把 BM25 和 answer score 融合，而不是只留 embedding 或只留 BM25？

只留 BM25 会对中文短 query 和主观表达过敏，字面没命中就很容易漏；只留语义分又会把一些看起来像相关、但其实字段边界不对的 schema 捞上来。字段检索这件事本身就是“字面约束 + 语义约束”的混合问题，所以这里做两路融合是合理的。

### Q5：这里为什么用 min-max normalize，而不是 softmax？

因为这里不需要概率解释，只需要把两路信号放到可加的同尺度空间里。softmax 会引入相对放大效应，尤其候选池规模变化时更不稳定；min-max 更像一个排序前归一化，足够朴素，也更容易 debug。

### Q6：为什么 ParamsRecall 要保留两套版本，传统版和 qwen 版同时存在？

这不是代码没清理干净，而是线上系统典型的“双轨制”。传统版稳定、可控、成本低，适合兜底；qwen 版抽取能力强，更适合复杂 query 和 need_prompt 场景。保留双轨意味着新链路效果没完全证实前，业务不会被一次模型回归拖死。

### Q7：`need_prompt=true` 这条链路的价值是什么？

它说明这个服务不只输出“数据”，还输出可直接被上游生成模型消费的 prompt 组件，比如 `params_text`、`oneshot`、`control`、`prompt_parts`。这会让上游业务服务更轻，因为不用自己再拼一层 prompt 规则，同时也让 prompt 版本管理更集中。

### Q8：ChunkRecall 里为什么还要让 LLM 做 `keyword ↔ query` 匹配，看起来很贵？

因为向量召回出来的是 keyword 主题簇，不是最终 query-doc 对齐结果。这里加一层 LLM 匹配，本质是在做 query rewrite 后的语义过滤，能压掉不少主题簇误召。但这一步确实贵，所以文档里也明确说了可以往轻量 cross-encoder 或实体约束迁。

### Q9：KGRecall 和 ChunkRecall 的职责边界怎么讲？

ChunkRecall 处理的是非结构化内容片段，偏“文章/文档里怎么说”；KGRecall 处理的是观点和属性节点，偏“图谱里有哪些可归纳的优缺点与依据”。前者更适合开放文本召回，后者更适合结构化观点总结。两者都可能服务回答，但证据形态不同。

### Q10：这套服务最容易漂移的指标是什么？

我认为是 `answer_score`、`bge_ranker_score` 和 schema retrieval 的 TopK 命中分布。因为这几个点一旦漂，往往上层业务先表现为“回答越来越像、但不准”或者“Plan 结构没错但字段越来越偏”，排障成本很高。所以分桶监控和关键中间输出落盘非常重要。

### Q11：如果线上 badcase 变多，你会先查哪里？

我会按链路拆：先看 schema retrieval 的 TopK 是否偏了，再看 qwen plan 输出有没有字段幻觉，再看 ParamsRecall 是否 fallback 频繁，最后看 decision / rerank 分布是否异常。原因很简单，Plan 选错字段和召回拿错候选，后面的总结模型再强也救不回来。

### Q12：这套服务如果要进一步演进，你会优先做什么？

我会优先做三件事：第一，把分数融合从散落在 handler 的规则收拢成统一配置；第二，把 `schema_retrieve`、ChunkRecall、Decision 这三条链路的离线评测和线上监控打通；第三，逐步把 LLM-only 的昂贵步骤替换成更轻的检索或 cross-encoder。前两件事决定系统可治理，第三件事才是成本优化。

---

# Motor Dynamic Summary：系统设计（人能看懂版）

## 1. 系统是什么

本项目提供一个 Thrift RPC 服务（Euler Server），对汽车/摩托搜索与内容理解场景输出：

- 内容摘要（单条/批量，图文/视频）
- 内容特征拼接（FeatureContext）与若干评分（相关性/时效/一致性等）
- 车系/车款参配与车系特征召回（含澄清与 Prompt 组装）
- 若干辅助能力：Query 拆解、Chunk 向量召回、多模态召回、KG 召回、Prompt 组件拼装

入口与 RPC 注册：

- [server.py](file:///cloudide/workspace/motor_dynamic_summary/server.py)
- Thrift 契约：[motor_dynamic_summary.thrift](file:///cloudide/workspace/motor_dynamic_summary/idls/idl/motor_dynamic_summary.thrift)

## 2. 组件分层

- Server 层：Euler + gevent，负责 RPC 注册、异常兜底、日志。
- Handler 层：每个 RPC 对应一个 handler，做入参校验、组织下游调用、组装 thrift response。
- RPC/Client 层：对外部系统（内容云、向量库、模型服务、Redis、HTTP 搜索 API）做封装。
- 配置层：大量策略/模板/映射都在 config/ 下以 JSON 形式加载（参配 schema、prompt 模板、黑名单等）。

## 3. Mermaid 视图

### 3.1 架构总览

```mermaid

flowchart TB

%% Clients

U[上游调用方\n【搜索/问答/服务编排】] -->|Thrift RPC| S[DynamicSummaryServer\nserver.py]

  

subgraph Server[服务进程]

S --> H1[handler/*\n入参校验/编排/组装Resp]

H1 --> C1[rpc/*\n下游Client封装]

H1 --> CFG[config/*\nJSON策略/模板/映射]

end

  

%% Downstream

C1 --> A[内容云 Article 服务\n【标题/正文/ASR/OCR/属性】]

C1 --> SP[SP 摘要/可回答性模型\n【no_answer_score/abstract】]

C1 --> REL[相关性模型\n【rel_score】]

C1 --> RERANK[BGE reranker\n【bge_ranker_score】]

C1 --> EMB[Embedding 服务]

C1 --> VDB[VikingDB 向量库\n【ChunkRecall】]

C1 --> RDS[Redis\n【缓存/pgc/prompt】]

C1 --> HTTP[HTTP 搜索 API\n【MotorSearchSortRecall/MultiModal】]

C1 --> LLM[大模型网关\n【plan/匹配/抽取】]

  

%% Data

H1 --> FC[FeatureContext\n跨模块内容载体]

H1 --> RS[ReadingSummary\n决策候选结构]

```

### 3.2 关键 RPC → handler 映射

```mermaid

flowchart LR

S[server.py register] --> DS[GetDynamicSummary\nvideo_summary_handle.summary_handler]

S --> VDS[GetVideoDynamicSummaryByGid\nvideo_summary_handle.video_summary_handler]

S --> FJ[GetAttrFeatureByGids\nfeature_join_handle.feature_join_handler]

S --> FR[CalcBatchFreshness\nfeature_join_handle.freshness_handler]

S --> PR[ParamsRecall\nparams_recall_handle_pre/_pro]

S --> DC[MakeDecision\ndecision_handle.decision_handler]

S --> CR[ChunkRecall\nchunk_recall_handle.chunk_recall_process]

S --> KG[KGRecall/TpRecall\nkg_recall_handle.*]

S --> QD[GetPlanDetachQueries\nquery_plan_detach_handle]

S --> QO[GetObjectivityDetachQueries\nquery_objectivity_detach_handle]

```

## 4. 关键数据结构

- FeatureContext：跨模块的“内容载体”，包含 gid/title/docs/asr/ocr/summary/features/relevance 等，用于摘要、评分与后续决策。
- 定义见 [motor_dynamic_summary.thrift:FeatureContext](file:///cloudide/workspace/motor_dynamic_summary/idls/idl/motor_dynamic_summary.thrift#L48-L70)
- ReadingSummary：面向“多 query、多候选内容”的阅读理解/决策结构，用于计算 answer/relevance/rerank 并输出 overall。
- 定义见 [motor_dynamic_summary.thrift:ReadingSummary](file:///cloudide/workspace/motor_dynamic_summary/idls/idl/motor_dynamic_summary.thrift#L78-L91)

## 5. 典型调用链（从用户视角）

### 5.1 单条摘要（GetDynamicSummary）

```mermaid

sequenceDiagram

autonumber

participant U as 上游调用方

participant S as server.py【GetDynamicSummary】

participant H as video_summary_handle.summary_handler

participant M as V3Client【摘要模型】

  

U->>S: DynamicSummaryReq【query,title,docs】

S->>H: summary_handler【req】

H->>M: predict【query,title,docs】

M-->>H: topic, summary

H-->>S: DynamicSummaryResp【topic,summary】

S-->>U: DynamicSummaryResp

```

- 输入：query + title + docs
- 处理：调用摘要模型 client（V3Client）
- 输出：topic + summary
- 代码：
- RPC 包装：[server.py:GetDynamicSummary](file:///cloudide/workspace/motor_dynamic_summary/server.py#L77-L92)
- 业务实现：[video_summary_handle.py:summary_handler](file:///cloudide/workspace/motor_dynamic_summary/handler/video_summary_handle.py#L50-L66)

### 5.2 批量特征拼接/摘要（GetAttrFeatureByGids）

```mermaid

sequenceDiagram

autonumber

participant U as 上游调用方

participant S as server.py【GetAttrFeatureByGids】

participant H as feature_join_handle.feature_join_handler

participant A as 内容云/Article Client

participant SP as SP摘要模型

participant REL as 相关性模型

  

U->>S: GetFeatureBatchReq【query,gid_batch,need_summary,need_features】

S->>H: feature_join_handler【req】

H->>A: feature_batch_join【gid_batch】

A-->>H: FeatureContext[] 【title/docs/asr/ocr/features...】

alt need_summary=true

H->>SP: get_search_plugin_summary【batch_inputs】

SP-->>H: abstract_results + sp_summary_model_result

H->>H: 回填 summary / no_answer_score

end

alt need_features包含relevance_score

H->>REL: relevance_score_for_feature_context_chunk【...】

REL-->>H: gid->RelevanceInfo【rel_score】

end

H-->>S: GetFeatureBatchResp【feature_infos】

S-->>U: GetFeatureBatchResp

```

- 输入：query + gid_batch（可选 title/docs 直接传入）
- 处理：内容云取 title/docs/asr/ocr + 结构化特征；可选调用 SP 摘要；可选相关性计算。
- 输出：FeatureContext 列表
- 代码：[feature_join_handle.py:feature_join_handler](file:///cloudide/workspace/motor_dynamic_summary/handler/feature_join_handle.py#L23-L155)

### 5.3 参配/车系特征召回（ParamsRecall / CarSeriesFeatureJoin）

```mermaid

sequenceDiagram

autonumber

participant U as 上游调用方

participant S as server.py【ParamsRecall】

participant H as params_recall_handle_pre.series_params_recall_handler_ppre

participant KG as KGRAG【parameter_llm_model】

participant P as params_recall_handle.series_params_recall_handler【传统召回】

participant DA as DA/SPO/车系识别

  

U->>S: ParamsRecallReq【question_query,need_prompt,extra_info】

S->>H: series_params_recall_handler_ppre【req】

H->>DA: 【可选】 多车系改写/识别

  

alt need_prompt=true

H->>KG: summary_predict / summary_construct_carLevel【...】

KG-->>H: prompt, params_text, oneshot, control, clarify/rethink, confidence

alt params_text为空且满足fallback条件

H->>P: series_params_recall_handler【req】

P-->>H: markdown_text + clarify/rethink + extra_info

end

else need_prompt=false

H->>P: series_params_recall_handler【req】

P-->>H: markdown_text + clarify/rethink

end

  

H-->>S: ParamsRecallResp【prompt/query_params_map/...】

S-->>U: ParamsRecallResp

```

- ParamsRecall：根据 query/series_id 等返回 markdown 或 prompt 片段；支持澄清与改写。
- RPC：[server.py:ParamsRecall](file:///cloudide/workspace/motor_dynamic_summary/server.py#L391-L422)
- 默认实现（need_prompt 支持）：[params_recall_handle_pre.py](file:///cloudide/workspace/motor_dynamic_summary/handler/params_recall_handle_pre.py)
- 车系特征 pipeline：[feature_join_handle.py:car_series_feature_handler](file:///cloudide/workspace/motor_dynamic_summary/handler/feature_join_handle.py#L176-L200)

### 5.4 决策（MakeDecision）

```mermaid

sequenceDiagram

autonumber

participant U as 上游调用方

participant S as server.py【MakeDecision】

participant H as decision_handle.reading_comprehension

participant SP as SP摘要模型

participant RR as BGE reranker

participant REL as 相关性模型

  

U->>S: ReadingComprehensionReq【queries, infos, need_features】

S->>H: reading_comprehension【queries, infos, need_features】

  

alt need_features包含answer_score

H->>SP: get_search_plugin_summary【batch_inputs】

SP-->>H: abstract + no_answer_score

H->>H: 回填 summary + answer_score

end

  

alt need_features包含bge_ranker_score

H->>RR: rerank_relevance_score_feature【all_queries, feature_contexts】

RR-->>H: bge_ranker_score[]

H->>H: 计算 overall_score

end

  

alt need_features包含relevance_score

H->>REL: relevance_score_for_feature_context【query, feature_contexts】

REL-->>H: rel_score

end

  

H-->>S: ReadingComprehensionResp

S-->>U: ReadingComprehensionResp

```

- 输入：多个 queries + 每个 query 的候选 ReadingSummary 列表
- 处理：调用 SP 摘要得到 answer_score、调用 reranker 得到 bge_ranker_score，拼 overall_score；可选 relevance。
- 代码：[decision_handle.py](file:///cloudide/workspace/motor_dynamic_summary/handler/decision_handle.py)

### 5.5 ChunkRecall（向量召回 + 匹配 + 摘要/打分）

```mermaid

sequenceDiagram

autonumber

participant U as 上游调用方

participant S as server.py【ChunkRecall】

participant H as chunk_recall_handle.chunk_recall_process

participant E as Embedding

participant V as VikingDB

participant R as Redis【cache_sp_*】

participant L as LLM【关键词-问题匹配】

participant SP as SP摘要模型

participant REL as 相关性模型

  

U->>S: ChunkRecallReq【query,rewrite_queries,topk】

S->>H: chunk_recall_process【query, rewrite, topk】

H->>E: embedding_encode【[query]】

E-->>H: query_vector

H->>V: recall【query_vector, topk=5】

V-->>H: keyword_clusters

H->>R: mget【cache_sp_{keywords-gid}】

R-->>H: docs_by_keyword

H->>L: llm_cal_query_relevance【queries, keywords】

L-->>H: match_pairs【query_idx, keyword_idx】

loop 每个匹配到的 query

H->>SP: get_search_plugin_summary

SP-->>H: abstract + no_answer_score

H->>REL: relevance_score_for_feature_context

REL-->>H: rel_score

H->>H: 计算最终分数并排序

end

H-->>S: ChunkRecallResp【items】

S-->>U: ChunkRecallResp

```

## 6. 外部依赖（上线必备）

- 内容云（Article 服务）：用于 gid → title/content/asr/ocr/基础特征。
- 摘要/可回答性模型（SP）、相关性/重排模型（bge reranker）、embedding 服务。
- 向量库 VikingDB：ChunkRecall 使用。
- Redis：缓存与部分特征/pgc prompt。
- HTTP 搜索 API：MotorSearchSortRecall / MultiModalRecall。

建议工程实践：将所有外部依赖统一封装成可替换 Client，并支持 offline stub（本地无内网也可跑通主链路）。

## 7. 运行与稳定性

- 并发模型：gevent + monkey patch；部分 handler 内部用 gevent 并发请求下游。
- 异常处理：server.py 对大多数 RPC 做 try/except，失败时写日志并返回 StatusCode=-1。
- 超时/重试：下游 client 各自实现不一致，需要统一治理（建议集中配置）。

## 8. 风险与治理建议

- 密钥/Token：当前代码存在硬编码 Key/Token 的风险，应迁移到环境变量或密钥服务，并禁止日志输出敏感信息。
- 可观测性：建议为每个 RPC 统一记录 req_id、耗时、下游错误码、关键分数分布（answer/relevance/rerank）。
- 契约一致性：thrift service 与 server.register 方法需定期对齐，避免线上/线下行为漂移。
