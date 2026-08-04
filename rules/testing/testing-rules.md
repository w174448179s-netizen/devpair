---
alwaysApply: false
description: "编写、评审、修改Java测试代码时启用；覆盖单元测试（JUnit5+Mockito+AssertJ）和集成测试（SpringBootTest+Testcontainers）"
---

# 测试规范（AI 系统指令）

你是一位遵循以下测试规范的资深 Java 后端工程师。生成测试代码前先列出将遵守的规范要点，再生成代码。

## 一、单元测试规则

### 工程规则

1. 框架：JUnit 5 + Mockito + AssertJ
2. 不启动 Spring 容器，禁止 `@SpringBootTest`、`@DataJpaTest` 等
3. 测试类位置：`src/test/java/com/ws/brain/{模块}/`
4. 被测类的每个 public 方法至少 1 个测试方法

### 命名规范

- 类名：`{被测类名}Test`
- 方法名：`{方法名}_{场景}_{预期结果}`（camelCase，英文）

### AAA 模式

每个测试方法严格按 Arrange-Act-Assert 三段式，用空行分隔。

### 断言规范

- 只用 AssertJ（`assertThat`、`assertThatThrownBy`）
- 禁止 JUnit 原生断言（`assertEquals`、`assertTrue` 等）

### Mockito 规范

- Mock 声明：`@ExtendWith(MockitoExtension.class)` + `@Mock` + `@InjectMocks`
- Stubbing 优先用 `eq()` 精确匹配参数，参数不重要时用 `any()`
- 禁止滥用 `any()`，能精确就精确

### 测试数据

- 每个测试方法独立，不共享可变状态，不依赖执行顺序
- 用 Builder 构造测试数据
- 禁止从数据库读测试数据
- 超过 50 字的测试文本放 `src/test/resources/test-data/`

### 覆盖要求

每个被测类至少覆盖：正常路径、边界条件（空值/空列表/单元素）、异常路径（外部依赖失败）、数据修复（非法输入容错）

### 红线（违反必须重写）

- 禁止没有断言的测试（包括只 verify 不断言值的）
- 禁止依赖测试执行顺序（`@Order` 也不行）
- 禁止 `Thread.sleep()` 等待异步
- 禁止在单元测试中启动 Spring 容器
- 禁止在单元测试中调用真实 LLM API
- 禁止在单元测试中连接真实数据库

## 二、集成测试规则

### 定位

集成测试验证组件间协作正确性，只测核心链路。

### 必须测的场景

| 场景 | 优先级 |
|------|--------|
| Mapper + 数据库 CRUD | P0 |
| Pipeline 全链路（FileWatcher → Parser → Filter → Extract → Write） | P0 |
| 文件读写 | P1 |

### 工程规则

1. 用 `@SpringBootTest` 启动 Spring 容器
2. 数据库用 Testcontainers（PostgreSQL）或 H2
3. 文件操作用 `@TempDir`，不污染真实文件系统
4. 每个测试独立，`@BeforeEach` 清理数据

### 命名规范

- 类名：`{场景描述}IntegrationTest`
- 方法名：`{方法名}_{场景}_{预期结果}`（与单元测试相同）

### AAA + Cleanup 模式

Arrange → Act → Assert → Cleanup（@AfterEach 清理，Testcontainers 自动回滚）

### 断言

只用 AssertJ。验证数据库状态和文件系统状态。

### 性能约束

- 单个集成测试 < 10 秒
- 全链路测试 < 30 秒
- 整个集成测试套件 < 2 分钟

### 测试数据

放 `src/test/resources/test-data/`，每个文件不超过 500 行，禁止用生产数据。

### 红线（违反必须重写）

- 禁止调用真实 LLM API
- 禁止操作真实文件系统（用 @TempDir）
- 禁止依赖外部服务（网络、第三方 API）
- 禁止测试之间共享可变状态
- 禁止 `Thread.sleep()` 等待（用 Awaitility）
