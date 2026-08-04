---
alwaysApply: false
description: "编写DO、DTO、VO分层对象，对象转换逻辑、MapStruct转换器编码时启用"
---

# 对象分层与MapStruct转换规范（AI 系统指令）

你是一位遵循以下对象分层规范的资深 Java 后端工程师。生成对象定义与转换代码前先列出将遵守的规范要点，再生成代码。

## 一、分层定义

| 层 | 包路径 | 命名规则 | V1 状态 |
|---|---|---|---|
| DO | `com.ws.brain.repository.entity` | `{表名驼峰}`，不加 DO 后缀 | ✅ |
| DTO | `com.ws.brain.dto` | `{业务含义}DTO` | ✅ |
| Req | `com.ws.brain.controller.req` | `{动作}{对象}Req` | ⏳ V2 |
| VO | `com.ws.brain.controller.vo` | `{对象}VO` | ⏳ V2 |
| Convert | `com.ws.brain.convert` | `{实体名}Convert` | ✅ |

V1 流转链路：`DB → DO → Convert → DTO → Service 之间传递 → Convert → DO → DB`

## 二、DO 规范

- 每个字段对应数据库一列
- 命名不加 DO 后缀（如 `ConversationSource`）
- 状态字段用枚举类型，不用字符串
- 禁止 DO 直接暴露给外部调用方
- 禁止在 DO 中加业务逻辑方法
- 禁止 DO 直接跨 Service 传递，必须先转 DTO

## 三、DTO 规范

- DTO 命名加 DTO 后缀（如 `KnowledgeExtractionDTO`）
- DTO 不映射数据库表，按业务场景聚合数据
- 可以包含嵌套静态类
- 可以在 DTO 中做字段裁剪（不需要的 DO 字段不映射进来）
- 禁止 DTO 中包含数据库操作逻辑

## 四、MapStruct Convert 规范

- 存放路径：`com.ws.brain.convert`
- 注解：`@Mapper(componentModel = "spring")`
- 同名字段自动映射，不同名字段必须用 `@Mapping(source = "...", target = "...")` 显式声明
- 枚举转换自动处理同值映射
- 忽略字段：`@Mapping(target = "fieldName", ignore = true)`
- 嵌套对象：定义子 Convert 方法，MapStruct 自动调用
- Convert 可直接注入 Service 使用（MapStruct 生成 Spring Bean）

## 五、禁止项

| 禁止 | 替代方案 |
|------|----------|
| `BeanUtils.copyProperties()` | MapStruct Convert |
| `ModelMapper` | MapStruct Convert |
| 手写 setter 逐个赋值（超过 3 个字段） | MapStruct `@Mapping` |
| DO 直接作为方法参数跨 Service 传递 | 先转为 DTO |
| DO 直接返回给调用方 | 先转为 DTO |
| 在 DTO 中写数据库查询 | DTO 只承载数据，逻辑放 Service |
