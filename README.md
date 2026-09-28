# WOW-离线数据飞轮（三轮）
针对ReAct Agent多轮对话系统的离线数据飞轮模块（共三轮）

1、Round1 诊断轮（获取系统基于ragas原生框架指标基准baseline - 提前冻结的200条评估集 - judgemodel：qwen-plus与生产系统分离）

2、Round2 验证轮（观测Round1后做的系统优化、知识库优化等带来的指标优化 - 提前冻结的200条评估集-统一judgemodel：qwen-plus）

3、Round3 压力轮（对标生产上线指标 + 系统能力天花板测试）

Round1, 2已结束，具体优化步骤、指标提升在每轮完整报告中。

Round1 诊断轮完整 report 链接：

Round2 验证轮完整 report 链接：

Round3 目标：5 项 RAGAS 原生指标对标生产上线标准 + 系统能力天花板测试

### Round 2 诊断及系统优化：

#### 1. Reasoning 类型问题幻觉（优化前：FaithFulness 0.64）

  **规模：** 16 条 badcase，占 badcase 总量的 43%。
  
  **根因：** 模型已检索到正确文档（CR = 1.0），但 ReAct 推理链在生成答案前混入了参数记忆，Rule 7
  仅约束输出层，无法拦截推理过程中的参数记忆渗入（post-rationalization）。

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

