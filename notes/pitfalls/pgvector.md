# PgVector 踩坑记录

## 2026-08：pgvector 扩展未安装导致启动失败
- **现象**：Spring Boot 启动时连接 PostgreSQL 报错，提示 vector 类型不存在
- **根因**：数据库未安装 pgvector 扩展
- **解法**：在数据库初始化脚本中执行 `CREATE EXTENSION IF NOT EXISTS vector;`
- **关联规则**：~/devpair/rules/components/rag-vector.md

## 2026-08：IVFFlat 索引未建导致全表扫描
- **现象**：数据超过 1000 条后，向量检索从毫秒级降到秒级
- **根因**：未建立 IVFFlat 索引，每次检索全表扫描
- **解法**：数据导入后建立 IVFFlat 索引（数据量 < 10 万时使用）
- **关联规则**：~/devpair/rules/components/rag-vector.md（索引选择）

## 2026-08：文本分块过小导致语义碎片化
- **现象**：RAG 检索结果与问题相关性差
- **根因**：文本分块设为 200 字符，切断了语义完整性
- **解法**：增大到 500-1000 字符，按段落边界切分，保留 100 字符上下文重叠
- **关联规则**：~/devpair/rules/components/rag-vector.md（文本分块）

## 2026-08：相似度阈值太低导致大量噪音
- **现象**：检索返回 5 条结果中只有 1-2 条相关
- **根因**：相似度阈值设为 0.5，大量不相关内容被召回
- **解法**：提高到 0.7，精准度明显改善，偶有漏召回可接受
- **关联规则**：~/devpair/rules/components/rag-vector.md（检索策略）

## 2026-08：更换嵌入模型后向量维度不匹配
- **现象**：更换嵌入模型后所有检索失败
- **根因**：新模型向量维度与旧列定义不一致
- **解法**：将模型名称和维度存入配置表，切换模型时自动重建向量列
- **关联规则**：~/devpair/rules/components/rag-vector.md
