# 【RAG 模型选型评测报告】SKILL

> **适用产品类别**：**B 类（RAG 知识库 / 问答产品）为主**——这是 RAG 选型的原生场景，每次换 LLM / Embedding / Rerank 都要出这份报告；**D 类（评测平台 / 内部工具）为辅**——评测平台要把 RAG 选型报告作为一类标准模板；A 类（对话式 Agent）只在 Agent 里挂 RAG、且 RAG 选型直接影响 Agent 效果时引用。C 类 AIGC 生成、E 类 B 端 SaaS AI 化一般不单独出 RAG 选型报告，引用 B 类结论即可。
> **文档目的**：把一次 RAG 三类模型（LLM / Embedding / Rerank）选型评测的"跑了多少题、谁赢谁输、为什么输、成本多少、推荐哪一组"讲清楚，让决策者直接做"上哪一套组合、其他组为什么落选、上线后监控什么"的判断，让算法知道下一步该调哪一段。这是一份**决策文档**，不是数据归档。读者是 PM、算法 / AI 工程负责人、业务决策者、QA。
> **撰写时机**：① RAG 模块 0→1 阶段，第一版知识库入库后、上线前；② 任何一类模型换版 / 新模型发布需要复评时；③ 线上 Faithfulness 跌破阈值、触发重评流程时。这是 RAG 项目里**唯一一份**直接决定"线上跑哪套模型组合"的文档。

---

## 〇、先选范式：RAG 选型报告 = 通用选型报告 + RAG 专项

通用《07_评测报告》的范式 A（大模型 / 能力选型类）是本报告的底座，但 RAG 选型比单模型选型多了 3 件事：
1. **三类模型联合选型**——不是选 1 个 LLM，而是同时选 LLM + Embedding + Rerank 的组合；
2. **RAG Triad 三维指标**——Faithfulness / Context Precision / Context Recall，不能只看端到端答案；
3. **检索层和生成层分开归因**——bad case 必须分清是"Embedding 没召回"还是"Rerank 没排上"还是"LLM 没读对"。

如果一次评测只评 LLM（不换 Embedding / Rerank），可以裁剪到 6 章；如果是完整选型，必须写满 10 章。

---

## 一、撰写前需要收集的资料清单

### 1.1 评测元信息
- **评测对象**：候选 LLM 列表、候选 Embedding 列表、候选 Rerank 列表（每类 2-4 个）
- **知识快照版本**：评测时锁定的 `kb_snapshot_version`（不能中途换）
- **评测集版本**：《21_RAG 模型评测集》的版本号、题量
- **执行方式**：离线脚本 / 线上灰度 / 盲评；LLM-as-Judge 用哪个模型、Judge Prompt 版本、温度
- **评测时间窗口**、执行环境（GPU / API、向量库版本）
- **评测人**、是否交叉盲评

### 1.2 候选模型清单
- **LLM**：模型名 + 版本、部署形态（API / 私有化）、上下文窗口、输入 / 输出价格、RPM/TPM、是否支持 JSON Mode / Function Calling
- **Embedding**：模型名 + 版本、维度、最大输入长度、支持语言、是否需要 query/passage 前缀指令、MTEB / C-MTEB 公开分
- **Rerank**：模型名 + 版本、Cross-Encoder 还是 LLM-based、最大输入长度、MS MARCO NDCG@10 公开分、延迟

### 1.3 评测数据集
- **业务 QA 集**：题量、5 类意图分布、golden chunk 标注完成率
- **公共基准对照**：C-MTEB Retrieval 子集、BEIR 抽 3-5 个、MS MARCO Rerank 子集
- **Train / Dev / Test 切分**：Dev 用来调 Prompt 和 Top-k 参数，Test 只跑一次

### 1.4 数据侧结果
- **检索层指标**：Context Precision@5、Context Recall@5、NDCG@10、MRR@10、Hit Rate@5
- **生成层指标**：Faithfulness、Answer Relevancy、Factual Correctness、Negative Rejection 正确率
- **系统层指标**：TTFT P50/P95、端到端 P95、单次请求 token 成本、月活预估成本
- **Bad case 素材**：每类典型 bad case 的 query / golden chunk / 召回结果 / LLM 输出 / 错误归因

### 1.5 决策侧
- **硬门槛**：Faithfulness ≥ ?%、Context Recall@5 ≥ ?%、P95 延迟 ≤ ?s、单次成本 ≤ ?￥
- **推荐方向**：效果优先 / 成本优先 / 延迟优先
- **不推荐清单**：哪些候选因为什么原因直接淘汰

---

## 二、资料来源渠道

| 资料类型 | 推荐渠道 |
|---|---|
| 候选模型公开分 | MTEB / C-MTEB / BEIR / MS MARCO 榜单、Hugging Face Model Hub、OpenRouter 价格横评 |
| 业务 QA 与 golden chunk | 《21_RAG 模型评测集》、业务知识 Owner 标注 |
| 检索层指标 | 离线评测脚本、向量库日志、RAGAS / TruLens 自动打分 |
| 生成层指标 | LLM-as-Judge（GPT-4o / Claude 3.5 以上）、人工抽样 100 条校验 |
| 系统层指标 | 压测报告、API 网关日志、成本账单 |
| Bad case 归因 | 工作流 trace、retrieval_id 全链路日志、人工复盘 |
| 上线建议 | PM / 算法 / 业务 Owner 三方对齐会 |

> **铁律**：RAG 选型报告的每个结论都必须能指着某道题、某个 golden chunk、某次 retrieval_id 讲——不能只给平均分。Embedding 好不好，要看 Recall@5；Rerank 好不好，要看"加了 Rerank 之后 Recall 提升了多少"；LLM 好不好，要看 Faithfulness，而不是"主观感觉答得更顺"。

---

## 三、完整框架结构（到三级标题）

> 严格 10 章：评测概述 → LLM 选型 → Embedding 选型 → Rerank 选型 → 8 组组合矩阵 → RAG Triad 指标 → 成本对比 → 推荐方案 → 风险与建议 → 附录。
> **第 2、3、4、5 章是 RAG 选型区别于单模型选型的核心**，缺一不可。

### 3.1 评测概述（元信息 + 核心结论）

- **这一节写什么**：一页纸讲清"评什么、用什么评、怎么判、一句话推荐哪一组"。
- **颗粒度要求**：元信息 ≥ 8 个字段 + 一段 3 句话的核心结论。
- **填写要点**：
  - **元信息必须含 `kb_snapshot_version` 和评测集版本**——没有这两个字段，报告的所有数字都不可复现。
  - **核心结论 3 句话**：① 推荐哪一套组合（LLM / Embedding / Rerank 各是哪个）；② 为什么（Faithfulness / Recall / 成本 / 延迟 4 维里哪 2 维领先）；③ 什么时候触发复评。
  - **不要在这一节堆数据**——数据放后面章节。
- **示例片段**：
  ```
  | 字段 | 值 |
  |---|---|
  | 评测对象 | LLM 候选 {3} 个、Embedding 候选 {3} 个、Rerank 候选 {2} 个 |
  | 知识快照 | kb_snapshot = {snap_2026Q4_r3} |
  | 评测集 | {业务 RAG 评测集 v1.3}，正式计分 {120} 题（含负样本 {25}） |
  | 执行方式 | 离线脚本；LLM-as-Judge = {模型 X}，温度 {0}，跑 {3} 次取均值 |
  | 判定口径 | 检索层按 golden chunk 机读；生成层按关键事实 + LLM Judge |
  | 评测日期 | {YYYY-MM-DD} |
  | 评测人 | {PM} + {算法} + {业务 Owner} |

  **核心结论**：
  1. 推荐组合 = {LLM-A} + {Embedding-B} + {Rerank-C}，端到端 Faithfulness = {0.92}、Context Recall@5 = {0.89}、P95 延迟 = {1.8}s、单次成本 = {￥0.012}。
  2. {LLM-B} 效果略好但成本高 {60%}、{Embedding-A} 在中文上 OOD 掉点 {18%}、{Rerank-D} 延迟 {3.2}s 超出预算。
  3. 上线后每小时抽 {100} 条跑 RAGAS，Faithfulness 跌破 {0.85} 或 Negative Rejection < {99%} 时触发复评。
  ```

### 3.2 LLM 选型对比（生成层）

- **这一节写什么**：把候选 LLM 在指令遵循、事实性、拒答、成本、延迟上的得分摆成一张表。
- **颗粒度要求**：对比表行是 LLM，列至少 7 列——模型 / Faithfulness / Answer Relevancy / Factual Correctness / Negative Rejection Acc / TTFT P50 / 单次成本；每格填具体数字，不要填"较好 / 一般"。
- **填写要点**：
  - **必须同时给"全量题"和"分意图"两层表**——只给平均分看不出 LLM 在多跳题上的短板。
  - **Negative Rejection 是 P0**——这一列 < 100% 的 LLM 直接淘汰，不用比其他维度。
  - **标注是否达硬门槛**：每列加 √/×。
- **示例片段**：

  **主表（全量 120 题）**：
  | LLM | Faithfulness | Answer Relevancy | Factual Correctness | Negative Rej Acc | TTFT P50 | 单次成本 | 硬门槛 |
  |---|---|---|---|---|---|---|---|
  | {LLM-A} | 0.92 | 0.88 | 0.89 | 100% (25/25) | 0.8s | ￥0.010 | √ 全过 |
  | {LLM-B} | 0.94 | 0.90 | 0.91 | 100% (25/25) | 1.2s | ￥0.016 | √ 全过 |
  | {LLM-C} | 0.85 | 0.82 | 0.83 | 96% (24/25) | 0.6s | ￥0.006 | × Negative Rej 不过 |

  **分意图表（关键行）**：
  | LLM | 基础事实 (48题) | 结构化数据 (15题) | 多跳推理 (22题) | SOP (18题) | 边界拒答 (17题) |
  |---|---|---|---|---|---|
  | {LLM-A} | 0.95 | 0.92 | 0.85 | 0.90 | 100% |
  | {LLM-B} | 0.96 | 0.94 | 0.89 | 0.92 | 100% |

  > **结论**：{LLM-B} 在多跳推理上比 {LLM-A} 高 {4} 个点，但成本高 {60%}、延迟高 {50%}；如果多跳题占比 < 20% 且成本敏感，选 {LLM-A}；如果多跳题是核心场景，选 {LLM-B}。{LLM-C} 因 Negative Rejection 漏 1 题（R51 硬编了一个相似但错误的答案），直接淘汰。

### 3.3 Embedding 选型对比（检索层）

- **这一节写什么**：把候选 Embedding 在召回质量、双语适配、编码延迟、成本上的得分摆成一张表。
- **颗粒度要求**：对比表行是 Embedding，列至少 7 列——模型 / 业务集 Recall@5 / 业务集 Precision@5 / C-MTEB Retrieval / 双语适配 / 编码延迟 / 成本；公共基准分数必须给出处。
- **填写要点**：
  - **业务集 Recall@5 是主指标**——公共分再高，在自己业务上 OOD 掉点就是废。
  - **双语适配**：如果产品有英文题，必须单独测英文 Recall@5；不能只看中文分。
  - **编码延迟**：Embedding 是每次检索都要跑的一跳，延迟直接影响 P95。
- **示例片段**：

  | Embedding | 业务集 Recall@5 | 业务集 Precision@5 | C-MTEB Retrieval | 英文 Recall@5 | 编码延迟 (1000 chunks) | 单次成本 |
  |---|---|---|---|---|---|---|
  | {Emb-A} | 0.89 | 0.72 | 62.3 | 0.85 | 120ms | ￥0.0002 |
  | {Emb-B} | 0.92 | 0.75 | 68.7 | 0.78 | 180ms | ￥0.0003 |
  | {Emb-C} | 0.74 | 0.61 | 55.2 | 0.88 | 80ms | ￥0.0001 |

  > **结论**：{Emb-B} 在中文业务集上 Recall@5 最高（{0.92}），但英文 Recall@5 掉到 {0.78}（OOD 掉点 {18%}）；如果产品中文为主，选 {Emb-B}；如果英文题占比 ≥ 10%，{Emb-A} 双语更稳。{Emb-C} 中文 Recall 过低直接淘汰。

### 3.4 Rerank 选型对比（重排层）

- **这一节写什么**：把候选 Rerank 在重排后首条命中率、比 Embedding 提升幅度、延迟上的得分摆成一张表。
- **颗粒度要求**：对比表行是 Rerank，列至少 6 列——模型 / NDCG@10 业务集 / 加 Rerank 后 Recall@5 提升 / Faithfulness 提升 / 延迟 / 额外成本。
- **填写要点**：
  - **主指标是"加 Rerank 后提升了多少"**——不是 Rerank 自己的绝对分，而是"加了 Rerank 比没加好多少"。
  - **如果某 Rerank 加了之后 Recall@5 提升 < 2%，Faithfulness 提升 < 1%，说明 Rerank 这一跳可以省掉**——直接用 Embedding Top-k 就行，节省延迟和成本。
  - **延迟是硬约束**：Rerank 是同步链路里最贵的一跳，P95 超预算直接淘汰。
- **示例片段**：

  | Rerank | NDCG@10 业务集 | 加 Rerank 后 Recall@5 提升 | Faithfulness 提升 | 延迟 (Top-12 chunks) | 额外成本 |
  |---|---|---|---|---|---|
  | {Rer-C} | 0.78 | +3.5% (0.89→0.925) | +2.1% | 200ms | ￥0.001 |
  | {Rer-D} | 0.75 | +1.8% (0.89→0.906) | +0.8% | 600ms | ￥0.002 |

  > **结论**：{Rer-C} 加了之后 Recall@5 提升 {3.5%}、Faithfulness 提升 {2.1%}、延迟 {200ms}，性价比明确；{Rer-D} 提升 < 2% 但延迟 {600ms}，ROI 不划算，淘汰。
  > **敏感性结论**：如果业务集本身 Recall@5 已经 ≥ 0.90 且多跳题占比低，Rerank 这一跳可以省掉，直接用 Embedding Top-5。

### 3.5 8 组组合矩阵（LLM × Embedding × Rerank 端到端压测）

- **这一节写什么**：前面 3 章是单维度对比，但真实线上是组合——必须把 Top-2 LLM × Top-2 Embedding × Top-2 Rerank = 8 组全跑一遍端到端。
- **颗粒度要求**：8 行组合表 + 一张 Pareto 前沿散点图描述（X=单次成本，Y=Faithfulness）。
- **填写要点**：
  - **不能只跑"我最喜欢的那 1 组"**——单维度最优 ≠ 组合最优。
  - **Pareto 前沿**：把 8 组散点画出来，落在前沿上的才是可选项；被其他组同时在效果和成本上压制的，直接淘汰。
  - **每组必须给 4 个数**：Faithfulness / Recall@5 / P95 延迟 / 单次成本。
- **示例片段**：

  | 组合 | LLM | Embedding | Rerank | Faithfulness | Recall@5 | P95 延迟 | 单次成本 |
  |---|---|---|---|---|---|---|---|
  | G1 | {LLM-A} | {Emb-A} | {Rer-C} | 0.92 | 0.89 | 1.8s | ￥0.012 |
  | G2 | {LLM-A} | {Emb-A} | 无 Rerank | 0.90 | 0.86 | 1.6s | ￥0.011 |
  | G3 | {LLM-A} | {Emb-B} | {Rer-C} | 0.93 | 0.92 | 1.9s | ￥0.012 |
  | G4 | {LLM-A} | {Emb-B} | 无 Rerank | 0.91 | 0.89 | 1.7s | ￥0.011 |
  | G5 | {LLM-B} | {Emb-A} | {Rer-C} | 0.94 | 0.89 | 2.4s | ￥0.018 |
  | G6 | {LLM-B} | {Emb-B} | {Rer-C} | 0.95 | 0.92 | 2.5s | ￥0.018 |
  | G7 | {LLM-B} | {Emb-A} | 无 Rerank | 0.92 | 0.86 | 2.2s | ￥0.017 |
  | G8 | {LLM-B} | {Emb-B} | 无 Rerank | 0.93 | 0.89 | 2.3s | ￥0.017 |

  > **Pareto 前沿分析**：
  > - **成本优先前沿**：G2（￥0.011 / 0.90）和 G4（￥0.011 / 0.91）——G4 同时压 G2，G2 淘汰。
  > - **效果优先前沿**：G6（0.95 / ￥0.018）——被 G3（0.93 / ￥0.012）在成本上大幅压制、只低 2 个点，属于"为了 2 个点多花 50% 成本"。
  > - **推荐落在 G3**（LLM-A + Emb-B + Rer-C）：Faithfulness 0.93、Recall 0.92、成本 ￥0.012、P95 1.9s，全部达硬门槛，且在 Pareto 前沿上。

### 3.6 RAG Triad 指标与 Bad case 归因

- **这一节写什么**：RAG Triad = Faithfulness（答案忠实）/ Context Precision（context 精准）/ Context Recall（context 召回）。本节给三维度得分 + bad case 归因饼图 + 每类典型样本。
- **颗粒度要求**：三维度得分表 + 错误归因饼图（4 类：Embedding 没召回 / Rerank 没排上 / LLM 没读对 / Prompt 没约束）+ 每类抽 2-3 条典型 bad case。
- **填写要点**：
  - **RAG Triad 三维度必须分开看**——只看 Faithfulness 会漏掉"召回是对的但 LLM 没读对"这种 case。
  - **bad case 归因必须落到 4 类之一**，不能笼统写"答得不好"。
  - **每类典型 bad case 给四要素**：query / golden chunk / 召回结果 / LLM 输出 / 错误类型。
- **示例片段**：

  **RAG Triad 三维度得分（G3 组合）**：
  | 维度 | 得分 | 硬门槛 | 是否达标 |
  |---|---|---|---|
  | Faithfulness（答案忠实） | 0.93 | ≥ 0.90 | √ |
  | Context Precision@5（context 精准） | 0.75 | ≥ 0.70 | √ |
  | Context Recall@5（context 召回） | 0.92 | ≥ 0.85 | √ |
  | Negative Rejection 正确率 | 100% (25/25) | 100% | √ |

  **错误归因饼图（120 题中未达标的 {N} 题）**：
  | 错误类型 | 题数 | 占比 | 根因 |
  |---|---|---|---|
  | Embedding 没召回 | {5} | {45%} | {Emb-B} 在{专业术语}上 OOD；Top-k 从 12 调到 20 可缓解 |
  | Rerank 没排上 | {3} | {27%} | {Rer-C} 对{长 chunk}排序不稳；chunk_size 从 1024 降到 512 可缓解 |
  | LLM 没读对 | {2} | {18%} | {LLM-A} 在{数字枚举}上漏项；Prompt 加"必须逐条列出"可缓解 |
  | Prompt 没约束 | {1} | {10%} | Prompt 未要求"引用来源"；导致用户无法溯源 |

  **典型 bad case**：
  > - **R25（多跳题）— Embedding 没召回**：query = "我上周刚充了但没用，是小孩充的能退吗"；golden chunk = `refund_policy` chunk_12 / chunk_15 + `minors_policy` chunk_22；Top-5 只召回 chunk_12，chunk_15 和 chunk_22 掉到 Top-8。**归因**：Embedding 对"未成年人"和"未使用"两个语义的相似度打分偏低。**修复建议**：Top-k 从 12 调到 20，Rerank 后保留 5 条。
  > - **R12（结构化题）— LLM 没读对**：query = "基础版和专业版差在哪"；golden chunk 召回正确，但 LLM 输出漏了"专业版含 C、D 两项"。**归因**：LLM-A 对枚举类输出容易漏项。**修复建议**：Prompt 加"必须逐条枚举所有差异，不得省略"。
  > - **R51（负样本）— LLM 硬编**：query 是库中完全没有的问题，LLM 编造了一个相似但错误的答案。**归因**：Prompt 未强制"context 中没有就说不知道"。**修复建议**：Prompt 加负样本硬约束。

### 3.7 成本对比（单次调用 + 月活预估）

- **这一节写什么**：把 8 组组合的成本摆清楚——单次调用成本 + 按月活预估的总成本 + 延迟预算分配。
- **颗粒度要求**：单次成本拆解表（Embedding + Rerank + LLM 输入 + LLM 输出 + Judge）+ 月活预估成本表 + 延迟预算分配表。
- **填写要点**：
  - **单次成本必须拆到每一跳**——不能只给总数。
  - **月活预估成本**：按业务 QPS 或月活 query 量估算，让决策者知道"上线后每月要花多少钱"。
  - **延迟预算分配**：Embedding 编码 + 向量检索 + Rerank + LLM 生成，每一跳占多少 ms，P95 超了要知道从哪跳省。
- **示例片段**：

  **G3 单次成本拆解**：
  | 环节 | 用量 | 单价 | 成本 |
  |---|---|---|---|
  | Embedding 编码（query） | {1} 次 × {512} tokens | ￥{0.0000002}/token | ￥0.0001 |
  | 向量检索 | {12} chunks | — | ≈ 0 |
  | Rerank（Top-12） | {12} chunks × {512} tokens | ￥{0.0000002}/token | ￥0.0015 |
  | LLM 输入（Prompt + context） | {2000} tokens | ￥{0.000004}/token | ￥0.008 |
  | LLM 输出 | {250} tokens | ￥{0.000012}/token | ￥0.003 |
  | Judge 打分（离线） | 摊销 | — | ￥0.001 |
  | **合计** | | | **￥0.0136** |

  **月活预估成本**（按 {10} 万次 query / 月）：
  - G3：￥{1360} / 月
  - G6：￥{1800} / 月（多花 {32%} 换 2 个点 Faithfulness）

  **延迟预算分配（G3，P95 = 1.9s）**：
  | 环节 | P50 | P95 | 占比 |
  |---|---|---|---|
  | Embedding 编码 | 60ms | 120ms | 6% |
  | 向量检索（Top-12） | 30ms | 80ms | 4% |
  | Rerank | 150ms | 200ms | 11% |
  | LLM 生成（TTFT） | 600ms | 900ms | 47% |
  | LLM 生成（总时长） | 1200ms | 1500ms | 79%（含 TTFT） |

  > **结论**：LLM 生成是延迟和成本的主要去向（占 {79%}）；如果要压 P95，优先换更快的 LLM，而不是调 Embedding / Rerank。

### 3.8 推荐方案（明确推荐哪一组 + 为什么其他组落选）

- **这一节写什么**：基于前面所有数据，明确给出推荐组合 + 备选组合 + 不推荐清单。
- **颗粒度要求**：推荐组合（首选）+ 备选组合（效果优先 / 成本优先）+ 不推荐清单（每组写清落选原因）。
- **填写要点**：
  - **推荐必须是 1 组**——不能给"G3 和 G4 都可以"。
  - **其他组落选原因要具体**——不是"效果不好"，而是"Negative Rejection 漏 1 题直接淘汰"、"P95 超预算 600ms 淘汰"、"多花 50% 成本只换 2 个点 Faithfulness"。
  - **必须显式说明"为什么不选效果最好的那组"**——这是决策者最关心的问题。
- **示例片段**：

  **推荐组合（首选）**：
  > **{G3 = LLM-A + Emb-B + Rer-C}**
  > - Faithfulness = {0.93}、Context Recall@5 = {0.92}、P95 = {1.9}s、单次成本 = {￥0.0136}、Negative Rejection = {100%}。
  > - 全部达硬门槛，落在 Pareto 前沿上。
  > - 适合：中文为主、成本敏感、多跳题占比 ~18% 的业务场景。

  **备选组合**：
  > - **效果优先版 = G6（LLM-B + Emb-B + Rer-C）**：Faithfulness = {0.95}，但成本 {+32%}、P95 {+0.6s}。适合多跳题占比 ≥ 30%、且业务对事实错误零容忍的场景。
  > - **成本优先版 = G4（LLM-A + Emb-B + 无 Rerank）**：单次成本 {￥0.0121}（省 {11%}）、P95 {1.7s}，但 Recall@5 掉到 {0.89}、Faithfulness 掉到 {0.91}。适合内部工具、对准确率要求 < 0.93 的场景。

  **不推荐清单及原因**：
  > - **{LLM-C}**：Negative Rejection 漏 1 题（R51 硬编），P0 红线不过，直接淘汰。
  > - **{Emb-A}**：中文 Recall@5 = {0.89} 比 Emb-B 低 {3} 个点，且成本没有明显优势，Pareto 被 G3 压制。
  > - **{Rer-D}**：加了之后 Recall 提升 {1.8%}、Faithfulness 提升 {0.8%}，但延迟 {600ms}、成本 {+￥0.002}，ROI 不划算。
  > - **{G7 / G8}**：LLM-B 不配 Rerank 时 Faithfulness 优势不足以覆盖成本劣势，被 G3 / G6 分别压制。

### 3.9 风险与建议（上线前还需验证什么 + 监控什么 + 什么时候触发重评）

- **这一节写什么**：推荐组合不是终点——上线前还要补什么验证、上线后监控哪些指标、什么情况下必须触发重评。
- **颗粒度要求**：上线前验证清单 + 线上监控指标清单 + 重评触发条件 3 段。
- **填写要点**：
  - **上线前验证**：真实接口联调、权限越权测试、Prompt 注入测试、极端 query 压测。
  - **线上监控**：和离线指标一一对应——每小时抽 {100} 条跑 RAGAS / TruLens。
  - **重评触发条件**：Faithfulness 跌破阈值、Negative Rejection < 99%、Embedding / Rerank 新版本发布、业务知识库大更新。
- **示例片段**：

  **上线前必须验证**：
  1. 真实接口联调：当前评测用的是{模拟快照}，上线前要把{真实知识快照}替换进来并全量重跑。
  2. 权限越权测试：跨租户 / 跨业务线题必须 100% 返回 `permission_denied`。
  3. Prompt 注入测试：恶意文档夹带注入指令，不得被执行。
  4. 极端 query 压测：长 query（> 1000 tokens）、多语言混合、ASR 错词。

  **线上监控指标（每小时抽 100 条跑 RAGAS）**：
  | 指标 | 报警阈值 | 处置 |
  |---|---|---|
  | Faithfulness | < {0.85} | 触发告警，PM 介入 |
  | Context Recall@5 | < {0.80} | 检查是否新入库文档切片异常 |
  | Negative Rejection 正确率 | < {99%} | P0，立即回滚 |
  | 端到端 P95 | > {3}s | 检查 LLM 限流 / 向量库负载 |
  | 单次成本 | > {￥0.020} | 检查是否 prompt 膨胀 |

  **重评触发条件**：
  - Faithfulness 连续 3 小时 < {0.85}；
  - 新模型发布（LLM / Embedding / Rerank 任一）；
  - 知识快照大更新（新增 > 10% 文档）；
  - 线上 bad case 周新增 > {20} 条且集中在同一意图分类。

### 3.10 附录

- **这一节写什么**：让读者能复现这次评测——评测集版本、golden chunk 清单、bad case 原始素材、Judge Prompt、实验环境。
- **颗粒度要求**：附录至少 7 项——评测集版本 / golden chunk 清单 / bad case 原始素材链接 / Judge Prompt 全文 / System+User Prompt 全文 / 实验环境快照 / 参考论文与文档。
- **填写要点**：
  - **Judge Prompt 必须全文贴**——没有 Judge Prompt，生成层指标无法复现。
  - **bad case 原始素材**：链接到评测平台的 retrieval_id 详情页 / 录屏 / 原始 JSON，不能只写"见 bad case 表"。
  - **实验环境快照**：Docker image / requirements.txt / 随机种子 / 评测脚本 commit hash。
- **示例片段**：
  > - **附录 A**：评测集版本 v{1.3}，{120} 题（golden chunk 标注完成率 {100%}）。
  > - **附录 B**：golden chunk 完整清单（{链接到多维表格}）。
  > - **附录 C**：bad case 原始素材（{链接到评测平台 retrieval_id 列表}）。
  > - **附录 D**：LLM-as-Judge Prompt 全文（{附后}）；Judge 模型 = {GPT-4o}，温度 = {0}，跑 {3} 次取均值；人工抽样 {100} 条校验 Judge 与人工一致性 = {0.89}。
  > - **附录 E**：System Prompt / User Prompt 全文（{附后}）。
  > - **附录 F**：实验环境快照——Python {3.11}、向量库 {Milvus 2.4}、Embedding 推理框架 {FastEmbed}、随机种子 {42}、评测脚本 commit = {abc123}。
  > - **附录 G**：参考论文与文档——RAGAS / TruLens / ARES / CRAG / RGB / MTEB / C-MTEB / BEIR / MS MARCO。
  > - **本结论仅对应知识快照 {snap_2026Q4_r3} 与评测集 v{1.3}；快照或评测集升级后必须全量重跑。**

---

## 四、不同产品类别的差异化要点

### 4.1 B 类（RAG 知识库 / 问答产品，主用）
- **必须写全 10 章**：尤其是 3.5（8 组组合矩阵）和 3.6（RAG Triad + bad case 归因）——这两章是 RAG 选型区别于单模型选型的核心。
- **常见坑**：只评 LLM 不评 Embedding / Rerank；只给平均分不给分意图表；bad case 归因只写"答得不好"不落到 4 类之一。

### 4.2 A 类（对话式 Agent，辅用）
- **写法**：如果 Agent 里挂了 RAG、且 RAG 选型直接影响 Agent 效果，引用本报告结论即可；额外加一章"Agent 侧表现"（多轮状态、工具调用、拒答话术）。
- **重点**：RAG 选型要和 Agent 的 Prompt 一起评——单独评 RAG 端到端 0.93，挂到 Agent 里可能掉到 0.85。

### 4.3 C 类（AIGC 多模态生成，辅用）
- **常见用法**：生图 Agent 用 RAG 检索素材库。
- **重点**：Embedding 要选多模态版本（图文对齐）；Rerank 可能用不上（素材库规模小）；Faithfulness 指标换成"素材相关度"。

### 4.4 D 类（评测平台 / 内部工具，主用）
- **评测报告本身是产品的一类标准模板**——重点是模板化、可复用、和 RAGAS / TruLens 自动打分对齐。
- **必须提供**：一键跑 8 组组合的脚本、自动生成对比表的能力、Pareto 前沿可视化。

### 4.5 E 类（B 端 SaaS AI 化，辅用）
- **重点**：多租户隔离是 P0——Negative Rejection 里的"跨租户越权"必须 100%；每个租户要单独抽 20 题跑一遍。
- **成本**：B 端客户按月付费，单次成本必须拆清楚才能定价。

---

## 五、完成度 Checklist

- [ ] 评测概述页写清元信息 ≥ 8 字段（含 `kb_snapshot_version` 和评测集版本）
- [ ] 核心结论 3 句话：推荐哪一组 / 为什么 / 什么时候触发重评
- [ ] LLM 对比表至少 7 列（Faithfulness / Answer Relevancy / Factual Correctness / Negative Rej / TTFT / 成本 / 硬门槛），并给分意图表
- [ ] Negative Rejection < 100% 的 LLM 已显式淘汰
- [ ] Embedding 对比表含业务集 Recall@5 / C-MTEB / 双语适配 / 编码延迟；OOD 掉点已显式标注
- [ ] Rerank 对比表主指标是"加 Rerank 后提升多少"，不是绝对分；提升 < 2% 的 Rerank 已建议省掉
- [ ] 8 组组合矩阵（Top-2 LLM × Top-2 Embedding × Top-2 Rerank）已全量跑出，Pareto 前沿已画
- [ ] RAG Triad 三维度（Faithfulness / Context Precision / Context Recall）分开打分，未只给端到端平均分
- [ ] Bad case 归因落到 4 类之一（Embedding 没召回 / Rerank 没排上 / LLM 没读对 / Prompt 没约束），每类有典型样本
- [ ] 成本拆解到每一跳（Embedding + Rerank + LLM 输入 + 输出 + Judge），并给月活预估
- [ ] 延迟预算分配表已给，P95 超支从哪跳省已说明
- [ ] 推荐组合唯一（不能给"都可以"），其他组落选原因具体
- [ ] 上线前验证清单（真实接口 / 越权 / 注入 / 极端 query）已列
- [ ] 线上监控指标和离线指标一一对应，报警阈值已给
- [ ] 附录含评测集版本 / golden chunk 清单 / bad case 素材 / Judge Prompt / 实验环境，可复现
- [ ] 显式声明"本结论仅对应当前快照和评测集版本，升级后必须全量重跑"

---

## 六、AI 生成 Prompt 模板

```
你是一名资深 AI 产品经理，负责为 {RAG 产品名} 做 LLM / Embedding / Rerank 三类模型的选型评测，并撰写《RAG 模型选型评测报告》。
请根据以下项目信息，生成一份 10 章结构的 RAG 选型评测报告。

【背景输入】
- RAG 产品名：{xxx RAG 问答服务}
- 知识快照版本：{kb_snapshot_version}（{入库日期}）
- 评测集：{业务 RAG 评测集 v{1.3}}，{120} 题（含负样本 {25} 题）
- 候选 LLM：{3} 个（{模型 A / B / C}），各自部署形态、上下文、价格、RPM
- 候选 Embedding：{3} 个（{Emb-A / B / C}），维度、语言、MTEB 公开分、价格
- 候选 Rerank：{2} 个（{Rer-C / D}），类型、最大长度、MS MARCO 分、价格
- 业务 QPS / 月活预估：{N 次 query / 月}
- 延迟预算 P95：{≤ ?s}
- 单次成本预算：{≤ ￥?}
- 硬门槛：Faithfulness ≥ {0.90} / Context Recall@5 ≥ {0.85} / Negative Rejection = {100%}
- 评测结果数据（已整理）：
  · LLM 对比表（Faithfulness / Relevancy / Factual / Negative Rej / TTFT / 成本）
  · Embedding 对比表（业务集 Recall@5 / C-MTEB / 双语 / 延迟 / 成本）
  · Rerank 对比表（加 Rerank 后 Recall 提升 / Faithfulness 提升 / 延迟 / 成本）
  · 8 组组合矩阵（G1-G8 的 Faithfulness / Recall@5 / P95 / 成本）
  · Bad case 清单（每道含 query / golden chunk / 召回 / 输出 / 归因 4 类之一）
- 推荐方向：{效果优先 / 成本优先 / 平衡}

【撰写要求】
1. 严格按照《RAG 模型选型评测报告 SKILL》10 章结构输出：评测概述 → LLM 选型 → Embedding 选型 → Rerank 选型 → 8 组组合矩阵 → RAG Triad + bad case 归因 → 成本对比 → 推荐方案 → 风险与建议 → 附录。
2. 每个三级标题下包含：这一节写什么 / 颗粒度要求 / 填写要点 / 示例片段。
3. LLM 对比表必须含全量表 + 分意图表两层；Negative Rejection < 100% 的 LLM 直接淘汰。
4. Embedding 对比主指标是业务集 Recall@5，公共分只作参考；OOD 掉点要显式标注。
5. Rerank 对比主指标是"加 Rerank 后提升多少"，不是绝对分；提升 < 2% 的建议省掉这一跳。
6. 8 组组合矩阵必须全量跑出，画 Pareto 前沿，落在前沿上的才是可选项。
7. RAG Triad 三维度（Faithfulness / Context Precision / Context Recall）分开打分；bad case 归因落到 4 类之一（Embedding 没召回 / Rerank 没排上 / LLM 没读对 / Prompt 没约束）。
8. 成本拆解到每一跳，给月活预估；延迟预算分配到每一跳。
9. 推荐组合唯一；其他组落选原因具体；显式说明"为什么不选效果最好的那组"。
10. 附录含评测集版本 / golden chunk 清单 / bad case 素材 / Judge Prompt 全文 / 实验环境快照，可复现。
11. 去具体化：不要使用任何真实公司名、产品名、行业名、模型名、URL；全部用 {占位符} 替代。
12. 输出为 Markdown，章节编号与 SKILL 框架一致。
```
