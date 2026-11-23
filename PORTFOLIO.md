# 技术社区后端二次开发说明

本仓库 Fork 自开源项目 [itwanger/paicoding](https://github.com/itwanger/paicoding)，用于学习 Spring Boot
社区系统的业务分层、鉴权、缓存、异步任务与工程设计。原项目的完整功能、作者信息和使用方式以
[README.md](README.md) 及原仓库为准。

本文档只整理本副本中可定位、可运行、可验证的重点模块，不把上游已有能力表述为从零实现。个人二次开发与排查重点如下。

## 1. 登录会话与强制下线

- 使用 JWT 完成签名与过期校验，并在 Redis 中维护 `token -> userId` 映射，解决无状态 JWT 无法主动失效的问题。
- 维护用户会话索引，支持登出即失效、按用户批量撤销会话与强制下线。
- 记录登录失败、登出和踢人事件，为后续风控与会话排查保留审计信息。

关键代码：

- [`UserSessionHelper.java`](paicoding-service/src/main/java/com/github/paicoding/forum/service/user/service/help/UserSessionHelper.java)
- [`LoginServiceImpl.java`](paicoding-service/src/main/java/com/github/paicoding/forum/service/user/service/user/LoginServiceImpl.java)
- [`UserSessionRevocationTest.java`](paicoding-service/src/test/java/com/github/paicoding/forum/service/user/service/help/UserSessionRevocationTest.java)

## 2. Redis 活跃排行榜

- 使用 Redis ZSet 保存日榜和月榜，按分数完成 TopN 排名查询。
- 使用 Hash 记录用户与行为组合，避免同一行为被重复加分。
- 按业务周期设置缓存 TTL，控制历史榜单数据占用。
- 围绕点赞、收藏、关注、评论和发文事件更新活跃分。

关键代码：

- [`UserActivityRankServiceImpl.java`](paicoding-service/src/main/java/com/github/paicoding/forum/service/rank/service/impl/UserActivityRankServiceImpl.java)
- [`UserActivityListener.java`](paicoding-service/src/main/java/com/github/paicoding/forum/service/rank/service/listener/UserActivityListener.java)
- [`RankController.java`](paicoding-web/src/main/java/com/github/paicoding/forum/web/front/rank/RankController.java)

## 3. 异步执行与超时兜底

- 通过自定义 `@AsyncExecute` 注解和 AOP 统一异步任务入口。
- 将任务提交与返回结果处理解耦，在规定时间内返回兜底结果。
- 在请求到异步线程的切换过程中显式传递用户和请求上下文，避免数据串号。

关键代码：

- [`AsyncExecute.java`](paicoding-core/src/main/java/com/github/paicoding/forum/core/async/AsyncExecute.java)
- [`AsyncExecuteAspect.java`](paicoding-core/src/main/java/com/github/paicoding/forum/core/async/AsyncExecuteAspect.java)
- [`AsyncUtil.java`](paicoding-core/src/main/java/com/github/paicoding/forum/core/async/AsyncUtil.java)

## 4. 缓存与稳定性

- 使用 Redis 与 Guava 组合本地缓存和分布式缓存，热点数据优先命中本地。
- 为本地缓存设置容量上限，避免无界增长导致内存溢出。
- 对会话元数据、大对象读取和异常响应做边界处理，提升故障可定位性。

关键代码：

- [`RedisClient.java`](paicoding-core/src/main/java/com/github/paicoding/forum/core/cache/RedisClient.java)
- [`GlobalViewInterceptor.java`](paicoding-web/src/main/java/com/github/paicoding/forum/web/hook/interceptor/GlobalViewInterceptor.java)

## 技术栈

- Java 8、Spring Boot、Spring MVC、MyBatis-Plus
- MySQL、Redis、RabbitMQ、Elasticsearch
- Docker、Maven、Liquibase

## 本地运行

环境变量模板见 [`.env.example`](.env.example)。首次运行请复制为 `.env` 并填写个人配置：

```bash
cp .env.example .env
```

构建与启动步骤、依赖服务和环境切换方式见原项目 [README.md](README.md)。

## 验证建议

使用项目约定的 Java 8 和 Maven 版本构建：

```bash
export JAVA8_HOME=/path/to/your/jdk8
JAVA_HOME="$JAVA8_HOME" PATH="$JAVA8_HOME/bin:$PATH" \
  mvn -pl paicoding-service -Dtest=UserSessionRevocationTest test
```

进一步可以补充：

- 登录、登出、踢人链路的接口集成测试
- 活跃排行榜重复加分与 TTL 行为测试
- 异步任务的超时、异常和上下文传递测试
- 50 并发、60 秒压测下的 QPS、P95 和错误率

## 边界说明

- 本仓库是个人 Fork，不替代原项目或原作者。
- 文章、评论、前端、搜索、消息队列等能力来自 paicoding 原有工程。
- 本文档记录的是个人二次开发与工程分析范围；面试展示时应能说明具体改动文件、排查过程和验证结果。
