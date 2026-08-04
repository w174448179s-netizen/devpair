# 智谱 GLM JSON 输出带 markdown 代码块包裹

## 现象

智谱 GLM 系列模型（glm-4、glm-4.5-air 等）在要求输出 JSON 时，经常用 markdown 代码块包裹：

````
```json
{
  "persons": ["张三"],
  "locations": ["北京"]
}
```
````

直接用 Jackson/Gson 解析会报 `JsonParseException`，因为字符串以 ` ```json ` 开头，不是合法 JSON。

## 影响

- JSON 解析失败 → 业务流程中断
- 如果解析失败后直接吞错返回 null，上层逻辑拿到空结果，问题更难排查

## 原因

智谱 GLM 的训练数据中大量包含 markdown 格式的代码块，模型倾向于用 markdown 包裹所有"代码"格式的内容，包括 JSON。这不是 bug，是模型行为。

DeepSeek 偶尔也有类似行为，但频率低。Ollama 本地模型通常干净。

## 解决方案

**解析前预清理，不作为错误处理，是标准流程。**

```java
// 去除 markdown 代码块标记
String cleaned = raw.replaceAll("(?s)^\\s*```(?:json|JSON)?\\s*\\n?", "")
                    .replaceAll("(?s)\\n?\\s*```\\s*$", "")
                    .trim();
```

完整方案见 `~/devpair/rules/components/llm-output.md`，预清理工具类 `LlmOutputCleaner` 统一收敛所有模型的输出清理逻辑。

## 教训

1. **LLM 输出不可信，解析前必须预清理**——这不是异常处理，是标准流程
2. **不同模型行为不同**，清理逻辑要覆盖所有在用模型
3. **rawResponse 必须存日志**，解析失败时能快速定位是模型输出问题还是代码问题
4. **清理逻辑统一收敛**，不要在每个调用点散落重复代码
