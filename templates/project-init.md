# 项目初始化模板

> 用于每个新项目的第一个 task。搭建项目骨架，后续所有 task 依赖此基础。

## 标准开发步骤

```
1. Maven/Gradle 项目骨架 + 依赖
2. application.yml 配置（数据库、日志、端口）
3. 全局异常处理（@RestControllerAdvice + 自定义异常）
4. 统一响应格式（Result<T>）
5. MyBatis 配置（数据源、连接池、Mapper 扫描）
6. 健康检查接口（/health）
```

## 需加载的规则文件

- `~/devpair/rules/core/always-apply.md` — 5 条铁律
- `~/devpair/rules/java/coding-rules.md` — 编码规范

## 交付物清单（参考）

```
src/main/java/com/example/{project}/
├── config/
│   ├── MyBatisConfig.java               # MyBatis 配置
│   └── WebConfig.java                   # Web 配置（CORS 等）
├── common/
│   ├── result/
│   │   ├── Result.java                  # 统一响应
│   │   └── ResultCode.java             # 状态码枚举
│   ├── exception/
│   │   ├── BusinessException.java       # 业务异常
│   │   └── GlobalExceptionHandler.java  # 全局异常处理
│   └── controller/
│       └── HealthController.java        # 健康检查

src/main/resources/
├── application.yml                      # 主配置
└── mapper/                              # Mapper XML 目录

pom.xml / build.gradle                   # 依赖管理
```

## 验收检查清单

- [ ] 全局异常处理器覆盖：业务异常、参数校验异常、未知异常
- [ ] 统一响应格式 Result<T>，包含 code、message、data
- [ ] application.yml 有数据库密码配置（即使为空占位）
- [ ] 日志级别为 INFO，不是 DEBUG
- [ ] CORS 限制来源，未用 allowAllOrigins
- [ ] 数据库连接池有超时配置
- [ ] 健康检查接口可访问
- [ ] 项目能正常启动

## task 文档示例

```markdown
# Task 01：项目初始化

## 所属模块
基础架构

## 前置依赖
无

## 任务目标
搭建 {项目名} 项目骨架，包含：
- Spring Boot 3 + Java 17 项目结构
- 全局异常处理 + 统一响应格式
- MyBatis + PostgreSQL 数据源配置
- 健康检查接口

## 需加载的规则文件（执行前必须逐一读取）
- ~/devpair/rules/core/always-apply.md
- ~/devpair/rules/java/coding-rules.md

## 交付物清单
{按上方交付物清单模板填充具体类名}

## 验收标准
1. 项目能正常启动，/health 接口返回 200
2. 代码严格遵守【需加载的规则文件】内全部约束
3. 全局异常处理覆盖业务异常、参数校验异常、未知异常
4. 数据库连接池有超时配置
5. 日志级别为 INFO

## 额外说明
{项目特殊配置，如特定数据库版本、端口等}
```
