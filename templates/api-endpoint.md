# API 接口开发模板

> 用于生成单个 API 接口的 task。比 CRUD 轻，适用于非标准接口（如文件上传、检索、导出等）。

## 标准开发步骤

```
1. 定义接口契约 → 入参 Req + 出参 VO
2. Service 接口 + 实现（业务逻辑）
3. Controller（参数校验 + 路由）
4. 异常处理（业务异常 → 全局处理器）
5. 单元测试（Service 层）
6. 接口联调
```

## 需加载的规则文件

- `~/devpair/rules/core/always-apply.md` — 5 条铁律
- `~/devpair/rules/java/coding-rules.md` — 编码规范
- `~/devpair/rules/java/object-convert-rules.md` — 对象转换规范
- `~/devpair/rules/testing/testing-rules.md` — 测试规范
- `{涉及特定组件时补充，如 rag-vector.md}`

## 交付物清单（参考）

```
src/main/java/com/example/{module}/
├── controller/
│   └── {Action}Controller.java          # 接口入口
├── service/
│   ├── {Action}Service.java             # 接口
│   └── impl/
│       └── {Action}ServiceImpl.java     # 实现
├── dto/
│   ├── req/
│   │   └── {Action}Req.java             # 入参
│   └── vo/
│       └── {Action}VO.java              # 出参
└── convert/                             # 如需对象转换
    └── {Entity}Convert.java

src/test/java/com/example/{module}/
└── service/{Action}ServiceImplTest.java
```

## 验收检查清单

- [ ] Controller 只做参数校验和路由
- [ ] 入参校验完整（@Valid / 手动判空）
- [ ] 异常由全局处理器统一处理
- [ ] 返回 VO，不直接返回 DO/Entity
- [ ] Service 用 @RequiredArgsConstructor 注入
- [ ] 所有 public 方法有 Javadoc
- [ ] 单元测试覆盖正常路径 + 异常路径
- [ ] 涉及组件的，遵守对应组件规则文件

## task 文档示例

```markdown
# Task XX：{接口功能名}

## 所属模块
{模块名称}

## 前置依赖
Task XX（依赖的 Service/表已就绪）/ 无

## 任务目标
实现 {接口功能描述}，包含：
- 接口路径：{HTTP 方法} {path}
- 入参：{关键字段}
- 出参：{关键字段}
- 业务规则：{核心逻辑}

## 需加载的规则文件（执行前必须逐一读取）
- ~/devpair/rules/core/always-apply.md
- ~/devpair/rules/java/coding-rules.md
- ~/devpair/rules/java/object-convert-rules.md
- ~/devpair/rules/testing/testing-rules.md
- {涉及组件时补充对应规则文件}

## 交付物清单
{按上方交付物清单模板填充具体类名}

## 验收标准
1. 功能满足任务目标描述
2. 代码严格遵守【需加载的规则文件】内全部约束
3. 处理边界异常：{具体异常场景}
4. 代码注释完整，符合编码规范
5. 单元测试覆盖核心逻辑（正常路径 + 异常路径）

## 额外说明
{项目特殊调整，没有则删除}
```
