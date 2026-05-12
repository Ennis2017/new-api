# 05 · Relay 核心：多 Provider 适配设计 ⭐

> 状态：⏳ 提纲（本节是整个项目的灵魂）
> 时长：约 2 小时

## 学习目标

- 理解 adapter 模式在 AI 网关里的应用
- 看懂 `Adaptor` 接口的 4 个核心方法
- 能从 0 想清楚"如何把 OpenAI 协议转成 Claude 协议"
- 看懂 `relay/relay-text.go` 主流程

## 大纲

1. **问题陈述**
   - 40+ 厂商，请求/响应格式各异
   - 流式协议都用 SSE 但 chunk 结构不同
   - 错误码风格不同
   - 如何不让代码爆炸？→ adapter 模式
2. **Adaptor 接口（看 `relay/channel/adapter.go`）**
   - `Init`：渠道初始化
   - `GetRequestURL`：上游 URL 拼接
   - `SetupRequestHeader`：鉴权头
   - `ConvertRequest`：⭐ 协议转换（OpenAI → 上游）
   - `DoRequest`：发请求
   - `DoResponse`：⭐ 协议转换（上游 → OpenAI）+ 流式处理
3. **OpenAI Adapter（标准样板）**
   - 走读 `relay/channel/openai/`
   - 它最简单：几乎是透传
4. **Claude Adapter（最复杂的对照组）**
   - 走读 `relay/channel/claude/`
   - messages 结构差异
   - system prompt 处理差异
   - tool_use 与 OpenAI function call 的映射
5. **relay 主流程**
   - `relay/relay-text.go` 串起整个流程
   - 预扣 → adapter.ConvertRequest → adapter.DoRequest → adapter.DoResponse（含流式累计）→ 结算
6. **不同模态的扩展**
   - text / image / audio / video / embedding / rerank
   - 各自的 relay-*.go

## 🛠️ 动手任务

- [ ] 找出 `relay/channel/adapter.go` 里的接口定义，列出全部方法
- [ ] 对比 `relay/channel/openai/adaptor.go` 与 `relay/channel/claude/adaptor.go` 的 `ConvertRequest`，列出至少 3 个差异
- [ ] 在本地用 curl 发一次请求，开 debug 日志，跟到 `DoResponse` 看实际数据

---

> 本节是本项目最核心的设计模式课，内容待填充。
