# RAG 向量检索规范

## 向量化

- 嵌入模型：优先使用本地 Ollama 部署（如 nomic-embed-text），离线场景保底
- 备用方案：DeepSeek Embedding API（需联网）
- 文本分块大小：500-1000 字符，按段落边界切分，保留上下文重叠 100 字符
- 分块偏移统一为字节偏移，禁止字符偏移与字节偏移混用（详见踩坑记录 text-chunking.md）
- 切分降级链：段落边界 → 句子边界 → 固定窗口硬切分，禁止跳过句子边界直接硬切

## 存储

- 向量数据库：PgVector（与业务数据库共用 PostgreSQL，减少运维组件）
- 索引类型：IVFFlat（数据量 < 10 万条时使用，构建快）
- 向量维度：取决于嵌入模型，nomic-embed-text 为 768 维

## 检索策略

- 默认返回 Top-K = 5 条最相似结果
- 相似度阈值：cosine similarity < 0.7 的结果丢弃
- 混合检索：关键词 BM25 + 向量语义，取交集/并集可配置
- 支持元数据过滤（文档类型、创建时间、标签）

## 性能约束

- 单次检索超时：5s
- 知识库文档总数 < 10 万时不做分库分表
- 嵌入生成并发上限：2（本地模型资源有限）

## 相关踩坑记录

- 详见 ~/devpair/notes/pitfalls/pgvector.md
- 详见 ~/devpair/notes/pitfalls/text-chunking.md — 字符/字节偏移错位、固定窗口不尊重句子边界
