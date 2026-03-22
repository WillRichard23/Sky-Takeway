# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 项目概述

苍穹外卖是一个外卖平台后端服务，类似于美团或饿了么。基于 Spring Boot 构建，为管理端和用户端（微信小程序）提供服务。

**技术栈：**
- Spring Boot 2.7.3, Java 8+
- MyBatis 2.2.0（注解 + XML 映射）
- MySQL 数据库 + Druid 连接池
- JWT 身份认证 (jjwt 0.9.1)
- Redis 缓存 + Spring Cache
- 阿里云 OSS 文件存储
- 微信支付 API v3
- WebSocket 实时通信
- Knife4j 3.0.2 API 文档

## 构建和运行命令

```bash
# 构建整个项目
mvn clean install

# 跳过测试构建
mvn clean install -DskipTests

# 运行应用
cd sky-server
mvn spring-boot:run

# 或打包后运行 JAR
java -jar sky-server/target/sky-server.jar
```

**服务地址：** `http://localhost:8080`
**API 文档：** `http://localhost:8080/doc.html` (Knife4j/Swagger UI)

## 项目结构

这是一个多模块 Maven 项目，包含三个模块：

```
sky-take-out/
├── sky-common/      # 公共工具类、常量、异常、配置属性
├── sky-pojo/        # DTOs、实体类、VOs
└── sky-server/      # 主应用（控制器、服务、映射器）
```

### 模块：sky-common

共享基础设施层，包含：
- **constant/** - 消息常量、JWT 声明、状态常量、密码常量、自动填充常量
- **context/** - `BaseContext`（ThreadLocal 存储当前用户 ID）
- **enumeration/** - 枚举类，如 `OperationType`
- **exception/** - `BaseException` 基类 + 12 个具体异常
- **properties/** - 配置属性类（`JwtProperties`、`AliyunOssProperties`、`WeChatProperties`）
- **result/** - `Result<T>` 统一响应包装（code=1 成功，code=0 失败）
- **utils/** - `JwtUtil`、`AliOssUtil`、`HttpClientUtil`、`WeChatPayUtil`

### 模块：sky-pojo

按用途组织的数据模型：
- **dto/** - 请求/响应 DTO（22 个类）- 用于 API 层
- **entity/** - 数据库实体（11 个类）- 映射数据库表
- **vo/** - 视图对象（18 个类）- 用于格式化响应

核心实体：`Employee`（员工）、`User`（用户）、`Dish`（菜品）、`DishFlavor`（菜品口味）、`Category`（分类）、`Setmeal`（套餐）、`SetmealDish`（套餐菜品）、`Orders`（订单）、`OrderDetail`（订单详情）、`ShoppingCart`（购物车）、`AddressBook`（地址簿）

### 模块：sky-server

主应用，采用分层架构：
- **annotation/** - 自定义注解（`@AutoFill` 公共字段自动填充）
- **aspect/** - 切面类（`AutoFillAspect` 公共字段自动填充逻辑）
- **config/** - `WebMvcConfiguration`（JWT 拦截器、Swagger、静态资源）、`OssConfiguration`（阿里云 OSS）
- **controller/admin/** - 管理端控制器（`EmployeeController`、`CategoryController`、`DishController`、`CommonController`）
- **handler/** - `GlobalExceptionHandler`（捕获所有 `BaseException` 子类）
- **interceptor/** - `JwtTokenAdminInterceptor`（保护 `/admin/**` 除登录外的所有接口）
- **mapper/** - MyBatis 映射器（注解 + XML）
- **service/** - 业务逻辑接口 + `impl/` 中的实现类

## 架构模式

### 认证流程
1. 管理员登录：POST `/admin/employee/login` -> 返回 JWT token
2. JWT 存储在 `token` 请求头中（可通过 `admin-token-name` 配置）
3. `JwtTokenAdminInterceptor` 对所有 `/admin/**` 请求验证 token
4. 提取的 EMP_ID 存储在 `BaseContext` 的 ThreadLocal 中
5. 通过 `BaseContext.getCurrentId()` 获取当前用户上下文

### 异常处理
- 所有自定义异常继承自 `BaseException`
- `GlobalExceptionHandler` 捕获 `BaseException` 并返回 `Result<T>`
- 不要抛出通用异常 - 使用 `sky-common` 中的具体业务异常

### 请求/响应模式
- 所有管理端接口返回 `Result<T>` 统一响应
- 请求体使用 DTO，复杂响应使用 VO
- 标准分页使用 `PageResult<T>` 包装类

### MyBatis 配置
- XML 映射文件位于 `src/main/resources/mapper/*.xml`
- 实体类别名：`com.sky.entity`
- 已启用驼峰命名映射：`map-underscore-to-camel-case: true`

### 公共字段自动填充
使用 AOP 切面自动填充实体的公共审计字段：
- **注解**：`@AutoFill(OperationType.INSERT/UPDATE)` 标记需要自动填充的 Mapper 方法
- **切面**：`AutoFillAspect` 拦截带有 `@AutoFill` 注解的方法
- **自动填充字段**：
  - INSERT 操作：`createTime`、`updateTime`、`createUser`、`updateUser`
  - UPDATE 操作：`updateTime`、`updateUser`
- **用户 ID 获取**：通过 `BaseContext.getCurrentId()` 从 ThreadLocal 获取当前登录用户 ID
- **使用示例**：在 Mapper 方法上添加 `@AutoFill(OperationType.INSERT)` 即可启用自动填充

## 数据库配置

**开发数据库**（application-dev.yml）：
- 主机：localhost:3306
- 数据库：sky_take_out
- 用户名：root
- 密码：在 `application-dev.yml` 中配置

**默认管理员账号**已预置在数据库中（查看 `Employee` 表）。

## 配置属性

应用在 `application.yml` 中使用自定义属性前缀：

- `sky.jwt.*` - JWT 设置（admin-secret-key、admin-ttl、admin-token-name）
- `sky.datasource.*` - 数据库连接
- `sky.alioss.*` - 阿里云 OSS（endpoint、access-key、bucket-name）
- `sky.wechat.*` - 微信支付配置

这些属性绑定到 `sky-common/properties/` 中的 `@ConfigurationProperties` 类。

## 重要实现说明

### 密码处理
密码使用 MD5 哈希加密：
- 新员工默认密码：`123456`（定义在 `PasswordConstant.DEFAULT_PASSWORD`）
- 登录时对前端传入的明文密码使用 `DigestUtils.md5DigestAsHex()` 加密后与数据库比对
- 新增员工时密码自动加密存储
- 查询员工信息时密码会被隐藏（替换为 `****`）

### 用户上下文
始终使用 `BaseContext.getCurrentId()` 获取当前用户 ID，用于审计字段（createUser、updateUser）。拦截器会自动从 JWT 声明中设置此值。

### MyBatis 映射器选择
- 简单查询使用注解（`@Select`、`@Insert` 等）
- 复杂查询或动态 SQL 使用 XML 映射器

### 分页查询
使用 PageHelper 实现分页：
- 在 Service 方法中调用 `PageHelper.startPage(page, pageSize)` 开启分页
- Mapper 方法返回 `Page<T>` 类型
- 从 `Page` 对象获取 `total` 和 `records` 构造 `PageResult<T>` 返回

### 日志级别
Mapper 查询记录在 DEBUG 级别，Service 为 INFO，Controller 为 INFO。可在 `application.yml` 的 `logging.level` 中调整。

### 代码语言
代码库使用中文作为日志消息和注释（目标市场为中国）。请遵循此约定以保持一致性。
