# 文本分块踩坑记录

## 问题 1（Bug）：readRange 字符/字节偏移错位

### 现象

`ChatChunkingStrategy.scanFile()` 扫描文件时记录的是字符偏移（`line.length()` 返回字符数），但 `ChatContentProvider.readRange()` 用 `RandomAccessFile.seek()` 定位，`seek()` 的参数是字节偏移。对中文内容（UTF-8 下一个汉字 3 字节），字符偏移 ≠ 字节偏移，导致读取位置错乱。

```
文件内容：你好世界Hello

scanFile 记录：
  "你好世界" → offset=0, length=4（字符数）

readRange 用字节偏移定位：
  seek(0) → 读 4 字节 → 得到 "你好"（4 字节 = 1.33 个中文字符）
  实际期望：读 "你好世界"（12 字节）
```

### 影响

- 读取内容截断或错位，分块数据不完整
- 中文比例越高，偏移越离谱（纯英文不受影响，容易在测试时漏掉）
- 偶发性 bug——英文文档测试通过，中文文档上线就出问题

### 原因

`RandomAccessFile` 是面向字节的 API，`seek()` 和 `read()` 都按字节操作。而 `String.length()`、`BufferedReader.readLine()` 等字符 API 返回的是字符数。两个体系混用，偏移体系不一致。

### 解决方案

**统一为字节偏移，不要混用字符偏移和字节偏移。**

方案 A（推荐）：scanFile 也用字节偏移

```java
// scanFile 中记录字节偏移
long byteOffset = 0;
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream(file), StandardCharsets.UTF_8))) {
    String line;
    while ((line = reader.readLine()) != null) {
        // 字节长度 = 字符串 UTF-8 编码后的字节数 + 换行符
        int byteLength = line.getBytes(StandardCharsets.UTF_8).length + 1;
        // 记录 byteOffset, byteLength
        byteOffset += byteLength;
    }
}
```

方案 B：readRange 不用 RandomAccessFile，改用字符读取

```java
// readRange 中用字符偏移读取
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream(file), StandardCharsets.UTF_8))) {
    reader.skip(startCharOffset);
    char[] buf = new char[length];
    reader.read(buf);
    return new String(buf);
}
```

方案 A 更适合大文件随机读取（`RandomAccessFile` 性能好），方案 B 简单但大文件性能差。

### 教训

1. **字符偏移和字节偏移是两套体系，不能混用**——`RandomAccessFile` 是字节 API，`String.length()` 是字符 API
2. **纯英文测试会掩盖问题**——ASCII 字符 1 字节 = 1 字符，必须用中文测试用例
3. **文件读写层从一开始就统一偏移体系**——要么全用字节，要么全用字符

---

## 问题 2（质量）：固定窗口 fallback 不尊重句子边界

### 现象

当段落超过 6000 token 时，降级为固定窗口切分（5000 token/窗口），切分点是硬编码的字符位置，可能切在句子中间。

```
原文：这是一段很长的文字。其中包含多个完整的句子。
      每个句子表达一个完整的意思。切分时不应该在句子中间截断。

错误切分（固定窗口）：
  窗口1：这是一段很长的文字。其中包含多个完整的句|子。
  窗口2：每个句子表达一个完整的意思。切分时不应该在句|子中间截断。
         ↑ 句子被腰斩，语义断裂
```

### 影响

- 分块语义不完整，LLM 检索到的上下文是断句
- 向量检索质量下降——半个句子的语义表示不准确
- LLM 基于断句生成回答，可能产生幻觉或错误理解

### 原因

fallback 逻辑只考虑 token 数量上限，不考虑语义边界。硬编码字符位置切分是最简单的实现，但没有回溯到最近的句子结束符（。！？.!?）。

### 解决方案

**固定窗口切分时，回溯到最近的句子边界。**

```java
/**
 * 在 maxTokens 限制内，回溯到最近的句子边界进行切分。
 *
 * @param text    待切分文本
 * @param maxTokens 最大 token 数
 * @return 切分点（字符偏移）
 */
private int findSentenceBoundary(String text, int maxTokens) {
    // 1. 先按 token 限制估算字符上限
    int charLimit = estimateCharLimit(maxTokens);
    if (text.length() <= charLimit) {
        return text.length();
    }

    // 2. 在 charLimit 附近向前搜索最近的句子结束符
    String sentenceEndings = "。！？.!?\n";
    for (int i = charLimit; i > charLimit * 0.8; i--) {
        if (sentenceEndings.indexOf(text.charAt(i)) >= 0) {
            return i + 1; // 包含结束符
        }
    }

    // 3. 找不到句子边界，退回硬切分（比切在随机位置好不了多少，但至少有兜底）
    return charLimit;
}
```

切分策略优先级：

| 优先级 | 策略 | 条件 |
|--------|------|------|
| 1 | 段落边界切分 | 段落 ≤ maxTokens |
| 2 | 句子边界切分 | 段落 > maxTokens，回溯到句子结束符 |
| 3 | 固定窗口硬切分 | 找不到句子边界（极少数情况） |

### 教训

1. **切分必须尊重语义边界**——段落 > 句子 > 固定窗口，逐级降级
2. **fallback 不是放弃质量**——降级方案也要有最低限度的语义保持
3. **token 数量是上限约束，不是切分点**——在上限内找最近的语义边界才是正确做法
4. **中英文句子结束符不同**——中文 `。！？`，英文 `.!?`，都要覆盖
