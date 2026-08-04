# CRUD 功能开发模板

> 用于生成标准的增删改查功能 task。AI 拆任务时基于此模板填充项目专属内容。

## 标准开发步骤

```
1. 数据模型 → DO + Mapper XML
2. 对象定义 → DTO + Req + VO + Convert
3. Service 接口 + 实现
4. Controller（参数校验 + 路由）
5. 单元测试（Service 层）
6. 集成验证（接口联调）
```

## 需加载的规则文件

- `~/devpair/rules/core/always-apply.md` — 5 条铁律
- `~/devpair/rules/java/coding-rules.md` — 编码规范
- `~/devpair/rules/java/object-convert-rules.md` — MapStruct 转换规范
- `~/devpair/rules/testing/testing-rules.md` — 测试规范

## 交付物清单（参考）

```
src/main/java/com/example/{module}/
├── controller/
│   └── {Entity}Controller.java          # REST 接口
├── service/
│   ├── {Entity}Service.java             # 接口
│   └── impl/
│       └── {Entity}ServiceImpl.java     # 实现
├── dto/
│   ├── {Entity}DTO.java                 # Service 间传递
│   ├── req/
│   │   ├── Create{Entity}Req.java       # 新增入参
│   │   └── Update{Entity}Req.java       # 修改入参
│   └── vo/
│       └── {Entity}VO.java              # 返回前端
├── convert/
│   └── {Entity}Convert.java             # MapStruct 转换器
└── repository/
    ├── {Entity}.java                    # DO
    └── {Entity}Mapper.java              # MyBatis Mapper

src/main/resources/mapper/
└── {Entity}Mapper.xml                   # SQL 映射

src/test/java/com/example/{module}/
└── service/{Entity}ServiceImplTest.java # 单元测试
```

## 验收检查清单

- [ ] Controller 只做参数校验和路由，无业务逻辑
- [ ] 异常由全局处理器统一处理，Controller/Service 无 try-catch 包装返回值
- [ ] 所有 public 方法有 Javadoc（功能说明、@param、@return）
- [ ] 类注释包含 @author、@since
- [ ] Service 用 @RequiredArgsConstructor 注入，依赖声明为 private final
- [ ] 返回 VO，不直接返回 DO/Entity
- [ ] DO 不跨 Service 传递，先转 DTO
- [ ] 对象转换用 MapStruct，无 BeanUtils.copyProperties()
- [ ] 状态字段用枚举，无魔法字符串
- [ ] 单元测试覆盖正常路径 + 异常路径
- [ ] 无 unused import、通配符 import

## task 文档示例

```markdown
# Task XX：{实体名} CRUD 功能

## 所属模块
{模块名称}

## 前置依赖
Task XX（数据库表已建好）/ 无

## 任务目标
实现 {实体名} 的增删改查接口，包含：
- 列表查询（分页）
- 详情查询
- 新增
- 修改
- 删除（逻辑删除）

## 需加载的规则文件（执行前必须逐一读取）
- ~/devpair/rules/core/always-apply.md
- ~/devpair/rules/java/coding-rules.md
- ~/devpair/rules/java/object-convert-rules.md
- ~/devpair/rules/testing/testing-rules.md

## 交付物清单
{按上方交付物清单模板填充具体类名}

## 验收标准
1. 功能满足任务目标描述
2. 代码严格遵守【需加载的规则文件】内全部约束
3. 处理边界异常：空数据、重复创建、删除不存在记录
4. 代码注释完整，符合编码规范
5. 单元测试覆盖核心逻辑（正常路径 + 异常路径）

## 额外说明
{项目特殊调整，没有则删除}
```
