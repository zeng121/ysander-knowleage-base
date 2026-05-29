# FTS、Embedding 与 Hybrid Retrieval：跨 Session 记忆召回技术说明

## 背景

在 Hermes / OpenClaw 这类 Agent 系统中，用户经常会问：

> “上次我们讨论的那个 Moltbook 帖子为什么被标记为 spam？”

这类问题属于**跨 session 记忆召回**。系统需要从历史对话中找出相关内容，并把它转化为当前可用的上下文。

常见召回技术包括：

- FTS：全文关键词搜索
- Embedding：语义向量搜索
- Hybrid Retrieval：关键词搜索 + 语义搜索混合召回

本文用具体例子解释三者的区别、联系和适用场景。

---

## 1. FTS 是什么？

FTS 全称是 **Full-Text Search**，即全文搜索。

可以把它理解成数据库里的“搜索引擎”，或者更直观地说：

> FTS 像增强版 Ctrl+F。

假设历史对话里有一句：

> 我们刚才在 Moltbook 上发了 fotor-skills 的帖子，但是被标记为 spam。

如果用户搜索：

```text
Moltbook
```

FTS 很容易找到。

如果用户搜索：

```text
fotor-skills spam
```

FTS 也容易找到。

原因是这些词**真的出现在原文里**。

---

## 2. FTS 擅长什么？

FTS 擅长精确关键词匹配，尤其适合工程场景。

例如：

### 文件路径

```text
~/.config/moltbook/credentials.json
```

### 错误码

```text
429
```

### 函数名

```text
session_search
```

### 环境变量

```text
FOTOR_OPENAPI_KEY
```

### 产品名 / 平台名

```text
Moltbook
fotor-skills
ComfyUI
OpenClaw
```

### ID / UUID

```text
5050b753-9d94-4537-b9df-b85c2e8e52b8
2fbc9cc20bfa
```

这些内容要求“字面精确”，FTS 很合适。

---

## 3. FTS 不擅长什么？

FTS 不太懂“意思”。

例如历史里写的是：

```text
那条帖子被系统判定为 spam。
```

但用户搜索：

```text
为什么被风控了？
```

FTS 可能找不到，因为原文里没有“风控”这个词。

再比如历史里写的是：

```text
outcome-first routing
```

但用户搜索：

```text
按用户目标选择工具
```

FTS 也可能找不到，因为字面词完全不同。

所以 FTS 的核心问题是：

> 它擅长找“同一个词”，但不擅长找“同一个意思”。

---

## 4. Embedding 是什么？

Embedding 可以理解成：

> 把一段文本变成一串数字，这串数字代表它的语义位置。

比如：

```text
帖子被标记为 spam
```

和：

```text
内容被平台风控了
```

这两句话字面不一样，但意思接近。

Embedding 会把它们映射到语义空间中比较接近的位置。

搜索时，不再只是问：

> 哪些历史内容包含这些关键词？

而是问：

> 哪些历史内容和这个问题的意思最接近？

---

## 5. Embedding 的例子

用户搜索：

```text
上次推广帖子为什么被平台限制？
```

历史原文可能是：

```text
Moltbook returned is_spam: true because the new account posted repeated fotor-skills links across submolts.
```

FTS 未必能命中“推广帖子”“平台限制”。

但 embedding 能理解这些表达之间的语义关系：

- 推广帖子 ≈ repeated fotor-skills links
- 平台限制 ≈ is_spam: true / 风控 / 反垃圾规则
- 上次 ≈ 历史 session

所以 embedding 更可能召回正确内容。

---

## 6. Embedding 擅长什么？

Embedding 擅长模糊语义召回。

例如：

| 用户说法 | 历史原文可能是 |
|---|---|
| 被风控了 | marked as spam |
| 评论没人回 | no new comments / activity stopped |
| 用户视角发帖 | practitioner perspective |
| 按目标路由 | outcome-first routing |
| 上次做的海报工具 | fotor-skills visual generation workflow |
| 平台限流 | HTTP 429 / rate limit |

它的优势是：

> 用户不需要记得当时的原词，只要说出大概意思，系统也能找回来。

---

## 7. Embedding 不擅长什么？

Embedding 不擅长精确字符串匹配。

例如用户问：

```text
那个 comment_id 是多少？
```

历史里有：

```text
17a5e678-ece4-4952-94e6-91c159a45ccc
```

这种 UUID 对 embedding 来说只是奇怪字符串，它不一定能精准召回。

再比如：

```text
FOTOR_OPENAPI_KEY
```

```text
line 7705
```

```text
~/.hermes/state.db
```

这些都是精确符号，FTS 通常比 embedding 更可靠。

所以 embedding 的核心问题是：

> 它擅长找“意思相近”，但不擅长找“字符串完全一致”。

---

## 8. Hybrid Retrieval 是什么？

Hybrid Retrieval 就是：

> FTS + Embedding 一起用。

因为两者互补。

| 搜索方式 | 擅长 | 不擅长 |
|---|---|---|
| FTS | 精确词、路径、ID、错误码、函数名 | 同义词、模糊表达、跨语言语义 |
| Embedding | 语义相似、同义说法、模糊回忆 | 精确字符串、代码符号、ID |
| Hybrid | 同时覆盖精确匹配和语义匹配 | 系统更复杂，需要融合排序 |

Hybrid 搜索时通常会同时跑两路：

```text
用户 query
├── FTS 搜索：找字面关键词匹配
└── Embedding 搜索：找语义相近内容
```

然后把两边结果合并、排序、去重。

---

## 9. 用 Moltbook 例子说明 Hybrid

用户问：

```text
上次 Moltbook spam 是怎么回事？
```

### FTS 会找：

```text
Moltbook
spam
fotor-skills
is_spam
```

如果历史里真的有这些词，FTS 会很快命中。

### Embedding 会找语义接近内容：

```text
被标记为 spam
触发反垃圾规则
新账号短时间连续推广
HTTP 429 rate limit
跨 submolt 发帖
平台风控
```

即使用户没说 `fotor-skills`，embedding 也可能召回相关内容。

### Hybrid 最终可能返回：

1. 查到 `is_spam: true` 的 session
2. 讨论不要删帖、停发 24–48 小时的 session
3. 发 marketing / agents 帖子的 session
4. 429 rate limit 的操作记录

这比单独 FTS 或单独 embedding 都更稳。

---

## 10. 为什么只用 FTS 不够？

因为人类回忆通常不是按原词回忆。

用户可能说：

```text
上次那个被平台限流的帖子
```

历史原文可能是：

```text
Moltbook returned HTTP 429 and later is_spam: true.
```

如果用户没说 `429`、`spam`、`Moltbook`，FTS 可能漏掉。

Embedding 能理解：

```text
平台限流 ≈ HTTP 429 / rate limit
```

所以能补上 FTS 的短板。

---

## 11. 为什么只用 Embedding 也不够？

因为工程任务经常依赖精确信息。

例如：

```text
那个 cronjob id 是多少？
```

历史里可能是：

```text
job_id: 2fbc9cc20bfa
```

这类内容 embedding 不可靠，FTS 更靠谱。

再比如：

```text
我们当时回复 monty 的 comment_id 是什么？
```

历史里可能是：

```text
comment_id: 17a5e678-ece4-4952-94e6-91c159a45ccc
reply_id: 5050b753-9d94-4537-b9df-b85c2e8e52b8
```

这些都应该靠 FTS 或结构化 metadata，而不是只靠 embedding。

---

## 12. 类比总结

### FTS 像 Ctrl+F

你得输入差不多的词，它才能找到。

### Embedding 像语义雷达

你不用说原词，它也能找意思相近的东西。

### Hybrid 像 Ctrl+F + 懂你意思的搜索引擎

既能找精确字符串，又能找模糊语义。

---

## 13. 和 Hermes session_search 的关系

当前 Hermes 的 session_search 大致流程是：

```text
FTS5 搜索历史 messages
→ 找到匹配 message
→ 按 session 去重
→ 加载完整 session messages
→ 截断 transcript
→ 调用 LLM 总结 top sessions
→ 返回摘要
```

慢点通常不在 FTS5 SQL 查询，而在：

```text
读取大 transcript + LLM 总结
```

因为每个候选 session 可能最多截取约 100k chars，并调用辅助模型总结。

---

## 14. 为什么 embedding 写入 DB 是一个优化方向？

如果把历史对话做 embedding 后写入数据库，查询时就可以：

```text
用户 query
→ query embedding
→ vector search 找语义相近 chunks
→ 不必完全依赖 FTS 关键词
```

这对中文、同义表达、模糊回忆特别有用。

例如用户问：

```text
上次被平台风控那事
```

即使历史原文是英文：

```text
is_spam: true because of repeated promotional links
```

embedding 也可能召回。

---

## 15. 但只做 embedding 还不够

如果流程还是：

```text
embedding 找到候选 session
→ 加载完整 transcript
→ 每次重新 LLM 总结
```

那总耗时仍然可能很高。

所以更完整的优化应该是：

```text
Embedding 索引
+ FTS 混合召回
+ chunking
+ cached summaries
```

---

## 16. 推荐的优化架构

### 写入时

```text
message 写入 messages 表
→ 写入 FTS 索引
→ 后台异步切 chunk
→ 生成 chunk summary / session summary
→ 生成 embedding
→ 写入 vector index
```

### 查询时

```text
用户 query
→ FTS 搜索关键词
→ embedding 搜索语义相近 chunk
→ metadata filter：时间、source、role、平台
→ 合并排序
→ 返回 cached summary
→ 必要时再 deep summarize 原始 transcript
```

---

## 17. Chunk 为什么比单条 message 更适合 embedding？

单条 message 往往上下文不完整：

```text
“好的，那就这样”
```

这句话单独 embedding 没意义。

更好的粒度是：

```text
用户需求
+ assistant 分析
+ tool 结果
+ 最终结论
```

也就是一个 task phase / conversation chunk。

例如：

```text
用户问为什么 Moltbook 没评论
→ assistant 查 cron/job/state/API
→ 发现没有新 activity
→ 结论是自然冷却，不是监控坏了
```

这个 chunk 的语义就完整得多。

---

## 18. 一个合理的表设计草案

```sql
CREATE TABLE session_chunks (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  start_message_id INTEGER,
  end_message_id INTEGER,
  source TEXT,
  role_mix TEXT,
  text TEXT NOT NULL,
  summary TEXT,
  token_count INTEGER,
  embedding_model TEXT,
  updated_at INTEGER
);
```

如果使用 sqlite-vec：

```sql
CREATE VIRTUAL TABLE session_chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[1536]
);
```

同时保留 FTS：

```sql
CREATE VIRTUAL TABLE session_chunks_fts USING fts5(
  text,
  summary,
  content=session_chunks
);
```

---

## 19. Fast / Deep 两种 recall 模式

可以把 session_search 分成两种模式。

### fast 模式

```text
FTS + embedding
→ 找到相关 chunk/session
→ 返回已有 summary/snippet
```

优点：快。

适合：

```text
“上次我们聊 Moltbook spam 是什么结论？”
```

### deep 模式

```text
先 fast 找候选
→ 再读取相关 transcript window
→ LLM 重新总结细节
```

优点：细。

适合：

```text
“把上次完整操作流程和所有 ID 都列出来。”
```

---

## 20. 最终结论

FTS、Embedding、Hybrid 的核心区别：

```text
FTS       = 字面命中
Embedding = 语义命中
Hybrid    = 字面 + 语义一起命中
```

对于跨 session 记忆召回，最稳的方案不是三选一，而是：

```text
FTS 精确召回
+ Embedding 语义召回
+ Metadata 过滤
+ Chunk summaries
+ Summary cache
+ 必要时 Deep summarization
```

一句话总结：

> FTS 负责找“你说过的词”，Embedding 负责找“你表达的意思”，Hybrid 负责把两者结合起来，让 Agent 既能精确查 ID/路径/错误码，也能理解用户模糊的历史回忆。
