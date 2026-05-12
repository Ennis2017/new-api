# 04 · 认证、中间件与请求生命周期

> 状态：⏳ 提纲
> 时长：约 1.5 小时

## 学习目标

- 理解多种认证方式（JWT / OAuth / WebAuthn）的取舍
- 看懂 Gin 的中间件链
- 能跟踪一条请求从 HTTP 接入到 controller 的每一步

## 大纲

1. **认证体系总览**
   - Session（管理后台）vs Bearer Token（API 调用）
   - JWT 基础概念
   - OAuth 第三方登录流程
   - WebAuthn/Passkey 是什么、为什么加
2. **Gin 中间件机制**
   - 类比 Express middleware
   - `c.Next()` 与 `c.Abort()`
   - 上下文传值 `c.Set / c.Get`
3. **关键中间件走读**
   - `middleware/auth.go` — token 校验、用户加载
   - `middleware/cors.go`
   - `middleware/rate-limit.go`
   - `middleware/distributor.go` — ⭐ 选渠道
   - `middleware/request-id.go` / 日志
4. **请求生命周期完整图谱**
   - 在第 1 节基础上加细节
5. **错误处理与统一响应**
   - `common.AbortWithError` 类工具
   - 错误码体系

## 🛠️ 动手任务

- [ ] 在 `middleware/auth.go` 找到从请求里取 token 的代码，写出取的优先级（header / query / cookie 哪个先）
- [ ] 阻断式排查：故意发一个没 Authorization 的请求，看错误是从哪个中间件抛出来的
- [ ] 画一张你自己的"请求生命周期"图（手画即可）

---

> 本节内容待填充。
