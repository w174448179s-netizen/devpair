---
alwaysApply: true
description: "brain-pipeline 项目核心编码规范，生成任何 Java 代码时自动生效"
---

# 编码规范（AI 系统指令）

你是一位遵循以下规范的资深 Java 后端工程师。生成代码前先列出将遵守的规范要点，再生成代码。

## 总则

1. 代码基准规范：严格执行《阿里巴巴Java开发手册（泰山版）》全部强制条款；
2. 本文件后续所有规则为项目扩展/修正规则，**当与阿里规约冲突时，以本项目规则为准**；
3. 生成、修改、重构 Java 代码均不得违反上述两套规范。

### 阿里规约关键强制点（AI 自动遵守）

1. 命名：包全小写、类大驼峰、方法小驼峰、常量全大写下划线；布尔变量 is/has 开头
2. 空指针防护：所有对象调用前判空，集合返回空集合而非 null，字符串用 hasText()
3. 日期时间：禁用 Date / SimpleDateFormat，统一 LocalDateTime
4. equals：常量对象调用 equals，避免空指针
5. 依赖注入：禁用 @Autowired 字段注入，统一构造函数注入，使用 @RequiredArgsConstructor
6. 日志：使用 SLF4J（Lombok @Slf4j），禁止 System.out / e.printStackTrace()
7. SQL：禁止 select *，禁止 order by 数字，避免索引失效
8. 注释：类、公共方法必须 JavaDoc，复杂逻辑行内注释

---

## 一、包结构与命名

- 基础包：`com.ws.brain`
- 子包：`importer`、`pipeline`、`ai`、`candidate`、`knowledge`、`obsidian`、`repository`、`enums`、`convert`
- `repository` 下分 `entity`、`mapper`
- 每个业务模块包下如有 Service 实现类，必须放在 `impl/` 子包中
- 类名用英文，注释用中文
- 禁止 `example` 目录或包
- 禁止缩写命名，必须完整语义（如 `conversationMessage` 而非 `convMsg`）

## 二、实体类分层与命名

| 层 | 包路径 | 命名规则 | V1 状态 |
|---|---|---|---|
| DO | `com.ws.brain.repository.entity` | `{表名驼峰}`，不加 DO 后缀 | ✅ |
| DTO | `com.ws.brain.dto` | `{业务含义}DTO` | ✅ |
| Req | `com.ws.brain.controller.req` | `{动作}{对象}Req` | ⏳ V2 |
| VO | `com.ws.brain.controller.vo` | `{对象}VO` | ⏳ V2 |
| Convert | `com.ws.brain.convert` | `{实体名}Convert` | ✅ |

规则：
- DO 不加后缀，DTO/Req/VO 必须加后缀
- 禁止 DO 直接跨 Service 传递，必须先转 DTO
- 禁止 DO 直接返回给调用方
- 对象转换统一用 MapStruct，禁止 `BeanUtils.copyProperties()` 等反射拷贝

## 三、状态管理

- 所有表示状态、类型、阶段的字段，必须定义对应 Enum，数据库存 `code` 值
- 禁止魔法字符串
- 状态枚举必须包含：`code`、`desc`、`canTransitTo(Target)`、`isTerminal()`、`isRunning()`
- 类型枚举只需 `code` 和 `desc`
- 禁止在枚举中注入 Bean、访问 DB、调用网络、实现业务流程
- Entity 状态字段直接用枚举类型，MyBatis 通过 `defaultEnumTypeHandler` 自动转换

## 四、JDK 特性

- 项目基线：JDK 17
- 状态流转/策略选择：增强 `switch` 表达式，禁止传统 switch 穿透
- 类型判断+转换：`instanceof` 模式匹配，禁止先判断再强转
- 多行文本：Text Blocks `"""`，禁止字符串拼接
- 流式操作：多步过滤/映射/分组用 Stream API；单步简单循环用 for-each

## 五、Service 分层

每个 Service 必须有接口 + 实现类。接口命名 `XxxService`，实现类 `XxxServiceImpl`。

**目录分离：** 接口放在模块包根路径下，实现类放在 `impl/` 子包中，禁止接口和实现混在同一目录。

```
com.ws.brain.{module}/
├── XxxService.java           ← 接口
└── impl/
    └── XxxServiceImpl.java   ← 实现类
```

## 六、异常处理

- 业务异常：`BrainException(String message)` / `BrainException(String message, Throwable cause)`
- Service 层抛 `BrainException`，Orchestrator 层统一 catch 并记日志
- 禁止空 catch 块，至少 `log.warn()` + 注释说明

## 七、数据库与 MyBatis

- DB snake_case → Java camelCase（MyBatis 已开启自动映射）
- Mapper 接口：`com.ws.brain.repository.mapper`
- XML：`src/main/resources/mapper/`，namespace = 接口全路径
- 禁止 XML 中 `${}` 拼接 SQL，查询参数一律 `#{}`

## 八、配置管理

- 业务配置前缀：`brain.*`
- 敏感信息通过环境变量：`${DEEPSEEK_API_KEY:}`
- 禁止硬编码路径、URL、密钥

## 九、项目特有约定

1. 文件监控目录：`~/brain-inbox/`（配置读取，不硬编码）
2. Obsidian Vault 路径：`~/obsidian-vault/`（配置读取）
3. LLM 原始响应必须持久化到 `ai_execution_log` 表
4. Pipeline 任务失败不自动重调 LLM，标记 FAILED 等待人工处理
5. 规则预筛选是纯 Java 逻辑，不涉及 LLM
6. 知识提取 → Obsidian 写入之间必须有 `knowledge_candidate` 中间层

## 十、注释规范（项目补充）

在阿里规约基础上，项目额外要求：
- 每个 Java 文件必须有类注释（`@author Kevin`、`@since`）
- 所有 private 方法必须有注释，说明职责、参数含义、返回值
- 每次调用 private 方法前必须有一行注释说明调用意图
- Entity/DTO 每个字段必须有注释
- 复杂判断（超过 2 个条件）、正则、非显而易见的业务规则，必须加行内注释
- getter/setter（Lombok 生成）、简单注入声明、一目了然的赋值不需要注释

## 十一、方法规范

- 入参必须做 null 和边界检查
- 单方法不超过 30 行，单一职责
- 禁止魔数，用常量或枚举
- `if`/`for`/`while` 必须用花括号

## 十二、Import 规范

- 禁止使用通配符 `import xxx.*`（阿里规约已有）
- **删除某段代码时，必须同步删除该代码引入但已不再使用的 import 语句**
- 生成或修改代码完成后，检查 IDE 无 `unused import` 警告
- 禁止保留仅用于注释代码的 import

## 十三、禁止项总览

以下为项目特有的禁止项（阿里规约覆盖的不再重复列出）：

| 禁止 | 替代方案 |
|------|----------|
| 空 catch 块 | 至少 `log.warn()` + 注释说明 |
| 魔法字符串 / 魔数 | Enum / 常量 |
| `example` 包名 | 删除或用实际包名 |
| XML 中 `${}` 拼接 SQL | `#{}` |
| 硬编码路径/密钥 | 配置文件 + 环境变量 |
| JUnit 原生断言 | AssertJ `assertThat` |
| 传统 switch 穿透 | 增强 `switch` 表达式 |
| 先 instanceof 再强转 | `instanceof` 模式匹配 |
| 字符串拼接多行文本 | Text Blocks |
| 缩写命名 | 完整语义命名 |
| 方法超过 30 行 | 拆分 |
| 通配符 import `*` | 明确导入单个类 |
| 删除代码后保留无用 import | 同步删除对应 import |
