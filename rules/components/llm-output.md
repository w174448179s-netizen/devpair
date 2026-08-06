# LLM 输出处理规范

> 适用场景：调用 LLM 并需要解析结构化输出（JSON）时。

## 一、JSON 输出必须显式约束

prompt 中必须明确告知 LLM 输出格式要求：

- 明确说"以 JSON 格式返回"
- 给出 JSON schema 或示例结构
- 禁止用"返回结构化数据"这种模糊描述

```text
正确（prompt 片段）：
请从文本中提取实体信息，以 JSON 格式返回，结构如下：
{
  "persons": ["人名"],
  "locations": ["地名"],
  "organizations": ["机构名"]
}
只返回 JSON，不要包含其他内容。

禁止：
请从文本中提取实体信息，以结构化格式返回。
```

## 二、输入侧预处理（降低概率，非万能）

传入 LLM 的文档内容可能包含 `"`、`\` 等 JSON 特殊字符。LLM 不是 JSON 序列化器，不能依赖它在输出时正确转义。

**传入 LLM 前，对文档内容做预处理：**

```java
/**
 * 对传入 LLM 的文本做安全预处理，消除 JSON 特殊字符隐患。
 *
 * @param raw 原始文档内容
 * @return 安全内容，可直接放入 LLM prompt
 */
public static String sanitizeForLlm(String raw) {
    if (raw == null) {
        return "";
    }
    // 将英文双引号替换为中文引号，降低 LLM 输出未转义引号的概率
    return raw.replace("\"", "「")
              .replace("\"", "」");
}
```

**注意：输入侧替换不是万能的。** LLM 可能在处理过程中将 `「」` 又转回 `"`，导致输出 JSON 中仍出现未转义引号。因此：

- 输入侧预处理只是**第一道防线**，降低出错概率
- **必须配合输出侧兜底**（见第四章修复重试规则）
- prompt 中补充指令：`"不要修改内容中的引号格式，保持原样输出"`
- **最可靠的方案是避免在 JSON 中放原始文档内容**——用提取结果（枚举、布尔、数字）代替原文

## 三、LLM 输出必须预清理

LLM 输出不可信，解析前必须做预清理。不同模型有不同行为：

| 模型 | 已知行为 | 清理方式 |
|------|---------|---------|
| 智谱 GLM | JSON 用 ` ```json ... ``` ` 包裹 | 去除 markdown 代码块标记 |
| DeepSeek | 偶尔在 JSON 前后加解释文字 | 提取首个 `{` 到末尾 `}` 的子串 |
| Ollama | 通常干净，但偶有多余换行 | trim + 去除首尾空白 |
| 通用兜底 | BOM 头、零宽字符 | 去除 BOM、不可见字符 |
| 内容引号未转义 | 文档内容中的 `"` 或 LLM 将 `「」` 转回 `"` 进入 JSON 字符串值未转义 | 输入侧预处理 + 输出侧 `escapeInnerQuotes` 修复 |

### 预清理工具类

```java
/**
 * LLM 输出预清理工具。
 * 在解析 JSON 前调用，处理已知的 LLM 输出格式问题。
 */
public final class LlmOutputCleaner {

    private LlmOutputCleaner() {}

    /**
     * 清理 LLM 输出，使其可被 JSON 解析器正确解析。
     *
     * @param raw LLM 原始输出
     * @return 清理后的 JSON 字符串
     */
    public static String cleanJson(String raw) {
        if (raw == null || raw.isBlank()) {
            return raw;
        }

        String result = raw;

        // 1. 去除 BOM 和不可见字符
        result = result.replaceAll("\\uFEFF", "")
                       .replaceAll("[\\u200B-\\u200D\\uFEFF]", "");

        // 2. 去除 markdown 代码块标记（智谱 GLM 常见行为）
        //    匹配: ```json\n...\n``` 或 ```\n...\n```
        result = result.replaceAll("(?s)^\\s*```(?:json|JSON)?\\s*\\n?", "")
                       .replaceAll("(?s)\\n?\\s*```\\s*$", "");

        // 3. 去除首尾空白
        result = result.trim();

        // 4. 提取首个 { 到末尾 } 的子串（处理 JSON 前后多余文字）
        int firstBrace = result.indexOf('{');
        int lastBrace = result.lastIndexOf('}');
        if (firstBrace >= 0 && lastBrace > firstBrace) {
            result = result.substring(firstBrace, lastBrace + 1);
        }

        return result;
    }
}
```

- 预清理在解析前执行，**不作为错误处理**，是标准流程的一部分
- 新发现的模型输出行为，更新本表格和清理逻辑

## 四、解析失败的三层处理

```
LLM 原始输出
    ↓
第 1 层：预清理（去 markdown、BOM、多余文字）
    ↓
第 2 层：解析（Jackson / Gson / 框架 Structured Output）
    ↓ 失败
第 3 层：修复重试（末尾逗号、缺失括号、单引号 → 双引号）
    ↓ 仍失败
抛异常，记录 rawResponse 供排查
```

### 修复重试规则

| 问题 | 修复方式 |
|------|---------|
| 末尾多余逗号 `",]` 或 `,}` | 正则去除 `,` |
| 缺失闭合括号 | 自动补全 `}` 或 `]` |
| 单引号包裹字符串 | 替换为双引号 |
| key 未加引号 | 正则补引号 |
| 字符串值内未转义的 `"` | 逐字符扫描，区分结构引号和内容引号，内容引号补 `\` 转义 |

**未转义引号修复思路**（最复杂的场景）：

```java
/**
 * 尝试修复 JSON 字符串值内未转义的双引号。
 * 原理：逐字符扫描，在字符串值内部遇到的 " 如果后面不是 JSON 结构字符
 *       （, } ] : 或换行+key），则判定为内容引号，补 \ 转义。
 *
 * @param json 清理后的 JSON 字符串
 * @return 修复后的 JSON 字符串
 */
public static String escapeInnerQuotes(String json) {
    StringBuilder sb = new StringBuilder();
    boolean inString = false;
    boolean escaped = false;

    for (int i = 0; i < json.length(); i++) {
        char c = json.charAt(i);

        if (escaped) {
            sb.append(c);
            escaped = false;
            continue;
        }

        if (c == '\\') {
            sb.append(c);
            escaped = true;
            continue;
        }

        if (c == '"') {
            if (!inString) {
                // 字符串开始
                inString = true;
                sb.append(c);
            } else {
                // 在字符串内遇到 "，判断是结束还是内容
                // 向后看，跳过空白，下一个有效字符是否是 JSON 结构字符
                int j = i + 1;
                while (j < json.length() && Character.isWhitespace(json.charAt(j))) {
                    j++;
                }
                char next = j < json.length() ? json.charAt(j) : '\0';

                if (next == ',' || next == '}' || next == ']' || next == ':') {
                    // 字符串结束
                    inString = false;
                    sb.append(c);
                } else {
                    // 内容引号，转义
                    sb.append("\\\"");

                    // 特殊处理：如果后面紧跟的是另一个 "（如 "" 配对），
                    // 跳过它避免重复转义
                    if (next == '"') {
                        i = j;
                    }
                }
            }
        } else {
            sb.append(c);
        }
    }

    return sb.toString();
}
```

- 该方法作为第 3 层修复的最后手段，仅在预清理 + 常规修复都失败后调用
- 不保证 100% 正确，复杂的嵌套引号场景可能误判
- 修复后仍解析失败，抛异常 + 记录 rawResponse

- 修复重试只做一次，不做无限循环修复
- 修复后仍解析失败，抛 `BusinessException`，**不静默吞错**
- rawResponse 完整存入日志（`ai_execution_log` 或 SLF4J），方便排查

## 五、优先使用 Structured Output

LangChain4j 的 Structured Output 能自动处理序列化，但仍需预清理：

```java
// 正确：Structured Output + 预清理兜底
@AiService
public interface EntityExtractService {

    @SystemMessage(fromResource = "prompts/extract-system.txt")
    EntityExtractResult extract(@UserMessage String text);
}

// 调用方在解析失败时，手动清理 + 重试
if (result == null) {
    // 框架 Structured Output 解析失败时的兜底
    String cleaned = LlmOutputCleaner.cleanJson(rawResponse);
    result = objectMapper.readValue(cleaned, EntityExtractResult.class);
}
```

- Structured Output 是首选，但不盲目信任框架的反序列化
- 框架反序列化失败时，走预清理 + 手动解析兜底

## 六、禁止项

| 禁止 | 替代方案 |
|------|----------|
| 直接解析 LLM 原始输出，不做预清理 | 先 `LlmOutputCleaner.cleanJson()` 再解析 |
| 传入 LLM 的文档内容不做预处理 | 先 `sanitizeForLlm()` 替换引号等特殊字符 |
| prompt 不说明输出格式 | 显式约束 JSON 结构 + 示例 |
| 解析失败直接吞错返回 null | 抛异常 + 记录 rawResponse |
| 修复重试无限循环 | 最多一次修复重试 |
| 信任框架 Structured Output 万能 | 加预清理兜底层 |
| 不同模型用不同的清理逻辑散落各处 | 统一收敛到 `LlmOutputCleaner` |

## 相关踩坑记录

- `~/devpair/notes/pitfalls/llm-json-output.md` — 智谱 GLM JSON 输出带 markdown 包裹 + 文档内容引号导致 JSON 解析失败
