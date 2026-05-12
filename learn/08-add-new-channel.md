# 08 · 二次开发实战：新增一个 Channel

> 状态：⏳ 提纲（毕业项目）
> 时长：约 2 小时

## 学习目标

- 综合运用前 7 节所学，从 0 接入一个新的上游
- 走完一个完整的二次开发流程：constant → relay/channel/xxx → 配置项 → 前端选项

## 大纲

1. **选一个目标 provider**
   - 推荐：找一个免费可注册的 OpenAI 兼容 endpoint（或自己 mock）
2. **第 1 步：常量与渠道类型**
   - 在 `constant/` 注册 channel type
3. **第 2 步：创建 adapter 目录**
   - `relay/channel/<yourname>/adaptor.go`
   - 实现 `Adaptor` 接口的所有方法
4. **第 3 步：协议转换（如果不是 OpenAI 兼容）**
   - `ConvertRequest` / `DoResponse`
5. **第 4 步：StreamOptions 支持**（项目 Rule 4）
   - 加到 `streamSupportedChannels`（如果支持）
6. **第 5 步：注册到分发**
   - `relay/channel/<yourname>` 被 distributor 识别
7. **第 6 步：前端渠道类型选项**
   - 在 web/default 里加渠道类型下拉
8. **第 7 步：本地端到端测试**
   - 后台建渠道 → 创建 token → curl 调通

## 🛠️ 动手任务

- [ ] 完成上述 7 步，本地能用新渠道发请求
- [ ] 写一份"我接入 X 渠道的踩坑记录"放到 process.md

## 毕业

走完本节意味着你已经具备：
- 读懂 new-api 任意模块
- 独立接入新的 AI provider
- 理解 AI 网关的核心设计权衡

---

> 本节内容待填充。
