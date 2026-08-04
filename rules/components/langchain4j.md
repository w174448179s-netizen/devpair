# LangChain4j AI Services 规范

> 适用场景：通过 AI Services 声明式接口调用大模型，多模型配置。

## 一、多模型 Bean 配置

每个模型定义为独立的 `@Bean`，按用途命名，禁止方法内 `new`。

```java
@Configuration
public class ModelConfig {

    /** 智谱 glm-4.5-air，主力模型 */
    @Bean
    public ChatLanguageModel glmModel(ModelProperties props) {
        return OpenAiChatModel.builder()
                .apiKey(props.getGlm().getApiKey())
                .baseUrl(props.getGlm().getBaseUrl())
                .modelName(props.getGlm().getModelName())
                .temperature(0.3)
                .timeout(Duration.ofSeconds(60))
                .build();
    }

    /** DeepSeek，备用模型 */
    @Bean
    public ChatLanguageModel deepseekModel(ModelProperties props) {
        return OpenAiChatModel.builder()
                .apiKey(props.getDeepseek().getApiKey())
                .baseUrl(props.getDeepseek().getBaseUrl())
                .modelName(props.getDeepseek().getModelName())
                .temperature(0.3)
                .timeout(Duration.ofSeconds(60))
                .build();
    }

    /** Ollama 本地模型，离线保底 */
    @Bean
    public ChatLanguageModel ollamaModel(ModelProperties props) {
        return OllamaChatModel.builder()
                .baseUrl(props.getOllama().getBaseUrl())
                .modelName(props.getOllama().getModelName())
                .temperature(0.3)
                .timeout(Duration.ofSeconds(120))
                .build();
    }
}
```

- 模型参数（apiKey、baseUrl、modelName、temperature、timeout）全部从 `application.yml` 读取，通过 `@ConfigurationProperties` 注入
- **timeout 必须显式配置**，不同模型超时容忍不同（本地模型给宽一点）
- temperature 按用途设：抽取/分类 0.1-0.3，创作/摘要 0.5-0.7

## 二、AI Service 声明式接口

### 核心原则：prompt 用注解管理，禁止 Java 代码拼接字符串

```java
// 正确：注解声明 prompt
@AiService
public interface SummaryService {

    @SystemMessage("你是一个文本摘要助手，将输入文本压缩为 200 字以内的摘要。")
    String summarize(@UserMessage String text);
}

// 禁止：Java 代码拼接 prompt
String prompt = "你是一个摘要助手，请总结以下文本：" + text;
chatModel.generate(prompt);
```

### 多模型时用 @Qualifier 指定

```java
@Bean
public SummaryService summaryService(
        @Qualifier("glmModel") ChatLanguageModel glmModel) {
    return AiServices.builder(SummaryService.class)
            .chatLanguageModel(glmModel)
            .build();
}
```

- 一个 AI Service 接口绑定一个模型，禁止运行时动态切换模型
- 不同模型各有一套 AI Service 接口（如 `GlmSummaryService` / `DeepseekSummaryService`），由 Service 层选择调用

### 接口命名与组织

```
com.example.ai/
├── config/
│   └── ModelConfig.java              # 多模型 Bean 定义
├── summary/
│   ├── SummaryService.java           # AI Service 接口
│   └── SummaryAiServiceFactory.java  # AiServices.builder() 工厂
├── extract/
│   ├── ExtractService.java
│   └── ExtractAiServiceFactory.java
└── chat/
    ├── ChatService.java
    └── ChatAiServiceFactory.java
```

- 每个业务场景一个独立 AI Service 接口，禁止一个接口塞多种用途
- AI Service 接口只定义 AI 能力，业务编排放在调用方的 ServiceImpl 中

## 三、Structured Output

返回类型直接用 POJO 或 Enum，框架自动反序列化，禁止手动解析 JSON。

```java
// 正确：返回类型即结构化结果
@AiService
public interface EntityExtractService {

    @SystemMessage("从文本中提取实体信息，返回 JSON 格式。")
    EntityExtractResult extract(@UserMessage String text);
}

@Data
@Builder
public class EntityExtractResult {
    /** 人名列表 */
    private List<String> persons;
    /** 地名列表 */
    private List<String> locations;
    /** 机构列表 */
    private List<String> organizations;
}

// 禁止：手动解析 AI 返回的 JSON
String response = chatModel.generate(prompt);
JSONObject json = new JSONObject(response);  // 禁止手动解析
```

## 四、模型选择策略

| 场景 | 首选模型 | 备用 | 说明 |
|------|---------|------|------|
| 日常对话 / 摘要 / 翻译 | glm-4.5-air | DeepSeek | 性价比高，速度快 |
| 复杂推理 / 代码生成 | DeepSeek | glm-4.5-air | 推理能力强 |
| 离线 / 隐私场景 | Ollama 本地 | — | 无网络依赖 |
| Embedding 向量化 | Ollama nomic-embed-text | — | 详见 rag-vector.md |

- 模型选择逻辑放在 ServiceImpl 中，不在 AI Service 接口层判断
- 切换模型 = 切换调用的 AI Service 接口，不改接口本身

## 五、ChatMemory（按需配置）

- 无状态调用（摘要、抽取、翻译）不配 Memory
- 有状态对话才配 `MessageWindowChatMemory`，窗口大小 10-20 条
- Memory 存储方式按场景选：单用户用内存，多用户用持久化（DB/Redis）

## 六、异常处理

- 模型调用失败（超时、限流、API 错误）抛 `BusinessException`，由全局异常处理器兜底
- 不在 AI Service 调用方 try-catch 后包装返回值
- 降级策略：主力模型失败 → 切备用模型 → 仍失败抛异常，不静默吞错

## 七、禁止项

> 仅列出 always-apply.md 和 coding-rules.md 之外的 LangChain4j 专项禁止项。

| 禁止 | 替代方案 |
|------|----------|
| 方法内 `new ChatLanguageModel` | `@Bean` 单例注入 |
| Java 代码拼接 prompt 字符串 | `@SystemMessage` / `@UserMessage` 注解 |
| 手动解析 AI 返回 JSON | Structured Output（返回 POJO/Enum） |
| 一个 AI Service 接口塞多种用途 | 按场景拆分独立接口 |
| 运行时动态切换模型 | 多套 AI Service 接口，Service 层选择 |
| 无 timeout 的模型配置 | 显式设置 timeout |
| 无状态场景配 ChatMemory | 不配或按需配 |

## 相关踩坑记录

- `~/devpair/notes/pitfalls/spring-ai.md` — ChatClient 连接池问题（Model 必须单例）
- `~/devpair/notes/pitfalls/agent-design.md` — Tool 接口过度设计（若使用 Tool 功能时参考）
