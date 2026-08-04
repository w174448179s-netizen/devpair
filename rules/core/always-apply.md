# 全局基础规范

> 此文件内容永远生效，条目控制在 5 条以内。

## 强制规则

1. **Controller 层只做参数校验和路由，不写业务逻辑。**
   业务逻辑一律下沉到 Service 层。

2. **异常统一使用 @RestControllerAdvice 全局处理。**
   禁止在 Controller 或 Service 中 try-catch 后手动包装返回值。

3. **所有 public 方法必须有 Javadoc 注释。**
   至少包含方法功能说明、@param、@return。

4. **使用 Lombok 替代手写 getter/setter/constructor。**
   实体类 @Data，Service @RequiredArgsConstructor。

5. **返回前端的数据必须用 DTO/VO，禁止直接返回 Entity。**
