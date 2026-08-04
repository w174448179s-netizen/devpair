# DevPair — AI 开发搭档

> 个人开发规则仓库 + 任务编排工作流

解决核心问题：**AI 写代码不记你的规范，每次都要从头教。**

---

## 这是什么

DevPair 不是知识库，不是 Wiki，不是第二大脑。

**DevPair = 个人开发规则仓库 + 任务文档工作流。**

你和 AI 的分工：

| 环节 | 你的工作 | AI 的工作 |
|------|---------|----------|
| 规则制定 | 定义核心约束、编码规范、踩坑经验 | 按模板格式化填充到 `rules/` |
| 架构决策 | 做技术选型、取舍判断 | 写决策记录 `decisions/` |
| 任务拆分 | 确定任务顺序、模块边界 | 读取 `commands/split-tasks.md`，生成 tasks 文档 |
| 日常开发 | 指定当前执行哪个 task | 读取 task 文档列出的规则文件，按规则写代码并自检 |
| 知识沉淀 | 判断哪些经验可复用 | 读取 `commands/capture-knowledge.md`，更新规则或踩坑记录 |

核心一句话：**你管"做什么"和"为什么"，AI 管"怎么做"和"按什么标准做"。**

---

## 工作原理

整个系统只有一条规则：**你命令 AI 读什么，AI 就读什么。**

```
你说「读取 X」→ AI 读 X → X 里列了 Y、Z → AI 读 Y、Z → AI 干活
```

没有自动加载，没有软链接，没有工具兼容性。你直接对 AI 说话，AI 去读文件。

- `AGENTS.md`：对话开始时读取，AI 了解你的偏好和核心约束
- `rules/`：task 文档里列了哪些，AI 就读哪些
- `commands/`：你说「读取 commands/split-tasks.md」，AI 就按指令执行
- `tasks/`：你说「读取 ./tasks/task-02-xxx.md」，AI 就读那个 task 并干活

---

## 目录结构

```
~/devpair/
│
├── AGENTS.md                               # 个人偏好和核心约束（对话开始时读取）
├── README.md                               # 本文件
│
├── commands/                               # AI 可执行指令
│   ├── split-tasks.md                      # 拆分架构 → 生成 tasks 文档
│   ├── execute-task.md                     # 执行单个子任务开发
│   └── capture-knowledge.md                # 开发完成 → 知识沉淀
│
├── rules/                                  # 开发规则
│   ├── core/
│   │   └── always-apply.md                 # 5 条核心铁律（每个 task 必加载）
│   ├── java/
│   │   ├── coding-rules.md                 # Java 编码规范（阿里规约 + 项目扩展）
│   │   └── object-convert-rules.md         # 对象分层与 MapStruct 转换规范
│   ├── testing/
│   │   └── testing-rules.md                # 单元测试 + 集成测试规范
│   ├── components/
│   │   └── rag-vector.md                   # RAG 向量检索规范
│   └── output/                             # JSON 输出格式规范（待填充）
│
├── decisions/                              # 技术决策记录（待填充）
│
├── notes/                                  # 踩坑记录
│   └── pitfalls/
│       ├── spring-ai.md                    # ChatClient 连接池问题
│       └── agent-design.md                 # Agent 工具接口过度设计
│
├── templates/                              # 任务模板（待填充）
│
└── reference/                              # 参考资料（待填充）
```

### 项目侧目录结构

```
your-project/
│
├── tasks/                                  # 本项目任务文档
│   ├── overview.md
│   ├── task-01-xxx.md
│   └── task-02-xxx.md
│
└── src/                                    # 项目源码
```

---

## 规则体系

规则分两层，互为补充，零重复：

| 层级 | 文件 | 定位 |
|------|------|------|
| 铁律 | `rules/core/always-apply.md` | 5 条不可违反的底线，每个 task 必加载 |
| 细则 | `rules/java/coding-rules.md` | 铁律的展开 + 阿里规约关键点 + 14 章扩展规则 |
| 细则 | `rules/java/object-convert-rules.md` | MapStruct Convert 使用规范，补充 coding-rules.md 第二章 |
| 细则 | `rules/testing/testing-rules.md` | 单元测试 + 集成测试规范 |
| 组件 | `rules/components/rag-vector.md` | RAG 向量检索组件规范 |

**规则之间的关系**：always-apply.md 是"是什么"（铁律），coding-rules.md 是"怎么做"（细则），其他文件按需加载。

---

## 设计原则

1. **规则是个人资产，不属于任何项目** — 放在 `~/devpair/`，独立 Git 仓库
2. **一切靠你命令 AI 去读** — 没有自动加载，没有软链接，你说读什么 AI 就读什么
3. **规则在 task 文档里显式引用** — 每个 task 明确列出"需加载的规则文件"路径
4. **踩坑记录与规则分离** — 规则文件只写"该怎么做"，踩坑在 `notes/pitfalls/`
5. **不写代码** — DevPair 本身只是一堆 Markdown 文件，零维护成本

---

## 日常开发循环

```
架构讨论（跟 AI 对话，敲定 v1 范围）
    ↓
对 AI 说：「读取 ~/devpair/commands/split-tasks.md，按指令执行」
    → AI 生成 tasks/overview.md + task-xx.md
    ↓
对 AI 说：「读取 ./tasks/task-02-xxx.md，按文档要求执行」
    → AI 读规则 → 写代码 → 自检
    ↓
对 AI 说：「读取 ~/devpair/commands/capture-knowledge.md，按指令执行」
    → AI 沉淀经验到 ~/devpair/
    ↓
下个项目直接复用 ~/devpair/ 里的所有规则
```

### 单任务开发循环

```
1. 你对 AI 说：「读取 ./tasks/task-xx.md，按文档要求执行」
2. AI 读取 task 文档
3. AI 读取「需加载的规则文件」列表里的每一个文件
4. AI 生成代码 + 单元测试
5. AI 自检：代码是否符合所有规则？
6. 你审核代码
   ├── 通过 → 下一个任务
   └── 不通过 → 告诉 AI 哪里不对 → AI 修改 → 回到步骤 5
```

---

## 三条 AI 指令

### 指令 1：拆分架构，生成 tasks

```
读取 ~/devpair/commands/split-tasks.md，按指令执行
```

AI 会基于架构方案在 `./tasks/` 目录生成整套开发文档。

### 指令 2：执行单个子任务

```
读取 ./tasks/task-02-xxx.md，按文档要求执行
```

AI 会读取 task 文档、加载规则文件、编写代码并自检。

### 指令 3：知识沉淀

```
读取 ~/devpair/commands/capture-knowledge.md，按指令执行
```

AI 会梳理开发过程中的踩坑和经验，更新规则文件或踩坑记录。

---

## 规则演进策略

| 场景 | 操作 |
|------|------|
| 踩了一个新坑 | 在 `notes/pitfalls/` 下追加，评估是否更新规则文件 |
| 发现规则写错了 | 直接改文件，`git commit` |
| 新学了一个技术 | 在 `notes/` 下写笔记 |
| 做了一个重要决策 | 在 `decisions/` 下新建文件 |
| 某条规则在多个项目重复出现 | 提取到 `rules/` |

---

## License

个人使用，私有仓库。
