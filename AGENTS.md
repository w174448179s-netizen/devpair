# DevPair — 魏胜的个人开发规范

## 我的身份

15 年 Java 后端架构师。个人项目，一个人开发。代码写给我自己用，不需要企业级的复杂度。

## 核心约束（永远生效，5 条）

1. Controller 只做参数校验和路由，不写业务逻辑。业务逻辑一律在 Service 层。
2. 异常统一用 @RestControllerAdvice 全局处理。禁止在 Controller 或 Service 中 try-catch 后手动包装返回值。
3. 所有 public 方法必须有 Javadoc 注释（功能说明、@param、@return）。
4. 使用 Lombok（@Data、@RequiredArgsConstructor），不手写 getter/setter/constructor。
5. 返回前端的数据必须用 DTO/VO，禁止直接返回 Entity。

## 技术偏好

- Java 17 + Spring Boot 3 + LangChain4j 1.17.2 + PgVector + Ollama
- 优先简洁方案，不过度设计
- 一个类只做一件事，单一职责优先
- 先跑通再优化，不要追求一步到位
- 不确定的地方主动问我，不要猜测

## 沟通偏好

- 所有回答使用中文
- 代码注释用中文
- 代码生成后自检是否符合上述 5 条核心约束
- 不要过度解释，直接做事

## 工作方式

我会通过 task 文档给你派发具体开发任务。

你收到 task 文档后，按以下步骤执行：

1. 读取 task 文档中「需加载的规则文件」列出的所有文件
2. 以这些规则文件为权威标准，编写代码和单元测试
3. 完成后逐条自检：代码是否匹配所有已加载规则
4. 如有冲突，优先遵循规则文件；项目特殊例外在 task 文档的「额外说明」里会提前写清楚

规则文件统一存放在 `~/devpair/rules/` 下。task 文档里会告诉你具体需要加载哪几个。
