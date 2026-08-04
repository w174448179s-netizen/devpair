# 对象转换规范（MapStruct）

> coding-rules.md 第二章定义了分层命名规则，本文件补充 MapStruct Convert 的具体使用规范。

## 一、Convert 基本规范

- 注解：`@Mapper(componentModel = "spring")`，由 Spring 容器管理
- 命名：`{实体名}Convert`，一个 DO 对应一个 Convert
- 同名字段自动映射，不同名字段必须用 `@Mapping(source = "...", target = "...")` 显式声明
- 枚举转换自动处理同值映射（code 值一致即可）
- 忽略字段：`@Mapping(target = "fieldName", ignore = true)`
- 嵌套对象：定义子 Convert 方法，MapStruct 自动调用
- Convert 可直接注入其他 Convert 或 Service（MapStruct 生成 Spring Bean）

## 二、转换方向

```
DB ↔ DO ↔ Convert ↔ DTO ↔ Service
                       ↕
                    Convert
                       ↕
                  VO / Req → Controller
```

- DO → DTO：Service 间传递前转换
- DTO → DO：写库前转换
- DTO → VO：返回前端前转换
- Req → DTO：Controller 接收参数后转换

## 三、禁止项

> coding-rules.md 禁止项速查已包含 `BeanUtils.copyProperties()`，以下为补充。

| 禁止 | 替代方案 |
|------|----------|
| `ModelMapper` | MapStruct Convert |
| 手写 setter 逐个赋值（超过 3 个字段） | MapStruct `@Mapping` |
| DTO 中写数据库查询 | DTO 只承载数据，逻辑放 Service |
