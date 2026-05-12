# 01 · 项目总览与本地跑通

> 时长：约 1.5 小时
> 前置：会用 git，能跑 Docker，了解 HTTP 基本概念

## 1. 什么是 AI 网关？为什么需要它

直接把 `new-api` 一句话定义：

> **一个统一的 OpenAI 兼容 API，背后能转发到 40+ 家上游 AI 厂商，并附带用户体系、Token 管理、用量计费、限流、负载均衡和管理后台。**

### 实际场景

想象一个团队/公司要用 AI：

- 后端工程师调 OpenAI
- 算法同学想试 Claude
- 业务方又想用国内的 DeepSeek 省钱
- 财务问："这个月各部门花了多少？"

如果每个人自己拿厂商 key，会出现：
- 同一份 OpenAI key 被到处贴，泄露风险大
- 没法统一限额，谁烧爆都不知道
- 想换厂商，所有代码都要改 base_url 和参数
- 没法做计费分账

**AI 网关解决的事**：

```
   各部门/各应用
        │
        ▼  （只认一个统一的 OpenAI 协议 + 自己的虚拟 token）
   ┌─────────────┐
   │  new-api    │  ← 用户体系、token、配额、计费、限流、统计
   └─────────────┘
        │  （智能选渠道、协议转换）
   ┌────┴────┬────────┬────────┬────────┐
   ▼         ▼        ▼        ▼        ▼
 OpenAI    Claude   Gemini   DeepSeek  Bedrock ...
```

对应用方：**只需要会 OpenAI 协议**，后面接什么模型不关心。
对管理方：**只需要管 new-api 这一层**，配置渠道、定价、配额。

### 这类项目的"设计核心"

设计一个 AI 网关，本质要解决 4 类问题，全部会在后续课讲到：

1. **协议适配**：OpenAI / Claude / Gemini 的请求/响应格式都不一样，要在一个网关里统一对外为 OpenAI 协议 → **第 5 节 Relay 核心**
2. **流式处理**：AI 接口大量是 SSE 流式，转发同时要拆包统计 token → **第 6 节 流式与计费**
3. **多租户与配额**：用户/Token/分组，预扣 + 结算，避免负数 → **第 6 节 计费**
4. **多渠道调度**：同一个模型可能有多个渠道，需要负载均衡 + 故障转移 → **第 7 节 限流与分发**

记住这 4 件事，后面的代码都是为它们服务。

## 2. 项目拓扑（这是张地图，先记不住没关系）

```
┌──────────────────────────────────────────────────────────┐
│                    Client (curl / SDK)                    │
└────────────────────────┬─────────────────────────────────┘
                         │ HTTP (OpenAI 兼容协议)
                         ▼
┌──────────────────────────────────────────────────────────┐
│  Gin HTTP Server (main.go)                                │
│  ├─ middleware/   认证 · 限流 · 日志 · 分发                │
│  ├─ router/       路由分组                                 │
│  │   ├─ api-router      管理 API（前端管理后台调用）         │
│  │   ├─ relay-router    AI API（/v1/chat/completions 等）  │
│  │   └─ dashboard       统计                              │
│  ├─ controller/   请求处理（薄）                            │
│  ├─ service/      业务逻辑                                 │
│  ├─ relay/        ⭐ 网关核心：协议转换 + 上游转发          │
│  │   └─ channel/  40+ provider 适配器                     │
│  └─ model/        GORM 数据访问                            │
│                                                            │
│  下面是它依赖的基础设施 ↓                                   │
└──────────────────────────────────────────────────────────┘
                         │
        ┌────────────────┼──────────────────┐
        ▼                ▼                  ▼
   ┌─────────┐    ┌──────────────┐    ┌─────────────┐
   │ SQLite/ │    │   Redis      │    │ 上游 AI 厂商  │
   │ MySQL/  │    │  (限流/缓存/ │    │ OpenAI/Claude│
   │ PG      │    │   分布式锁) │    │ /Gemini 等   │
   └─────────┘    └──────────────┘    └─────────────┘
```

### 🟢 Node 类比：分层结构

| Node/Express 习惯 | new-api 对应 |
|---|---|
| `app.js` 入口 | `main.go` |
| `routes/*.js` | `router/*.go` |
| Express middleware | Gin middleware (`middleware/`) |
| `controllers/*.js` | `controller/*.go` |
| `services/*.js` | `service/*.go` |
| 直接用 `mysql2` 写 SQL | GORM（一个 ORM，类似 Sequelize/Prisma） |
| 单进程多回调 | **多 goroutine 并发**（第 2 节专讲） |

### 关键文件清单（先认识它们的位置，不用读）

| 路径 | 作用 |
|---|---|
| `main.go` | 启动入口：加载配置 → 连 DB/Redis → 注册路由 → 启动 Gin |
| `router/main.go` | 路由总入口，把子路由挂到 Gin |
| `router/relay-router.go` | **本项目最重要的路由**：`/v1/*` 都在这里 |
| `middleware/distributor.go` | "选哪个渠道转发"的逻辑入口 |
| `relay/relay-text.go` 等 | 不同模态的转发器（文本/图片/音频/视频） |
| `relay/channel/openai/` | 一个 provider 适配器的标准样板 |
| `model/main.go` | DB 连接 + 跨库兼容辅助 |
| `setting/` | 运行时配置（费率、模型映射、限流参数） |

## 3. 一条请求的完整生命周期（这是核心，要消化）

以 `POST /v1/chat/completions` 为例。

```
1. Client → 发起 HTTP，带 Authorization: Bearer sk-xxxx
                │
2. Gin 路由匹配  → router/relay-router.go 的 /v1/chat/completions
                │
3. 中间件链      → 依次执行：
                  - middleware/auth.go        校验 token、查用户
                  - middleware/rate-limit.go  Redis 令牌桶限流
                  - middleware/distributor.go ⭐ 选渠道：根据 model 名 + 分组负载均衡
                │
4. Controller   → controller/relay.go 的 Relay 函数
                  - 判断模式（chat/embedding/image/audio…）
                  - 调 relay 层
                │
5. Relay 核心   → relay/relay-text.go 之类
                  ┌──────────────────────────────────────┐
                  │ a) 预扣配额（按估算 token 先扣，避免负 │
                  │    数）—— service/quota.go            │
                  │ b) 协议转换：OpenAI → 上游格式         │
                  │    （比如 Claude 的 messages 结构不同）│
                  │ c) DO：发请求到上游                    │
                  │ d) 流式：SSE 边转发边累计 token        │
                  │ e) 协议转换：上游格式 → OpenAI         │
                  │ f) 结算：按实际用量重算配额，回写日志   │
                  └──────────────────────────────────────┘
                │
6. 响应         → 流式：边产生边 flush；非流式：一次性返回
                │
7. 异步落库     → 日志、用量、计费写入 DB
```

### 🟢 Node 类比

这条链路放到 Express 里大概是：

```js
app.post('/v1/chat/completions',
  auth,            // = middleware/auth.go
  rateLimit,       // = middleware/rate-limit.go
  pickChannel,     // = middleware/distributor.go
  async (req, res) => {
    await preDeductQuota(req.user, estimatedTokens)
    const upstreamBody = convertOpenAIToClaude(req.body)
    const upstreamRes = await fetch(channel.url, { body: upstreamBody })
    for await (const chunk of upstreamRes.body) {
      const openaiChunk = convertClaudeChunkToOpenAI(chunk)
      res.write(openaiChunk)              // 边转发
      tokensUsed += countTokens(chunk)    // 边统计
    }
    await settleQuota(req.user, tokensUsed)  // 结算
  }
)
```

new-api 干的就是这件事，只是规模化、协议覆盖更全、错误处理更严密。

## 4. 数据模型速览（先看名字，第 3 节细讲）

| 表 | 作用 |
|---|---|
| `users` | 用户 |
| `tokens` | 用户创建的 API key（对外那个 `sk-xxx`） |
| `channels` | 上游渠道：一个渠道 = 一个上游 key + 配置 |
| `abilities` | 渠道支持的模型映射（用来快速查"模型 X 有哪些渠道可用"） |
| `logs` | 每次调用日志 + 计费 |
| `quotas`、`redemptions` | 配额、兑换码 |
| `user_groups` | 分组（不同分组可以走不同渠道、不同费率） |

### 📘 基础补课：什么是关系型数据库

Node 里你可能只用过 MongoDB？关系型数据库（MySQL/PostgreSQL/SQLite）的核心区别：

- 数据是**表 + 行 + 列**，列是预先定义类型的（schema）
- 表与表通过**外键**关联（比如 `tokens.user_id` 指向 `users.id`）
- 用 **SQL** 查询，但通常项目里用 **ORM**（GORM）写成对象操作，ORM 帮你生成 SQL
- 支持**事务**：多个写入要么全成功要么全失败（计费、扣额必须用）

第 3 节会专门讲，现在记住"有这么个东西"即可。

### 📘 基础补课：什么是 Redis

简单理解 Redis：**内存里的超快键值数据库**，主要解决三件事：

1. **缓存**：DB 查到的热数据放进来，避免每次查表
2. **计数器/限流**：用 `INCR` 原子操作做"每分钟最多 N 次"
3. **分布式锁/协调**：多个 new-api 实例时，用 Redis 协调

第 3 节细讲，现在记住"它是用来做加速和限流的"。

## 5. 本地把项目跑起来 🛠️

按以下步骤一步步来，每步遇到问题先停下来排查再继续。

### 步骤 1：确认依赖

```bash
go version    # 需要 >= 1.22
bun --version # 任意现代版本即可
docker --version
```

如果缺 Go：[golang.org/dl](https://golang.org/dl)
如果缺 bun：`curl -fsSL https://bun.sh/install | bash`

### 步骤 2：启动后端依赖（PostgreSQL + Redis）

```bash
make dev-api
```

它做的事：用 `docker-compose.dev.yml` 起一个 **PostgreSQL 15** 和 **Redis 7** 容器（外加可选的后端容器）。

> ⚠️ 关于"默认数据库"的细节：
>
> - **项目代码本身的默认是 SQLite**（不设环境变量时落 SQLite 文件，零依赖）。判定逻辑见 `model/main.go:118` 的 `chooseDB` 函数。
> - **`docker-compose.dev.yml` 这套开发环境**显式把 `SQL_DSN` 设成了 `postgresql://...`（见 `docker-compose.dev.yml:30`），所以 `make dev-api` 起来的是 PG，不是 MySQL。
> - 项目同时支持 SQLite / MySQL >= 5.7.8 / PostgreSQL >= 9.6 三种数据库（参见根目录 `CLAUDE.md` 的 Rule 2）。
>
> 切换数据库的方式（按 `SQL_DSN` 前缀）：
>
> | `SQL_DSN` | 用的数据库 |
> |---|---|
> | 未设置 | SQLite（默认） |
> | `postgres://...` 或 `postgresql://...` | PostgreSQL |
> | 以 `local` 开头 | SQLite |
> | 其他非空（如 `user:pass@tcp(host:3306)/db`） | MySQL |

确认成功：

```bash
docker ps
# 应看到 postgres 和 redis 两个容器在运行（如果起了后端容器，还会有 new-api-dev）
```

> 想完全零依赖跑通：跳过这一步，直接进步骤 3 `go run main.go`，会自动用 SQLite。但 Redis 限流等功能会退化为内存模式。

### 步骤 3：先构建一次前端（必须）

```bash
make build-all-frontends
```

> 🚧 **为什么后端启动前必须先构建前端？**
>
> 看 `main.go:38-48`：
>
> ```go
> //go:embed web/default/dist
> var buildFS embed.FS
> //go:embed web/classic/dist
> var classicBuildFS embed.FS
> ```
>
> `//go:embed` 是 Go 1.16+ 的特性，**编译时把指定目录打包进二进制**——这样发布时只需要一个可执行文件，不用额外带前端资源。
>
> 硬性要求：**embed 指向的目录必须真实存在**，否则编译期直接报：
>
> ```
> main.go:44:12: pattern web/classic/dist: no matching files found
> ```
>
> 刚克隆下来的仓库还没构建过前端，两个 `dist/` 都不存在，所以第一次必须先跑一遍构建让 embed 有内容可打包。
>
> 一次构建之后，后续开发**不需要每次重新构建前端**——前端用 dev server 热更新（步骤 5），后端只读 embed 进去的那份"占位"前端（你不会通过后端访问前端，所以无所谓）。
> 只有当你想"模拟生产模式直接通过后端 3000 端口访问前端"时才需要重新 `make build-frontend`。

成功标志：`web/default/dist/` 和 `web/classic/dist/` 都出现 `index.html` 等产物。

### 步骤 4：启动后端

新开一个终端：

```bash
go run main.go
```

第一次会下载 Go 依赖，可能要几分钟。看到类似 `New API vX.X.X started` 即成功，默认监听 `3000`。

### 步骤 5：启动前端 dev server（热更新）

再开一个终端：

```bash
make dev-web
# 等价于：cd web/default && bun install && bun run dev
```

成功后访问浏览器给的地址（通常 `http://localhost:3001` 或类似）。

**前后端分离开发的工作流（方法 B，推荐）**：

| 终端 | 命令 | 改动后怎么办 |
|---|---|---|
| 1 | `make dev-api` | 起 PG + Redis（容器一直跑） |
| 2 | `go run main.go` | 改 Go 代码 → `Ctrl+C` 再重启 |
| 3 | `make dev-web` | 改前端 → 自动热更新，无需重启 |

前端 dev server（`:3001`）会自动把 API 请求代理到后端 `:3000`，所以浏览器只访问 `:3001` 这一个地址。后端 embed 的那份前端资源在这种模式下**不会被用到**，仅仅是为了让 Go 能通过编译。

### 步骤 6：初始化账号

打开管理后台，**首次注册的用户自动成为管理员**。注册一个，登录进去看一眼。

### 步骤 7：跑通一次调用 🛠️

1. 在后台 → **渠道** → 添加一个渠道（比如用任意 OpenAI 兼容服务，或者先填个假的练手）
2. 在后台 → **令牌** → 创建一个 token（会得到 `sk-xxxxxx`）
3. 用 curl 调一次：

```bash
curl http://localhost:3000/v1/chat/completions \
  -H "Authorization: Bearer sk-你创建的token" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [{"role": "user", "content": "hi"}]
  }'
```

即使渠道不真，你也应该看到一个**来自 new-api 自身的错误响应**（比如"渠道返回错误"），说明请求确实进了网关。

## 6. 第一次代码走读（边读边问自己问题）🛠️

以下文件按顺序打开，每个文件读 5 分钟，**不需要看懂细节**，目标是认识它的位置和职责：

1. `main.go:50` — `func main()` 入口，看启动了哪些东西
2. `router/main.go` — 总路由如何挂载子路由
3. `router/relay-router.go` — `/v1/*` 路由表
4. `middleware/distributor.go` — 找到"根据 model 选渠道"的入口（搜 `Distribute`）
5. `controller/relay.go` — 找到 `Relay` 函数
6. `relay/relay-text.go` — `TextHelper` 之类的函数，看协议转换在哪
7. `relay/channel/openai/adaptor.go` — 一个标准 adapter 长什么样（重点看 `ConvertRequest` / `DoRequest` / `DoResponse` 三个方法）

读完后试着不看代码回答以下问题（答不上来就回去翻）：

- [ ] 一条 `/v1/chat/completions` 请求经过哪几个中间件？
- [ ] "选哪个上游"是在哪一步决定的？
- [ ] 协议转换（OpenAI → Claude）的代码大致在哪个目录？
- [ ] 计费是在请求**前**扣还是**后**扣？为什么？

## 7. 本节小结

- AI 网关的核心价值：**统一协议 + 统一管控 + 多渠道调度**
- 项目分层与 Express 类比：router → middleware → controller → service/relay → model
- 一条请求的生命周期：认证 → 限流 → 选渠道 → 预扣 → 转换 → 转发 → 结算
- 你已经跑通了本地环境，并第一次走读了核心目录

## 🛠️ 本节动手任务（完成后到 process.md 打勾）

- [ ] 跑通本地后端 + 前端，能登录管理后台
- [ ] 在管理后台创建一个渠道和一个 token
- [ ] 用 curl 调一次 `/v1/chat/completions`（成功失败都行，能进网关就行）
- [ ] 按"第一次代码走读"的清单读完 7 个文件
- [ ] 在 `process.md` 写 2–3 句你对"AI 网关核心职责"的理解

## 下一节预告

[02 · Go 速成（给 Node 开发者）](./02-go-for-node-devs.md) — 不会把你教成 Go 高手，但会让你读懂这个项目的所有代码：goroutine、channel、struct/interface、错误处理、import 路径与模块。
