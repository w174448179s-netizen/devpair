# 全局基础规范

> 每个 task 必加载。5 条铁律，不可违反。

## 强制规则

1. **Controller 只做参数校验和路由，不写业务逻辑。**
   业务逻辑一律下沉到 Service 层。

2. **异常统一使用 @RestControllerAdvice 全局处理。**
   禁止在 Controller 或 Service 中 try-catch 后手动包装返回值。

3. **所有 public 方法必须有 Javadoc 注释，类必须有类注释。**
   Javadoc 至少包含功能说明、@param、@return。类注释包含 @author、@since。

4. **使用 Lombok，禁止 @Autowired 字段注入。**
   实体类 @Data，Service @RequiredArgsConstructor 构造函数注入。

5. **返回前端必须用 DTO/VO，禁止直接返回 Entity/DO。**
   DO 禁止跨 Service 传递，必须先转 DTO。
