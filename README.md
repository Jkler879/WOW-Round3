# WOW-数据飞轮（三轮）
针对ReAct Agent多轮对话系统的离线数据飞轮模块（共三轮）

1、Round1 诊断轮（获取系统基于ragas原生框架指标的基准baseline - 提前冻结的200条评估集 - judgemodel：qwen-plus）
2、Round2 验证轮（观测Round1后做的系统优化、知识库优化等带来的指标优化 - 提前冻结的200条评估集-统一judgemodel：qwen-plus）
3、Round3 压力轮（对标生产上线指标 + 系统能力天花板测试）
