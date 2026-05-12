# 06 · 流式响应与 Token 计费

> 状态：⏳ 提纲
> 时长：约 2 小时

## 学习目标

- 理解 SSE 流式响应在 Go 里如何实现
- 看懂"边转发边统计 token"的代码
- 理解预扣 + 结算的计费模型
- 看懂 `pkg/billingexpr/` 表达式计费引擎的设计

## 大纲

1. **SSE 协议速成**
   - `text/event-stream`、`data: {...}\n\n` 格式
   - 与 WebSocket 的区别
2. **Go 里如何写 SSE**
   - `http.Flusher` 接口
   - Gin 的 `c.Stream`
3. **流式转发 + 实时统计**
   - 读上游 chunk → 累计 token → 转换格式 → 写回客户端
   - 关键代码位置
4. **Token 计数**
   - tiktoken / 估算 / 上游返回三种来源
   - 项目里的策略选择
5. **预扣 vs 结算**
   - 为什么必须预扣（流式调用结束才知道实际用量，避免负数）
   - 估算函数与结算函数
6. **比例与定价系统**
   - `setting/ratio_setting/` 配置
   - 模型倍率、补全倍率、分组倍率
7. **billingexpr 表达式引擎**
   - **必读 `pkg/billingexpr/expr.md`**（项目 Rule 7）
   - 为什么需要表达式（阶梯计费、动态定价）
   - 变量 / 函数 / 编辑器 / 存储 / 预扣 / 结算 / 日志展示全链路

## 🛠️ 动手任务

- [ ] 通读 `pkg/billingexpr/expr.md`
- [ ] 找出"预扣配额"的函数（提示：在 service/ 或 relay/ 下，搜 preConsume）
- [ ] 写一个简单的计费表达式，能跑通

---

> 本节内容待填充。
