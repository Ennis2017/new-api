# 03 · 数据层：GORM + 多数据库 + Redis

> 状态：⏳ 提纲
> 时长：约 2 小时

## 学习目标

- 理解关系型数据库的核心概念（表/索引/事务/外键）
- 看懂 GORM 在项目里的标准用法
- 理解 Redis 在本项目的 3 大用途
- 看懂 `model/` 目录下任意一个模型文件

## 大纲

1. **关系型数据库速成**（📘 基础补课）
   - schema / 行 / 列 / 索引 / 事务的意义
   - 为什么 AI 网关用关系型 DB（强一致：计费不能丢）
2. **三库兼容是怎么做到的（项目 Rule 2）**
   - GORM 抽象掉 90%
   - 剩下 10% 用 `commonGroupCol` 之类变量切换
   - 看 `model/main.go` 的初始化流程
3. **核心表设计走读**
   - `users` / `tokens` / `channels` / `abilities` / `logs`
   - 关键索引为什么这么建
4. **GORM 常见模式**
   - 模型定义 + tag
   - `Find / First / Where / Updates / Transaction`
   - 钩子 `BeforeCreate` 等
5. **Redis 在项目里的 3 大用途**
   - 缓存（用户 / token 信息）— 减少 DB 查询
   - 限流（令牌桶 / 计数器）— `middleware/rate-limit.go`
   - 分布式协调（多实例部署时）
6. **内存缓存 + Redis 双层缓存策略**
   - 看 `common/` 里相关代码

## 🛠️ 动手任务

- [ ] 启动后 `docker exec -it <mysql容器> mysql -uroot -p` 进去看表结构
- [ ] 在 `model/user.go` 找到 `User` struct，对应数据库哪些列？
- [ ] 用 `redis-cli` 连进 Redis，启动后端发起几次请求，观察出现了哪些 key

---

> 本节内容待填充。
