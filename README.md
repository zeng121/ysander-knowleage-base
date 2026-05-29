# Ysander 知识库

围绕 **Agent 工程化** 与 **Hermes / OpenClaw 系统** 沉淀的技术笔记，内容来自线上 agent 服务的迁移探索与代码阅读实践。

## 文档索引

| 文档 | 主题 | 一句话概述 |
| --- | --- | --- |
| [Agent 垂类应用最佳实践](agent-vertical-app-best-practices.md) | Agent 工程化 | 从 Codex → Hermes 迁移中沉淀的 agent 行为约束、测试体系与风险控制认知 |
| [Hermes 消息到回复的关键路径](hermes-message-to-response-flow.md) | 系统架构 | gateway 模式下一条消息从平台入口到 agent、工具调用、持久化与发送的完整链路 |
| [跨 Session 记忆召回技术说明](fts-embedding-hybrid-session-recall.md) | 检索 / 记忆 | FTS、Embedding 与 Hybrid Retrieval 三种召回技术的区别、联系与适用场景 |
| [数据清洗方法论](agent-data-cleaning.md) | 数据工程 | 从原始数据到分析就绪数据的三层架构、清洗决策框架与工程实践 |

## 适用读者

- 正在把 LLM agent 从原型推向线上服务的工程师
- 关注 agent 可靠性、可测试性与风险控制的开发者
- 需要理解 Hermes 消息流转与跨 session 记忆机制的同学

## 约定

- 所有文档为独立主题，可单独阅读
- 内容基于具体实践与本机代码整理，随认知迭代持续更新
