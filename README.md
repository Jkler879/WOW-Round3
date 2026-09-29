# WOW-离线数据飞轮（三轮）
针对ReAct Agent多轮对话系统的离线数据飞轮模块（共三轮）

第一轮：Round 1 诊断轮（获取系统基于ragas原生框架指标基准baseline - 提前冻结的200条评估集 - judgemodel：qwen-plus与生产系统分离）

第二轮：Round 2 验证轮（观测Round1后做的系统优化、知识库优化等带来的指标优化 - 提前冻结的200条评估集-统一judgemodel：qwen-plus）

第三轮：Round 3 压力轮（对标生产上线指标 + 系统能力天花板测试）

Round 1, 2 轮已结束，具体优化步骤、指标提升在每轮完整报告中。

Round 3 目标：5 项 RAGAS 原生指标对标生产上线标准 + 系统能力天花板测试

### 完整飞轮报告：
  [Round 2 RAGAS Diagnostic
  Report](https://htmlpreview.github.io/?https://github.com/Jkler879/WOW-Round3/blob/main/round2_evaluation_report.html)
  
  [Round 1 RAGAS Baseline
  Report](https://htmlpreview.github.io/?https://github.com/Jkler879/WOW-Round3/blob/main/round1_baseline_report.html)

### 各轮次指标对比：

  | 指标 | Round 1 基准 | Round 2 | Round3 | 生产上线目标 |
  |------|:-----------:|:-------:|:-------:|:----------:|                                   
  | Faithfulness (F) | 0.490 | **0.610** ↑ |  | ≥ 0.70 |
  | Answer Relevancy(AR) | 0.633 | **0.726** ↑|| ≥0.70 ✅|
  | Context Precision (CP) | 0.947 ⚠️ | **0.950** ↑⚠️ |  | ≥ 0.85 |
  | Context Recall (CR) | 0.903 ⚠️ | **0.937** ↑⚠️ |  | ≥ 0.85 |
  | Answer Correctness (AC) | 0.590 | **0.594** ↑| | ≥0.65 |
> ⚠️ CP / CR 当前虚高，根因为评估集由 Claude Sonnet 基于知识库 top 200 合成，模拟数据中的问题关键词汇与原文高度重叠且缺少口语化表达，导致检索难度被严重低估。Round 3 前已补充 60 条词汇多样化的模拟用户问题，预期 CP / CR将回落至真实水平（详情阅读下文：6、CP (0.950) CR (0.937) 值虚高）

### Round 2 诊断及优化（7项系统优化）：

#### 1. Reasoning 类型问题幻觉（优化前：FaithFulness 0.61）

  **规模：** 16 条 badcase，占 badcase 总量 43%。
  
  **根因：** 模型已检索到正确文档（CR = 1.0），但 ReAct 推理链在生成答案前混入了参数记忆，当前系统 Rule 7仅约束输出层，无法拦截推理过程中的参数记忆渗入（post-rationalization）。

  **优化方案**：
  | Opt | 核心操作 | 生效时机 |
  |------|---------|---------|
  | System Prompt 添加 Rule8：检索层证据提取 | 推理前强制列出原文证据片段（注明来源），标明文档空白；后续推理只能基于已列片段 |检索返回后的 `agent_node` 调用 |
  | System Prompt 添加 Rule9：模型输出前自检 | 输出答案前逐项核查推断是否有原文依据；无依据内容删除或标注"文档无直接记载" | 同一`agent_node`，紧接 Opt1 |

  > Rule 8 约束推理过程(强制证据提取)，Rule 9约束模型输出（强制输出自检），配合系统原有Rule 7（证据不足时明确表态），三条规则覆盖推断类问题的完整生命周期，共同拦截推理过程中的参数记忆渗入，提升系统核心幻觉指标 FF。

  **参考文献：**
  - *Correctness is not Faithfulness in RAG Attributions* —https://arxiv.org/abs/2412.18004
  - Evidence-First 是切断 post-rationalization 的核心手段。RAG 系统中高达 57%的引用为后验合理化——模型先形成答案再反向贴引用，Citatio是事后标签而非推理起点。Evidence Extraction通过强制"先提取、再推理"颠倒这一顺序。
  
  - *Illocutionary Explanation Planning for Source-Faithful Explanations*（CoI，+34% source faithfulness）—
  https://arxiv.org/abs/2604.06211
  - Chain-of-Illocution（CoI）在解释生成任务上验证，Evidence-First 结构平均带来 +34% source faithfulness 提升（与RAGAS原生Faithfulness 为同类指标，非直接对应数值）。
  
  - *Dissociation of Faithful and Unfaithful Reasoning in LLMs* —https://arxiv.org/abs/2405.15092
  - LLM 的 CoT 链存在忠实与非忠实两种模式，二者在输出文本上几乎无法区分，只有在推理前强制提取证据才能切断参数记忆渗入的通道。


#### 2. Simple / Multi-hop 类型问题幻觉

  **规模：** 8 条 badcase，F < 0.3，非 reasoning 题型。
  
  **Simple 类根因：** 问题本身是单步事实查询（某人是哪里人、某事发生在哪年），检索文档里有明确答案，但模型直接用参数记忆里的"印象"覆盖了文档中的事实，没有经过推理链，是最简单粗暴的一种幻觉形式。系统原有 Rule 1 已经有接地约束，但约束强度不够——模型在训练数据中形成的强先验会静默地覆盖检索结果，而不是把冲突明显暴露出来。

  **Multi-hop 类根因：** 需要跨多个文档片段串联事实。模型检索到了 Chunk A（事实1）和 Chunk B（事实2），在两个 chunk之间搭桥连接的那一步，用参数记忆"补全"了中间环节，而不是严格从已检索的原文中推导。桥接步骤没有文档依据，但在输出文本上与完全有依据的连接难以区分。

  **优化方案**：
  | 改动 | 位置 | 内容 |
  |---|---|---|
  | 添加问题分类器 | `查询改写模块` | Qwen3-4B 新增输出字段 `QUESTION_TYPE`，对用户问题进行simple/reasoning/multi-hop三分类任务，透传至 `AgentState` |
  | 不同类型执行规则 | `agent.py` `QUESTION_TYPE_RULES` | simple：事实冲突以文档为准并标注原文；multi_hop：拆解子问题 →每跳单独检索 →每跳标注来源 |
  | 动态 `max_steps` | `main.py`（流式 + 非流式接口） | simple=3 / reasoning=5 / multi_hop=7，为 multi_hop多跳检索提供足够步数空间（原系统采用硬上限5步锁死ReAct Agent循环步数） |


#### 3. 答案焦点偏移

  **规模：** 13 条 badcase，Faithfulness ≥ 0.3（答案有文档支撑），Answer Correctness < 0.4（答案方向偏离）
  
  **根因：** 
  1、选择性接地（最常见）：文档里有 A、B、C 三条信息，问的是 A，模型认真地用 B 和 C 回答了，还附上了文档出处。F 尚可，但 AC低，因为根本没答到点上。

  2、理解偏差：问的是"X 的影响是什么"，模型解读成"X 是什么"，忠实地从文档里引用了对 X 的定义，答非所问但有据可查。

  3、答案不完整：文档包含答案的完整信息，但模型只提取了其中一部分关键点，导致 AC 被拉低（ground truth 要求更完整的覆盖）。

  **优化方案：**

  | 维度 | 内容 |
  |------|------|
  | **System Prompt** | Rule 10：问题焦点锚定（所有题型）①调用 `knowledge_retriever` 前：先用一句话明确本题的直接答案对象（即问题要求输出的具体内容，而非背景信息或相关事件）；②输出最终答案前：对照答案对象自检，若仅覆盖背景则重新聚焦 |
  | **实现位置** | `src/core/ReAct_Agent/tools/agent.py` —`SYSTEM_PROMPT` 行为准则 Rule 10 |
  | **与 Rule 8/9 的关系** | 互补而非重叠：Rule 8/9 约束"推断内容是否有文档依据"（防幻觉），Rule 10约束"答案方向是否对准了被问的那件事"（防偏答） |
  | **预期收益** | Answer Correctness ↑，尤其针对选择性接地和问题理解偏差两类失败模式|
  | **论文引用** | [Before Reasoning Fails](https://arxiv.org/abs/2608.02011)（2026）；[What Would Fix This RAGFailure?](https://arxiv.org/abs/2608.08944)（2026）|

#### 4. Reranker引发回归

  **规模：** 相比 Round 1 新增 9 条 badcase
  
  **根因：** 

  **优化方案：**
  1、原重排模型为 BGE-rerank-v2-m3 int8自量化版本，现升级为 FP32 原版模型
  2、重排模型从 CPU 移植至 GPU，由 TEI 框架负责部署，内置 FP 16 量化

#### 5. RAGAS 评估框架内置英文提示词造成 AR 指标虚低 

  **规模：** Round 2 共 183 条评估数据，37 条 AR 直接归零，整体 AR 均值受拖累 从 0.79 跌至 0.63，偏离真实水平。
  
  **根因：** 
  
  1、RAGAS 框架 AnswerRelevancy 内置英文 prompt，驱动 Judge LLM 从中文答案反推问题
  
  2、Judge LLM（qwen-plus）收到英文指令 + 中文答案，对 37 条数据生成了英文问题，与中文原始问题做 embedding 相似度时跨语言失配，余弦相似度归零。并非系统真实表现，是评估框架的语言错配导致的虚假低分

  **优化方案：**
  飞轮离线评估时，覆盖 RAGAS 内置的英文 question_generation 指令为中文版本，强制 Judge LLM 输出中文问题，消除跨语言失配，AR 恢复真实值。

#### 6、CP (0.950) CR (0.937) 值虚高

  
  **规模：** 183条全量评估集
  
  **根因：**   
  1、评估集是从知识库top 200 中抽取并冻结的数据，交给 LLM Claude Sonnet 5.0 生成虚拟用户提问和标准答案。
  
  2、Claude Sonnet 从 chunk 文本生成问题时，问题的词汇天然来自 chunk 本身。
    
    - BM25 看到问题关键词，精确匹配 chunk → CP 虚高
    
    - 向量检索 问题 embedding 与 chunk embedding 高度相近（语义本就来自同一段文本）→ CR 虚高
    
  3、真实用户问题的本质区别：词汇鸿沟（Vocabulary Gap）包含大量日常用语、同义词、缩写 / 句子长度短促稀疏 / 关键词重叠极低
  
  真实用户问法：
  
    - "滑雪比赛穿过森林那种是怎么玩的？"  ←没有 "cross-country" "groomed course" 等关键词
  
    - "泰勒斯威夫特唱什么类型的歌"       ←而不是 "Taylor Swift 的音乐风格是什么"
  
    - "自闭症小孩有什么表现"              ←而不是 "自闭症的主要症状有哪些"
  
  **优化方案：**
  | 方案 | 做法 | 规模 |
  |------|------|---------|
  | **改写现有问题** | 对现有合成问题做 paraphrase，指令约束"改用口语化表达" | 30条，满足中心极限定理，看清指标方向 |
  | **回避关键词** | LLM 生成时禁止使用 chunk 原文关键词汇 | 30条，总量 60 条，误差 ±5%，可信 |
  | **混合评估** | 与原评估集一起送入系统，CR/CP得到真实指标 | 183 原评估集 + 60 条模拟真实用户评估集 |
  

  **参考文献：**
  | 论文 | 结论 |
  |------|------|
  | [Beyond Benchmark Scores (2025)](https://arxiv.org/pdf/2609.14579) | 合成问题 CP虚高，真实用户查询词汇极度稀疏，两者分布存在根本性差异 |
  | [Can we Evaluate RAGs with Synthetic Data? (2025)](https://arxiv.org/pdf/2508.11758) |合成评估集对检索策略选择产生误导，基于合成数据的优化在真实流量上无实际收益 |
  | [DataMorgana / SIGIR LiveRAG (2025)](https://arxiv.org/html/2501.12789v1) | 生产级 RAG评估需要多样化问题类型，覆盖词汇鸿沟场景 |
  | [Synthetic Question Generation for RetrievalEvaluation](https://suzyahyah.github.io/nlp/2024/08/03/Retrieval-Evaluation.html) | LLM生成问题天然继承源文本词汇，导致检索评估偏乐观 |

  #### 7、F 改进统计显著性未认证，未做假设检验
  
  **规模：** Round2 全量 183 条评估数据

  **根因：**
  每轮 F 分数改进（如 Batch1→Batch2 +0.12）仅凭均值对比，未验证提升是系统优化效果还是批次间采样偏差导致的噪声。

  **优化方案：**

  | 检验方法 | 原理 | 判断标准 |
  |---------|---------|---------|
  | Mann-Whitney U 检验 | 核心检验，输出P值，判断两组分布差异是否显著 | p < 0.05 认定显著 |
  | Bootstrap 95% 置信区间 | 输出每组均值的 95% 置信区间 | 两组 CI 不重叠则显著 |
  | Cohen's d | 判断提升是否有实际意义（p 显著但 d 极小 = 统计显著但无实用价值） | d ≥0.5 为中等效应，d < 0.2 即便显著也无实际价值 |

  **已有结果（Batch1 vs Batch2）：**

  | 指标 | Batch1 | Batch2 | Δ| p 值 | 结论 |
  |------|--------|--------|---|------|------|
  | Faithfulness | 0.490 | 0.610 | +0.12 | 0.0001 | ✅ 显著提升 |
  | Answer Relevancy | 0.633 | 0.643 | +0.01 | 0.496 | —无显著差异 |
  | Context Precision | 0.947 | 0.950 | +0.002 | 0.883 | —无显著差异 |
  | Context Recall | 0.903 | 0.935 | +0.033 | 0.511 | —无显著差异 |

  > Faithfulness 统计显著（p=0.0001），非虚高指标。Round 3 跑完仍有提升空间（上文优化 4、优化 3支撑）。

  > Round 3 沿用，全部优化落地后重新执行检验，以 p < 0.05 + Cohen's d ≥0.5 双重标准认证。
  

  

  
  
