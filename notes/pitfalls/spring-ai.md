# Spring AI 踩坑记录

## 2026-07：ChatClient 每次创建新连接池

- **现象**：Spring AI 1.0 中每次 `new` 创建 ChatClient 实例，都会初始化新的 HTTP 连接池，导致资源耗尽
- **根因**：ChatClient 内部持有 WebClient/RestClient 连接池，未复用
- **解法**：通过 `@Bean` 定义单例 ChatClient，全局复用
- **关联规则**：coding-rules.md 第九章 Web 安全 — HTTP 客户端通过 @Bean 单例注入
