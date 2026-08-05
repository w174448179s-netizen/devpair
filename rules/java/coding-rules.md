# Java 编码规范

> always-apply.md 是 7 条铁律，本文件是展开细则。两者共同构成完整编码标准。

## 总则

1. 代码基准规范：严格执行《阿里巴巴Java开发手册（泰山版）》全部强制条款
2. 本文件为扩展规则，**当与阿里规约冲突时，以本文件为准**
3. 生成、修改、重构 Java 代码均不得违反上述两套规范

### 阿里规约关键强制点

1. 命名：包全小写、类大驼峰、方法小驼峰、常量全大写下划线；布尔变量 is/has 开头
2. 空指针防护：所有对象调用前判空，集合返回空集合而非 null，字符串用 hasText()
3. 日期时间：禁用 Date / SimpleDateFormat，统一 LocalDateTime
4. equals：常量对象调用 equals，避免空指针
5. 日志：使用 SLF4J（Lombok @Slf4j），禁止 System.out / e.printStackTrace()
6. SQL：禁止 select *，禁止 order by 数字，避免索引失效

---

## 一、命名规范

- 类名用英文，注释用中文
- 禁止缩写命名，必须完整语义（如 `conversationMessage` 而非 `convMsg`）
- 禁止 `example` 目录或包

## 二、实体类分层

| 层 | 命名规则 | 说明 |
|---|---|---|
| DO | `{表名驼峰}`，不加后缀 | 对应数据库表 |
| DTO | `{业务含义}DTO` | Service 间传递，按业务场景聚合 |
| Req | `{动作}{对象}Req` | 接口入参 |
| VO | `{对象}VO` | 接口出参 |
| Convert | `{实体名}Convert` | MapStruct 转换器 |

- DO 不加后缀，DTO/Req/VO 必须加后缀
- DTO 使用 `@Builder` 模式，禁止 setter 链
- 对象转换统一用 MapStruct，禁止 `BeanUtils.copyProperties()` 等反射拷贝

## 三、状态管理

- 所有表示状态、类型、阶段的字段，必须定义对应 Enum，数据库存 `code` 值
- 禁止魔法字符串
- 状态枚举必须包含：`code`、`desc`、`canTransitTo(Target)`、`isTerminal()`、`isRunning()`
- 类型枚举只需 `code` 和 `desc`
- 禁止在枚举中注入 Bean、访问 DB、调用网络、实现业务流程
- Entity 状态字段直接用枚举类型，MyBatis 通过 `defaultEnumTypeHandler` 自动转换

## 四、JDK 17 特性

- 状态流转/策略选择：增强 `switch` 表达式，禁止传统 switch 穿透
- 类型判断+转换：`instanceof` 模式匹配，禁止先判断再强转
- 多行文本：Text Blocks `"""`，禁止字符串拼接
- 流式操作：多步过滤/映射/分组用 Stream API；单步简单循环用 for-each

## 五、依赖注入

- 禁止手写构造函数，统一用 `@RequiredArgsConstructor`（Lombok 生成构造函数）
- 禁止 `@Autowired` 字段注入，必须构造函数注入（`@RequiredArgsConstructor` 自动实现）
- 需要注入的依赖声明为 `private final`，`@RequiredArgsConstructor` 仅为 final 字段生成构造函数
- Spring 配置类（`@Configuration`）中的 `@Bean` 方法例外，不适用此规则

```java
// 正确
@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {
    private final UserMapper userMapper;
    private final UserConvert userConvert;
}

// 禁止
@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private UserMapper userMapper;  // 禁止字段注入

    public UserServiceImpl(UserMapper userMapper) {  // 禁止手写构造函数
        this.userMapper = userMapper;
    }
}
```

## 六、Service 分层

每个 Service 必须有接口 + 实现类。接口命名 `XxxService`，实现类 `XxxServiceImpl`。

接口放在模块包根路径下，实现类放在 `impl/` 子包中，禁止混在同一目录：

```
com.example.{module}/
├── XxxService.java           ← 接口
└── impl/
    └── XxxServiceImpl.java   ← 实现类
```

## 七、异常处理

- 业务异常统一用项目自定义异常（如 `BusinessException`）
- Service 层抛业务异常，上层统一 catch 并记日志
- 禁止空 catch 块，至少 `log.warn()` + 注释说明

## 八、数据库与 MyBatis

- DB snake_case → Java camelCase（MyBatis 自动映射）
- XML 中 namespace = Mapper 接口全路径
- 禁止 XML 中 `${}` 拼接 SQL，查询参数一律 `#{}`
- 数据库连接池必须有超时配置
- PostgreSQL JSONB 字段：MyBatis XML 中写入和查询必须显式类型转换 `#{metadata}::jsonb`，否则报类型不匹配错误

## 九、配置管理

- 禁止硬编码路径、URL、密钥
- 敏感信息通过环境变量注入
- 业务配置统一前缀，通过 `@ConfigurationProperties` 读取
- `application.yml` 必须有密码配置（即使为空也要占位）
- 日志级别不能是 DEBUG

## 十、Web 安全

- CORS 必须限制来源，禁止 `allowAllOrigins`
- HTTP 客户端通过 `@Bean` 单例注入，禁止方法内 `new`

## 十一、注释规范

- 所有 private 方法必须有注释，说明职责、参数含义、返回值
- 每次调用 private 方法前必须有一行注释说明调用意图
- Entity/DTO 每个字段必须有注释
- 复杂判断（超过 2 个条件）、正则、非显而易见的业务规则，必须加行内注释
- getter/setter（Lombok 生成）、简单注入声明、一目了然的赋值不需要注释

## 十二、方法规范

- 入参必须做 null 和边界检查
- 单方法不超过 30 行，单一职责
- 禁止魔数，用常量或枚举
- `if`/`for`/`while` 必须用花括号

## 十三、Import 规范

- 禁止通配符 `import xxx.*`
- 删除代码时必须同步删除已不再使用的 import 语句
- 禁止保留仅用于注释代码的 import
- 生成或修改代码完成后，确保无 unused import 警告

## 十四、禁止项速查

> 仅列出 always-apply.md 7 条铁律之外的禁止项。

| 禁止 | 替代方案 |
|------|----------|
| 空 catch 块 | 至少 `log.warn()` + 注释说明 |
| `example` 包名 | 删除或用实际包名 |
| XML 中 `${}` 拼接 SQL | `#{}` |
| JUnit 原生断言 | AssertJ `assertThat` |
| 传统 switch 穿透 | 增强 `switch` 表达式 |
| 先 instanceof 再强转 | `instanceof` 模式匹配 |
| 字符串拼接多行文本 | Text Blocks |
| 缩写命名 | 完整语义命名 |
| 方法超过 30 行 | 拆分 |
| 通配符 import `*` | 明确导入单个类 |
| 删除代码后保留无用 import | 同步删除对应 import |
| `BeanUtils.copyProperties()` | MapStruct Convert |
| `Date` / `SimpleDateFormat` | `LocalDateTime` |
| `System.out` / `e.printStackTrace()` | SLF4J @Slf4j |
| CORS `allowAllOrigins` | 限制具体来源 |
| 方法内 `new` HTTP 客户端 | `@Bean` 单例注入 |
| DTO setter 链 | `@Builder` 模式 |
| 日志级别 DEBUG | INFO 或以上 |
| 手写构造函数 | `@RequiredArgsConstructor` |
