# 数据清洗方法论

> 从原始数据到分析就绪数据的通用原则、设计决策和工程实践。

---

## 一、核心认知

### 1.1 清洗不是"把数据弄干净"

很多人把数据清洗理解为去重、去空值、格式化。这是最浅的一层。

真正的清洗决策是：**哪些字段留在原始层、哪些提取到清洗层、哪些扔掉**。每一个决策都是在"查询性能 vs 信息保真度"之间做权衡。

### 1.2 三层数据架构

```
原始层 (Raw)       →  原封不动，不可篡改，永远可回溯
清洗层 (Cleaned)   →  提取高频字段、分区、扁平化，日常查询用
聚合层 (Aggregated) →  预计算指标，dashboard 直接读
```

清洗层的定位：**不是替代原始层，是加速查询的缓存层**。任何清洗层的字段都能在原始层找到对应来源。

### 1.3 坏清洗的代价

坏清洗比不清洗更危险。不洗只是慢；洗错了是不可逆的信息损失，而且下游分析会基于残缺数据得出错误结论。

---

## 二、清洗决策框架

面对一个原始表，问自己四个问题：

### 2.1 留什么——完整性检查

**原则：完整 JSON/结构体全保留。**

任何嵌套的、结构化的、未来可能有分析价值的字段，保留一份完整副本。清洗层不应该成为数据坟墓。

```
✅ questions_json  STRING   -- 完整保留
✅ question_text   STRING   -- 再提取一份扁平版本
❌ 只留 question_text，丢弃 questions_json
```

**自检**：如果两年后有人想回答一个你今天没想到的问题，你的表能支持吗？能→好；不能→你丢了数据。

### 2.2 拆什么——高频字段提取

**原则：分析中经常作为 WHERE / GROUP BY / 聚合对象的字段，提取为列。**

不要提前优化——你不知道的字段就留在 JSON 列里。只提取你确定会用到的。

提取的触发条件：
- 有业务语义的 ID（用户、会话、消息、任务）
- 分类字段（平台、角色、状态、类型）
- 时间戳（查询必须有时间范围）
- 耗时/金额等数值（需要聚合计算）
- 高频过滤条件中的布尔标记

不需要提取的：
- 内部序列号（seqId、内部递增 ID）
- 心跳/调试事件
- 完全不确定用途的字段——留在 JSON 里

### 2.3 算什么——预计算衍生字段

**原则：派生一个 BOOLEAN 或计数，比每次都解析 JSON 便宜。**

```
原始数据：answers JSON 里有没有 message_error 事件
✅ has_error BOOLEAN            -- 查询时直接 WHERE has_error = true
❌ 每次查都 json_extract 找 error 事件
```

常见的衍生字段：
- `has_xxx`：是否有图片/错误/评分/风控
- `xxx_count`：图片数量/工具调用次数
- `xxx_list`：收集所有出现的某个值
- 时间差：end - start

### 2.4 丢什么——明确放弃

有意识地决定**不保留**某些信息，而不是无意识地丢失。

应该丢的：
- 调试/内部噪音（心跳、序列号）
- 完全确定的冗余（同时存了 epoch ms 和 ISO timestamp，留一个）
- 超出存储/查询预算的巨型文本（截断，在 JSON 列里保留完整版）

**不应该默认为"用不到所以不保存"**——你今天用不到不代表以后用不到。

---

## 三、Schema 设计原则

### 3.1 列命名

- 全小写 + 下划线
- 布尔字段 `has_xxx` / `is_xxx`
- 计数字段 `xxx_count`
- JSON 保留列 `xxx_json`
- URL/列表 `xxx_urls`（ARRAY 类型，不要逗号分隔字符串）

### 3.2 类型选择

| 场景 | 推荐类型 | 避免 |
|------|---------|------|
| ID / 短文本 | STRING | - |
| 真/假 | BOOLEAN | 别用 0/1 字符串 |
| 时间戳 | TIMESTAMP | 别存 epoch 让下游自己转 |
| 毫秒耗时 | BIGINT | 不需要 TIMESTAMP 精度 |
| 多值列表 | ARRAY\<STRING\> | 别用逗号拼接 |
| 嵌套对象 | STRING (JSON) | 不要拆成几十个列 |

### 3.3 分区

**总是分区。** 任何时间序列表都要分区，否则 Athena/Spark 每次都全表扫描。

- 分区键通常是 `date`（DATE 类型）
- 分区粒度：天级（99% 的场景够用）
- 分区字段从业务时间派生，不是 ETL 执行时间

### 3.4 保留溯源性

每条清洗后的记录必须能追溯到原始数据：
- 保留原始 ID（MongoDB `_id`、Kafka offset、上游表主键）
- 列名用 `source_id` / `origin_id`

---

## 四、常见反模式

### 4.1 "JOIN 式清洗"

把清洗做成 JOIN 多个表的宽表。问题：
- 原始数据变更时宽表难维护
- JOIN 逻辑和清洗逻辑耦合

**原则：清洗层做提取和格式化，聚合层做 JOIN。**

### 4.2 "过度扁平化"

把嵌套 JSON 拆成几十个列。问题：
- Schema 臃肿，新增字段要 ALTER TABLE
- 大量 NULL 列
- 维护成本远超收益

**原则：JSON 保留 + 高频字段提取。只提取 20% 最常用的字段。**

### 4.3 "无声丢弃"

字段默默没了，没有任何记录说明为什么丢、丢了什么。这是最危险的反模式。

**原则：任何字段的丢弃决策都要有明确理由，并在文档中记录。**

### 4.4 "只洗当前需要的数据"

今天只查最近 30 天，就只建 30 天的分区逻辑。三个月后需求变了，历史数据没法回溯。

**原则：清洗逻辑应该是全量友好的。通过分区控制查询范围，不是通过逻辑裁剪。**

---

## 五、ETL 工程实践

### 5.1 幂等性

清洗作业必须可重跑。方式：
- `INSERT OVERWRITE` 目标分区（而非 append）
- 依赖原始层的时间戳，不依赖 ETL 执行时间

### 5.2 增量策略

- **追加不可变数据**（日志、消息）：每天跑前一天分区，`INSERT OVERWRITE` 当天分区
- **可变数据**（用户状态、订单）：全量刷新或 CDC
- **小表**：全量刷新，成本可接受

### 5.3 质量检查

ETL 跑完不应该"没有报错就是成功"。至少检查：

| 检查项 | 方式 |
|--------|------|
| 行数波动 | 日增量 vs 历史均值 |
| 空值率 | 核心字段 NOT NULL 比例 |
| 时效性 | 最新分区是否生成 |
| 数据漂移 | 某分类占比突然变化 |

### 5.4 工具选择

| 场景 | 工具 |
|------|------|
| SQL 清洗（Athena/Trino/Spark SQL） | Athena scheduled query、dbt |
| 复杂逻辑清洗 | Spark（PySpark）、Flink |
| 流式清洗 | Kafka Streams、Flink SQL |
| 编排 | Airflow、Step Functions、Dagster |

---

## 六、案例：Agent 对话数据清洗

以 Fotor AI Agent 的对话数据为例，说明上述原则的应用。

### 6.1 背景

- 原始层：`aws_docdb_chat_message`——MongoDB DocumentDB 经 Athena Connector 同步，每行的 `_doc` 是完整的 JSON 文档
- 旧清洗层：`user_agent_chat_message`——早期 ETL 产物，丢失了大量字段
- 需要重建一个高质量清洗层

### 6.2 旧表的问题（反模式实例）

| 反模式 | 具体表现 |
|--------|---------|
| 无声丢弃 | `state`、`processTime`、`appId`、`riskDetects` 全部丢失，没有文档记录原因 |
| 过度简化 | `answers` 从完整事件流（thinking → tool → heartbeat → image）压成 `output_text` + `output_image`，丢失推理过程和工具调用链路 |
| 缺少衍生字段 | 没有 `has_error`，每次都要解析 JSON 找 error 事件 |

### 6.3 新表设计决策

应用第二节的四问框架：

**留什么**：
- `questions_json`、`answers_json`、`dify_json`、`risk_detects_json` 全保留

**拆什么**：
- 维度：uid、conversation_id、message_id、app_id、role
- 时间：create_time（epoch → TIMESTAMP）、date（分区键）
- 状态：state、rating

**算什么**：
- `has_image_input` / `has_image_output`——布尔衍生
- `has_error` / `has_rating` / `has_risk_detect`——布尔衍生
- `tool_names` / `models` / `task_ids`——从 events 中收集
- `question_text` / `answer_text` / `thinking_text`——文本提取
- `process_time_ms`——原始字段直接透传

**丢什么**：
- 内部事件（seqId、heart_beat_ping）
- 冗余时间戳（createTime 转换后不保留 epoch 版本）

### 6.4 目标 schema 要点

32 个字段，按职责分组：
- 维度（5）：message_id, conversation_id, uid, app_id, role
- 时间（3）：create_time, process_time_ms, date
- 状态（3）：state, rating, has_rating
- 用户内容（5）：questions_json, question_text, has_image_input, input_image_count, input_image_urls
- assistant 内容（5）：answers_json, answer_text, thinking_text, has_image_output, output_image_urls
- 工具调用（4）：tool_calls_json, tool_names, task_ids, models
- 错误（4）：has_error, error_code, error_type, error_message
- 风控/反馈/关联（4）：has_risk_detect, risk_detects_json, dify_json, feedbacks
- 元数据（4）：source_id, agent_type, provider, deleted

---

## 七、检查清单

设计一个新的清洗表时，逐项检查：

- [ ] 原始层的每一列，清洗层要么保留、要么提取、要么明确记录丢弃原因
- [ ] 嵌套 JSON/结构体有完整保留列（`xxx_json`）
- [ ] 高频 WHERE/GROUP BY 字段提取为列
- [ ] 有布尔衍生字段减少 JSON 解析（`has_xxx`）
- [ ] 时间戳从 epoch/string 转为 TIMESTAMP
- [ ] 有 date 分区键
- [ ] 保留了原始 ID（source_id）
- [ ] ETL 逻辑幂等（INSERT OVERWRITE）
- [ ] 有质量监控（行数、空值率、时效性）
