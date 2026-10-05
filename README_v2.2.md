# WOW-Data Flywheel 离线数据飞轮（三轮）

针对 ReAct Agent 多轮对话系统的离线数据飞轮模块，共三轮：

| 轮次 | 定位 | 说明 |
|------|------|------|
| Round 1 诊断轮 | 建立基准 | 基于 RAGAS 原生框架获取 baseline；提前冻结的 200 条评估集；Judge 模型 qwen-plus，与生产系统分离 |
| Round 2 验证轮 | 验证优化 | 观测 Round 1 后的 7 项系统与知识库优化效果；同一冻结评估集、同一 Judge 模型 |
| Round 3 压力轮 | 上线评估 | 观测 Round 2 后的 10 项系统与知识库优化效果；对标生产上线指标；新增 60 条贴近真实用户问法的合成数据，测出真实 CP / CR（已完成） |

完整报告：

[Round 3 Evaluation Report](https://htmlpreview.github.io/?https://github.com/Jkler879/WOW-Round3/blob/main/round3_evaluation_report.html)

[Round 2 Diagnostic Report](https://htmlpreview.github.io/?https://github.com/Jkler879/WOW-Round3/blob/main/round2_diagnostic_report.html) 

[Round 1 Baseline Report](https://htmlpreview.github.io/?https://github.com/Jkler879/WOW-Round3/blob/main/round1_baseline_report.html)

## 📊 各轮次指标对比

| 指标 | Round 1 基准 | Round 2 | Round 3 | 生产上线目标 |
|------|:-----------:|:-------:|:-------:|:----------:|
| Faithfulness (F) | 0.490 | 0.610 | **0.708** ✅ | ≥ 0.70 |
| Answer Relevancy (AR) | 0.633* | 0.643* | **0.832** ✅ | ≥ 0.70 |
| Context Precision (CP) | 0.947 ⚠️ | 0.950 ⚠️ | **0.949**（整合合成数据指标后 0.904）✅ | ≥ 0.85 |
| Context Recall (CR) | 0.903 ⚠️ | 0.937 ⚠️ | **0.917**（整合合成数据指标后 0.882）✅ | ≥ 0.85 |
| Answer Correctness (AC) | 0.590 | 0.594 | **0.663** ✅ | ≥ 0.65 |

> ⚠️ **Round 1 - Round 2 阶段 CP / CR 虚高：** R1–R2 评估集为初版合成数据（问题用词与原文相似 / 无关键词回避 / 缺少口语化表达）。Round 3 CP/CR 值为原评估集 185 条 + 合成 60 条模拟真实用户数据（添加：关键词规避/口语化表达/间接指代/俗称替换/引入场景噪声等）综合评估（见下文 Round 3 结果）。


## 🚀 Round 3 上线前结果

**1. 原评估集：三轮数据飞轮统一评估集（177 条）**

| 指标 | R2 | R3 | 变化 | 结论 |
|---|---|---|---|---|
| F | 0.613 | 0.706 | +0.092 | 显著（p = 0.000078） |
| AC | 0.596 | 0.662 | +0.065 | 显著（p = 0.000039） |
| CP | 0.954 | 0.948 | −0.005 | 无差异 |
| CR | 0.936 | 0.916 | −0.020 | 边缘下降（返回块数从 5 降到 1–2 的代价） |

- **提升来源：** 给模型的材料更少更准（优化 8），要求模型写得更短更聚焦（优化 9，答案中位长度 188 → 121 字），SP 规则从约 40 条精简到约 17 条。
- **短板：** reasoning 题 F 0.597 / AC 0.596，是唯一未达标的题型。

**2. 新增合成数据：模拟真实用户问法数据（60条）**

| 集合 | gold 命中 | 失败（拒答 / 跳过检索） | CP* | CR* | F | AC |
|---|---|---|---|---|---|---|
| 改写集 | 27 / 30 | 3 | 0.888 | 0.900 | 0.768 | 0.694 |
| 对抗集 | 25 / 30 | 4 | 0.833 | 0.833 | 0.664 | 0.680 |

\* 失败按 0 分计入。

- 同一题换成口语问法后，CP / CR 下降约 0.12，量化了原评估集的虚高幅度。
- 失败全部源于查询改写把俗称或口语翻错（如"越野滑雪"→ off-road skiing），检索分数降到 0.1 量级后被拒答线拦下。
- **真实 CP / CR**（245 条，失败计 0）：**CP 0.904、CR 0.882**，达到上线目标；只看合成数据为 0.861 / 0.867，在目标线附近。

**3. 上线前必须修复**

| 优先级 | 问题 | 方案 |
|---|---|---|
| P0 | 口语问法误拒 10%（60 条中 6 条） | 拒答前用用户原始问题让 4B 二次判定 |
| P0 | 跳过检索 2.7%（原题 185 条中 5 条），且会谎称"根据知识库" | 代码补检索兜底，空答案推送兜底话术 |
| P0 | 查询改写同步调用阻塞事件循环（单 worker 吞吐约 1 req/s） | 改为 `asyncio.to_thread`，再做并发压测 |
| P1 | reasoning 题 F / AC 未达标 | 生成后逐句接地核验 |
| P1 | 改写器俗称、专有名词翻译错误 | 提示词示例 → QLoRA 微调 |

## 🧪 Round 3 合成数据质量

**规模：** 30 条改写数据 + 30 条对抗性数据。

**生成模型：** Claude Sonnet 5.5 负责改写数据生成；Claude Opus 5.5 负责质量评估与修正，并生成对抗性数据。

**合成逻辑：**
- **改写集：** 从 Round 2 结果中按 topic 抽取 30 个高 Faithfulness 题（F ≥ 0.7，15 simple + 15 reasoning），避开 chunk 关键词改写为口语化问法，GT 沿用原题，用于逐题配对比较。
- **对抗集：** 从 Round 2 结果中（排除改写集）分层随机抽取 30 个 gold chunk（15 simple / 12 reasoning / 3 multi_hop，seed=42），按 **chunk → GT → 问题** 顺序生成：GT 只取自 gold chunk 原文并逐字校验依据句，问题采用以下四类问法：

| 问法 | 数量 | 示例 |
|------|------|------|
| 间接指代 | 12 | 史努比那个品种的狗为啥鼻子那么灵？ |
| 俗称替换 | 12 | 四脚蛇 → 蜥蜴、躁郁症 → 双相情感障碍、洋柿子 → 西红柿 |
| 前提核查 | 4 | 只陈酿了两年的苏格兰威士忌能叫 Scotch 吗？ |
| 场景噪声 | 2 | 医生说我有点贫血让我吃补铁的药片…… |


**参考文献：**
1. Filice et al. *Generating Q&A Benchmarks for RAG Evaluation in Enterprise Settings* (DataMorgana). ACL 2025 Industry. [arXiv:2501.12789](https://arxiv.org/abs/2501.12789)：可配置的问题类别，提升词汇与句法多样性。
2. Zhu et al. *RAGEval: Scenario Specific RAG Evaluation Dataset Generation Framework.* ACL 2025. [arXiv:2408.01262](https://arxiv.org/abs/2408.01262)：抽取原文依据，保证答案可追溯。
3. Sivasothy et al. *RAGProbe: An Automated Approach for Evaluating RAG Applications.* [arXiv:2409.19019](https://arxiv.org/abs/2409.19019)：构造问答变体，定位 RAG 失效点。


## 🔧 Round 2 诊断及优化（10 项：系统 7 项 + 评估 3 项）

Round 2 共 37 条 badcase，按根因归类后制定以下优化。

| # | 问题 | 方案 | 目标指标 |
|---|------|------|---------|
| 1 | Reasoning 题推理过程混入参数记忆（16 条，R2 reasoning F 0.498） | SP Rule 8 证据提取优先 + Rule 9 输出前自检 | F |
| 2 | Simple / Multi-hop 题参数记忆覆盖或桥接文档（8 条） | 题型分类器 + 题型差异化规则 + 动态 max_steps | F |
| 3 | 答案有据但偏离问题焦点（13 条） | SP Rule 10 问题焦点锚定 | AC |
| 4 | 重排器引发回归（新增 9 条） | ONNX INT8 CPU → TEI GPU FP16，候选池扩大 | F / CR |
| 5 | AR 大量误判为 0 | 中文提示词 + 回避判定对齐 RAGAS 原版语义（zh_v2） | AR（测量） |
| 6 | 合成评估集用词泄露导致 CP / CR 虚高 | 新增 60 条改写 + 对抗数据 | CP / CR（测量） |
| 7 | F 提升未经统计检验 | Mann-Whitney U + Bootstrap CI + Cohen's d | — |
| 8 | 父段落整段打分稀释相关句、固定阈值误杀 | 段落级 RRF + MaxP 重排 + 双阈值分档 + 4B 充分性判定 | F / CP |
| 9 | 答案过长（与 F、AC 负相关） | 首句作答 + 题型长度参考 + 文档取舍规则 | F / AC |
| 10 | 翻译改变原文语义、专有名词写法不一 | 中文转述规范 + 专有名词保真 | AC |

<details>
<summary><b>1. Reasoning 类型问题幻觉</b></summary>

**规模：** 16 条 badcase，占 badcase 总量 43%；Round 2 reasoning 题 F 0.498。

**根因：** 模型已检索到正确文档（CR = 1.0），但 ReAct 推理链在生成答案前混入了参数记忆。原有 Rule 7 只约束输出层，拦不住推理过程中的参数记忆渗入（post-rationalization）。

**优化方案：**

| 改动 | 核心操作 | 生效时机 |
|------|---------|---------|
| SP Rule 8：证据提取优先 | 推理前在思考过程中列出相关原文片段、标明文档空白；后续推理只能基于已列片段；最终答案不出现原文摘录 | 检索返回后的 `agent_node` 调用 |
| SP Rule 9：答案输出前自检 | 逐项核查答案中的推断与因果陈述能否在 Rule 8 列出的片段中找到依据；无依据内容一律删除，不以任何标注形式保留 | 同一 `agent_node`，紧接 Rule 8 |

> Rule 8 约束推理过程，Rule 9 约束输出，配合原有 Rule 7（证据不足时明确表态），覆盖推断类问题的完整生命周期。

**参考文献：**
- [Correctness is not Faithfulness in RAG Attributions](https://arxiv.org/abs/2412.18004)：RAG 系统中高达 57% 的引用为后验合理化，模型先形成答案再反向贴引用；先提取、再推理可以颠倒这一顺序。
- [Illocutionary Explanation Planning for Source-Faithful Explanations](https://arxiv.org/abs/2604.06211)：Evidence-First 结构在解释生成任务上平均带来 +34% source faithfulness（与 RAGAS Faithfulness 为同类指标，数值不直接对应）。
- [Dissociation of Faithful and Unfaithful Reasoning in LLMs](https://arxiv.org/abs/2405.15092)：CoT 存在忠实与非忠实两种模式，输出文本上几乎无法区分。

</details>

<details>
<summary><b>2. Simple / Multi-hop 类型问题幻觉</b></summary>

**规模：** 8 条 badcase，F < 0.3。

**根因：**
- **Simple：** 检索文档里有明确答案，模型却用参数记忆中的"印象"静默覆盖了文档事实。原有 Rule 1 的接地约束强度不够。
- **Multi-hop：** 模型检索到 Chunk A 和 Chunk B，但在两者之间搭桥的那一步用参数记忆补全，桥接没有文档依据。

**优化方案：**

| 改动 | 位置 | 内容 |
|---|---|---|
| 题型分类器 | 查询改写模块 | Qwen3-4B 新增输出字段 `QUESTION_TYPE`（simple / reasoning / multi_hop），关键词规则兜底，透传至 `AgentState` |
| 题型差异化规则 | `agent.py` `QUESTION_TYPE_RULES` | simple：以文档事实为准，与训练印象冲突时注明"根据知识库记录"；multi_hop：拆解原子子问题，依据尚未出现在已返回文档中的子问题单独检索，每个推理环节必须有文档依据，禁止用参数记忆桥接 |
| 动态 `max_steps` | `main.py`（流式 + 非流式） | simple=3 / reasoning=5 / multi_hop=7（原为固定 5 步） |

</details>

<details>
<summary><b>3. 答案焦点偏移</b></summary>

**规模：** 13 条 badcase，F ≥ 0.3（答案有文档支撑），AC < 0.4（方向偏离）。

**根因：**
1. **选择性接地**（最常见）：文档里有 A、B、C，问的是 A，模型却用 B 和 C 作答。
2. **理解偏差**：问"X 的影响是什么"，模型答成"X 是什么"。
3. **答案不完整**：文档包含完整答案，模型只提取了一部分。

**优化方案：** SP Rule 10 问题焦点锚定（所有题型）：调用 `knowledge_retriever` 前先用一句话明确直接答案对象；输出前对照答案对象自检，只覆盖了背景的要重新聚焦。Rule 8/9 防的是"推断无依据"，Rule 10 防的是"答非所问"，两者互补。

**参考文献：** [Before Reasoning Fails](https://arxiv.org/abs/2608.02011)（2026）；[What Would Fix This RAG Failure?](https://arxiv.org/abs/2608.08944)（2026）

</details>

<details>
<summary><b>4. 重排器引发回归</b></summary>

**规模：** 相比 Round 1 新增 9 条 badcase（R1 good → R2 bad）。

**根因：**
1. **量化分数漂移：** 重排模型为自行 INT8 量化的 ONNX CPU 版本，边界样本评分误差被 Top-K 截断放大，原本靠前的正确文档被排到第 3–5 位后跌出截断线。
2. **候选池过窄：** RRF 融合后送重排的候选数偏少，正确文档排在截断位之后时重排器永远看不到。

**优化方案：**

| 维度 | 内容 |
|------|------|
| 模型与部署 | 改用 bge-reranker-v2-m3 原版权重，由 TEI 在 GPU 上以 FP16 推理，消除自量化误差；重排延迟 1.359 s → 0.0364 s（提速 37×），检索层整体延迟 11 s → 6 s |
| 候选池扩大 | vector / BM25 召回 15 → 20，rrf_final_top_k 15 → 25 |
| 质量门控（已由优化 8 替换） | 当时采用固定阈值 0.45，全部低于阈值时保留最高分 1 条兜底 |

</details>

<details>
<summary><b>5. AR 大量误判为 0</b></summary>

**规模：** R2 中 21 条 AR 恰好为 0。

**根因（R3 更正）：** AR 恰好为 0 的机制是评审模型 3 次反推均判定答案"含糊回避"。中文指令把"给出答案 + 说明某部分资料缺失"也判为回避，而系统提示词恰恰要求说明知识库未覆盖的部分，导致越诚实的答案越容易被打 0 分。最初认定的"跨语言相似度归零"并非主因。

**优化方案（zh_v2）：** 保留 RAGAS 原生计分逻辑，只把回避定义对齐原版语义（仅"完全没有给出实质信息"才算回避），并换成 3 个中文示例。验证：24 条误判 0 分恢复 22 条，真正回避的答案 3/3 仍判为 0。

</details>

<details>
<summary><b>6. CP / CR 虚高</b></summary>

**根因：**
1. 评估集由 Claude Sonnet 5.5 读取知识库 Top 200 生成，问题用词天然来自 chunk 本身：BM25 精确命中 → CP 虚高；问题与 chunk 语义同源，向量高度相近 → CR 虚高。
2. 真实用户问题存在词汇鸿沟：口语、同义词、俗称多，句子短，与原文关键词重叠低。例如"滑雪比赛穿过森林那种是怎么玩的？"不会出现 cross-country、groomed course 等词。

**优化方案：** 新增 60 条合成数据，与原 183 条分批送入系统、单独评估（构造方法见下文 Round 3 合成数据）。

| 数据集 | 做法 | 规模 |
|------|------|------|
| 改写集 | 对已有高 F 原题做口语化改写，避开 chunk 原文关键词，GT 沿用原题 | 30 条 |
| 对抗集 | 从 gold chunk 出发按 chunk → GT → 问题顺序新生成，采用间接指代、俗称替换、前提核查、场景噪声四类问法 | 30 条 |

> 60 条样本量有限，主要用于判断 CP / CR 的方向与量级，不作为精确估计。

**参考文献：**

| 来源 | 结论 |
|------|------|
| [Beyond Benchmark Scores (2025)](https://arxiv.org/pdf/2609.14579) | 合成问题 CP 虚高，与真实用户查询分布存在根本差异 |
| [Can we Evaluate RAGs with Synthetic Data? (2025)](https://arxiv.org/pdf/2508.11758) | 合成评估集会误导检索策略选择 |
| [DataMorgana / SIGIR LiveRAG (2025)](https://arxiv.org/html/2501.12789v1) | 生产级 RAG 评估需要覆盖词汇鸿沟的多样化问题 |
| [Synthetic Question Generation for Retrieval Evaluation](https://suzyahyah.github.io/nlp/2024/08/03/Retrieval-Evaluation.html)（博客） | LLM 生成的问题继承源文本词汇，检索评估偏乐观 |

</details>

<details>
<summary><b>7. F 提升的统计显著性检验</b></summary>

**根因：** Round 1 → Round 2 的 F 提升（+0.12）仅凭均值对比，未区分系统改进与批次采样噪声。

**检验方法：**

| 方法 | 作用 | 判断标准 |
|------|------|---------|
| Mann-Whitney U | 判断两组分布差异是否显著 | p < 0.05 |
| Bootstrap 95% CI | 每组均值的置信区间 | 两组 CI 不重叠 |
| Cohen's d | 判断差异的实际大小 | 0.2 小 / 0.5 中 / 0.8 大 |

**结果（Batch 1 vs Batch 2）：**

| 指标 | Batch 1 | Batch 2 | Δ | p 值 | 结论 |
|------|--------|--------|---|------|------|
| Faithfulness | 0.490 | 0.610 | +0.120 | 0.0001 | ✅ 显著（d=0.444，小效应） |
| Answer Relevancy | 0.633 | 0.643* | +0.010 | 0.496 | 无显著差异 |
| Context Precision | 0.947 | 0.950 | +0.003 | 0.883 | 无显著差异 |
| Context Recall | 0.903 | 0.937 | +0.034 | 0.511 | 无显著差异 |

\* AR 检验使用的是修复中文 prompt 前的值。

> Round 3 全部优化落地后重新检验，以 p < 0.05 为显著标准，同时报告 Cohen's d 衡量提升幅度。

</details>

<details>
<summary><b>8. 检索层重构（small-to-big）</b></summary>

**根因：**
1. 原 RRF 按单句计分，同一父段落的多个命中句各占一个候选名额，25 个名额实际只有十几个不同段落。
2. 父段落由一段对话中多篇维基百科文章的句子拼接而成，对整段打分时相关句被稀释，gold 段落常被 0.45 阈值过滤，多数题只返回 1 块。

**优化方案：**
- **段落级 RRF：** 双路召回单句（向量 Top 30 + BM25 Top 30），每一路先把命中句归并到所属父段落，再按段落融合，取前 25 个不同段落送重排。
- **MaxP 重排：** 整段和命中单句在同一次 TEI 请求中打分，段落分取两者中的最高分。
- **双阈值分档（参考 CRAG，阈值在代码与配置中，与 SP 分离）：** 取代"固定阈值 0.45 + 保留 top1"，并去掉保底块（trace 显示保底块 8 个中 7 个分数 < 0.1）。最高分 ≥ 0.3 返回高分段落；0.1–0.3 由本地 Qwen3-4B 逐块判定能否回答，pass 的块带低置信标记并注入低置信 SP 模块；< 0.1 或判定全 fail 时由代码直接输出固定拒答话术。
- **多跳预算：** 多跳题整题设段落总预算，同题内多次检索不重复返回已返回的段落，预算用尽后由代码拦截检索。
- **工程化：** 重排分数不再返回给模型；参数全部配置化（环境变量覆盖，免重建镜像）；重排失败降级为 RRF 排序；每次检索输出 `RETRIEVAL_TRACE` 结构化日志与 Langfuse 候选明细，支撑参数离线标定。

</details>

<details>
<summary><b>9. 答案长度控制</b></summary>

**根因：** Round 2 中，答案长度与 F、AC 分别负相关 −0.45 和 −0.41；答案中位长度 188 字，GT 仅 64 字；最短三分位的答案 F 0.759 / AC 0.684，已经过线。

**优化方案（System Prompt）：**
1. 首句直接作答，给出最后一条相关事实就结束。
2. 按题型给长度参考：simple 约 100 字，reasoning 约 200 字，multi_hop 每个推理环节一句。超出部分必须是文档中与问题直接相关的事实。
3. 新增「文档取舍」规则：与问题无关的文档内容不写进答案。

</details>

<details>
<summary><b>10. 翻译与专有名词保真</b></summary>

**优化方案（System Prompt）：**
1. 答案全部用中文转述，不引用英文原句。
2. 专有名词统一写成"中文译名（英文原名）"，没有通用译名时保留文档原写法。
3. 翻译保持原文的确定程度、数量和因果强度，例如 may 不译成肯定，associated with 不译成导致，more than 不译成确切数字。
4. 专有名词以检索文档里的写法为准，不采信检索查询中翻译出来的写法，也不补充文档里没有的名、头衔或别称。

</details>

## 🔧 Round 1 诊断及优化（7 项）

Round 1 共 62 条 badcase（占 180 条的 34.4%，F=None 的 13 条已排除），其中 reasoning 题占 68%。按 4 类根因归类后制定以下 7 项优化，效果已在 Round 2 验证（F 0.490 → 0.610，p = 0.0001）。

| # | 问题 | 方案 | 目标指标 |
|---|------|------|---------|
| 1 | BM25 检索整段拼接文本，关键词匹配被长文本稀释 | BM25 改为单独索引单句 `cs_text`，与向量粒度对齐（small-to-big） | CP / CR |
| 2 | 重复 chunk 进入重排，同一内容多次计分、挤占 topK | RRF 融合后、重排前去重 | CP |
| 3 | 重排保底阈值过松（< −5.0 才判空），几乎不起作用 | 收紧为 −1.0，全部低于阈值时保留 top1 兜底 | F |
| 4 | 送重排的候选池偏小 | RRF 输出候选 10 → 15，最终 topK 仍为 5 | CR |
| 5 | 6 个主题知识库无对应内容（CR = 0） | 针对缺口主题生成解释性 chunk 增量入库 | CR |
| 6 | Reasoning 题参数记忆覆盖检索、答案发散 | System Prompt 三项：只答所问、低分区禁用内置知识、删除"自信"措辞 | F / AR |
| 7 | 查询改写扩大了问题范围 | 改写 Rule 6 "完整性" → "范围保真" | AR |

> 优化 3 与优化 6 中的低分阈值已先后被 Round 2 优化 4（TEI 重排 + 0.45 门控）和优化 8（双阈值分档 + 4B 判定）取代。

<details>
<summary><b>1. BM25 检索粒度与向量不一致</b></summary>

**根因：** 向量检索对单句 `cs_text` 做嵌入，BM25 却检索 `content`（多条 `cs_text` 拼接成的长文本），关键词可能命中拼接文本中的任意一句，匹配精度被稀释，BM25 路召回质量下降。

**优化方案：** 重建知识库，BM25 Function 单独索引 `cs_text` 字段，与向量嵌入粒度对齐；交给 LLM 的仍是完整 `content` 段落，遵循 Small-to-Big 策略：细粒度匹配、段落级生成。

**位置：** `src/core/redis-stream/create_milvus_collection.py`

</details>

<details>
<summary><b>2. 重复 chunk 进入重排</b></summary>

**根因：** 同一段落的多个单句命中后携带相同的父段落内容进入重排，导致同一内容被多次计分并占用 topK 名额。

**优化方案：** 在 RRF 融合之后、重排之前按内容去重，确保每条候选唯一（Round 3 进一步升级为段落级 RRF，见 Round 2 优化 8）。

**位置：** `src/core/ReAct_Agent/tools/retriever.py`

</details>

<details>
<summary><b>3. 重排保底阈值过松</b></summary>

**根因：** 原逻辑只有当所有重排分数 < −5.0 时才返回空，阈值极松，几乎所有低质量文档都会进入上下文。

**优化方案：** 阈值收紧为 −1.0；全部低于阈值时仍返回最高分 1 条，保证 LLM 始终有文档可参考，由 LLM 判断内容是否可用。

**后续：** 已被 Round 2 优化 4 与 Round 3 优化 8 取代。

</details>

<details>
<summary><b>4. 重排候选池偏小</b></summary>

**优化方案：** RRF 融合后送入重排的候选从 10 条扩至 15 条，重排器在更大的候选池中筛选，最终输出 topK 不变（5 条），提升精排质量上限。

**后续：** Round 2 进一步扩至 25（优化 4）。

</details>

<details>
<summary><b>5. 知识库覆盖缺口</b></summary>

**规模：** 6 个主题 CR = 0（Parenting / Pizza / Tomato / Field hockey / Unicorn / Dragon），知识库中不存在能回答相关问题的内容。

**优化方案：** 针对这 6 个主题生成英文解释性内容，经 Redis Stream 增量写入 Milvus，补齐覆盖缺口。

**位置：** `data/processed/chunks/wow_supplement_badcase6.json`

</details>

<details>
<summary><b>6. Reasoning 题幻觉与答案发散</b></summary>

**规模：** 42 条 badcase 属于"Reasoning 题参数记忆覆盖检索"（R1 reasoning F 仅 0.351）；32 条 AR &lt; 0.3 属于"答案方向偏离"。

**根因：** 模型在推断中调用参数知识填补逻辑缺口；问题问 A，答案却覆盖 A + B + C。

**优化方案（System Prompt）：**

| 改动 | 内容 | 目标指标 |
|------|------|---------|
| 回复风格第 3 条（新增） | 只回答被问到的内容，补充信息须后置并标注，避免答案发散 | AR |
| Rule 3 | 低分阈值 0.2 → 0.35，低分区明确禁止用内置知识填补 | F |
| Rule 4 | 删除"自信"措辞，改为"直接引用文档内容，严格遵守接地约束" | F |

</details>

<details>
<summary><b>7. 查询改写扩大问题范围</b></summary>

**根因：** 改写 Rule 6 要求"确保生成的问题包含所有必要上下文"，导致改写后的问题范围宽于原问题，例如原问"导演是谁"被扩展出"拍摄背景""上映时间"，检索和回答随之发散。

**优化方案：** Rule 6 改为"范围保真"：只填补指代消解所需的最小上下文，改写结果的覆盖范围不得宽于原始问题。

**位置：** `src/core/query_rewrite/query_rewriter.py`

</details>
