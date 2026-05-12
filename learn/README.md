# new-api 学习路径 · AI 网关设计

> 面向已熟练 React/TS、有 Node 后端基础、想系统学习 AI 网关设计思路的学习者。
> 颗粒度：每节 1–2 小时深度学习，含代码走读 + 概念讲解 + 动手任务。

## 学习者画像（用来指导讲解风格）

- 后端经验：写过 Node，**没用过 Redis 与关系型数据库，Node 也不算熟练**
- 前端经验：熟练 React + TypeScript
- 目的：**学 AI 网关的整体设计思路**（架构、relay、计费、限流、多渠道）
- 偏好：大而深，类比 Node 讲 Go，**不重复讲前端**

## 路径总览（8 节）

| # | 主题 | 重点 | 状态 |
|---|---|---|---|
| 01 | [项目总览与本地跑通](./01-overview.md) | 什么是 AI 网关 · 请求全链路 · 本地启动 | ✅ 已发布 |
| 02 | [Go 速成（给 Node 开发者）](./02-go-for-node-devs.md) | goroutine vs 事件循环 · package/struct/interface · Gin 类比 Express | ⏳ 提纲 |
| 03 | [数据层：GORM + 多数据库 + Redis](./03-data-layer.md) | 关系型 DB 基础 · GORM 模式 · Redis 作用 · 项目里的核心表 | ⏳ 提纲 |
| 04 | [认证、中间件与请求生命周期](./04-auth-middleware.md) | JWT/OAuth/Passkey · Gin 中间件链 · 一条请求如何打到 controller | ⏳ 提纲 |
| 05 | [Relay 核心：多 Provider 适配设计](./05-relay-core.md) | adapter 模式 · convert→do→parse 三段式 · 40+ 渠道如何统一 | ⏳ 提纲 |
| 06 | [流式响应与 Token 计费](./06-streaming-billing.md) | SSE 流式转发 · 用量统计 · billingexpr 表达式引擎 | ⏳ 提纲 |
| 07 | [限流、分发与渠道管理](./07-ratelimit-distribution.md) | Redis 令牌桶 · 渠道负载均衡 · 故障转移与冷却 | ⏳ 提纲 |
| 08 | [二次开发实战：新增一个 Channel](./08-add-new-channel.md) | 从 0 到 1 接入一个新 provider | ⏳ 提纲 |

## 怎么用这套教程

1. **按顺序学**：每节会假设你已掌握前面的概念。
2. **看进度文件 [`process.md`](./process.md)**：记录你当前学到哪、做完了什么任务、卡在哪里。
3. **每节末尾有"动手任务"**：完成任务后在 `process.md` 打勾，再继续下一节。
4. **新开窗口续学**：让 Claude 读 `process.md` 即可衔接（已通过项目 `CLAUDE.md` 配置）。

## 约定

- 代码定位用 `file:line` 格式，可在 IDE 直接跳转。
- 涉及 Go 概念时会写 **🟢 Node 类比** 帮助理解。
- 涉及 Redis/DB 基础概念时会写 **📘 基础补课**。
- 动手任务标记 **🛠️**，需要你实际操作。
