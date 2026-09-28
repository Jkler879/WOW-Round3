# WOW-数据飞轮（三轮）
针对ReAct Agent多轮对话系统的离线数据飞轮模块（共三轮）

1、Round1 诊断轮（获取系统基于ragas原生框架指标的基准baseline - 提前冻结的200条评估集 - judgemodel：qwen-plus）
2、Round2 验证轮（观测Round1后做的系统优化、知识库优化等带来的指标优化 - 提前冻结的200条评估集-统一judgemodel：qwen-plus）
3、Round3 压力轮（对标生产上线指标 + 系统能力天花板测试）

Round1,2已结束，具体优化步骤、指标提升在每轮完整报告中。

Round3目标：5项RAGAS原生指标对标生产上线标准 + 系统能力天花板测试

### Round 2 优化内容：

  #### 1. Reasoning 类型问题幻觉

  **规模：** 16 条 badcase，占 badcase 总量的 43%。

  **根因：** 模型已检索到正确文档（Context Recall = 1.0），但在 ReAct推理链中混用了参数记忆作为论据，最终答案未能忠实于检索结果，未能拦截推理过程中的参数记忆渗入。
  
  **优化方案：** 在 `agent.py` 的 system prompt 中强制插入 Evidence Extraction 前置步骤。模型必须先从检索结果中逐条列出原文证据，再在证据范围内推理，同时声明文档未覆盖的部分。此改动仅涉及 system prompt，LangGraph 图结构不变，实际生效时机为 **初次检索返回结果后**的agent_node 调用。

  **相关论文观点支撑：**
  - 1、Evidence-First 是切断 post-rationalization 的核心手段。RAG 系统中高达 57%
  的引用为后验合理化——模型先形成答案再反向贴引用，Citatio是事后标签而非推理起点。Evidence Extraction
  通过强制"先提取、再推理"颠倒这一顺序。
  
    **引用：**
  *Correctness is not Faithfulness in RAG Attributions* —https://arxiv.org/abs/2412.18004
     
  - 2、Chain-of-Illocution（CoI）在解释生成任务上验证，Evidence-First 结构平均带来 +34% source faithfulness 提升（与 RAGAS原生
  Faithfulness 为同类指标，非直接对应数值）。
  
    **引用：**
  *Illocutionary Explanation Planning for Source-Faithful Explanations* —https://arxiv.org/abs/2604.06211

  - 3、LLM 的 CoT
  推理链存在忠实与非忠实两种模式，二者在输出文本上几乎无法区分，只有在推理前强制提取证据才能切断参数记忆渗入的通道。
  
    **引用：**
  *Dissociation of Faithful and Unfaithful Reasoning in LLMs* —https://arxiv.org/abs/2405.15092


  优化2：在 ReAct 最终答案生成前增加 self-check step：列出每条推断对应的文档依据，无法对应的自动删除
