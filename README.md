# WOW-数据飞轮（三轮）
针对ReAct Agent多轮对话系统的离线数据飞轮模块（共三轮）

1、Round1 诊断轮（获取系统基于ragas原生框架指标的基准baseline - 提前冻结的200条评估集 - judgemodel：qwen-plus）
2、Round2 验证轮（观测Round1后做的系统优化、知识库优化等带来的指标优化 - 提前冻结的200条评估集-统一judgemodel：qwen-plus）
3、Round3 压力轮（对标生产上线指标 + 系统能力天花板测试）

Round1,2已结束，具体优化步骤、指标提升在每轮完整报告中。

Round3目标：5项RAGAS原生指标对标生产上线标准 + 系统能力天花板测试

Round3执行前优化工作：
1、Reasoning 题 CoT 幻觉
规模：16 条badcase，占 badcase 总量的 43%。
根因：模型有正确的检索文档，但在推理过程中把自己的"记忆"当论据用了，没用文档。
优化1：在 agent.py 的 system prompt 中，对 Reasoning 类问题强制插入 Evidence Extraction前置步骤。模型必须先从检索结果中逐条列出原文证据，再在证据范围内推理
优化2：在 ReAct 最终答案生成前增加 self-check step：列出每条推断对应的文档依据，无法对应的自动删除
优化3：针对 6 个持续顽固主题（Veganism / Goodfellas / Horror film 等）补充解释性 KB chunk，让检索文档本身可以提供推断所需信息，增量入库至知识库。
