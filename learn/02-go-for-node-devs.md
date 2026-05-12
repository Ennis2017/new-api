# 02 · Go 速成（给 Node 开发者）

> 状态：⏳ 提纲（待填充完整内容）
> 时长：约 1.5 小时

## 学习目标

学完本节你能：**读懂 new-api 里 90% 的 Go 代码**，知道每个语法长什么样、对应 Node 里的什么概念。
不要求会写复杂的 Go，只要求"能读懂、能改"。

## 大纲

1. **Go module 与包**
   - `go.mod` 类比 `package.json`
   - import 路径 = GitHub 路径，没有 node_modules
   - 包级可见性规则（首字母大写 = 导出）
2. **基本类型与 struct**
   - `struct` 类比 TS interface + class
   - tag：`json:"name"` 类比 zod schema
3. **函数 / 多返回值 / 错误处理**
   - `func name(args) (T, error)` 模式
   - 没有 try/catch，所有错误显式 `if err != nil`
4. **指针入门**
   - `*T` 与 `&v`，什么时候用指针什么时候用值
   - 关联到项目 Rule 6（DTO 指针类型保留显式零值）
5. **interface 与隐式实现**
   - 没有 `implements` 关键字，鸭子类型
   - 对应项目里的 `Adaptor` 接口（每个 channel 实现它）
6. **goroutine 与 channel**
   - `go fn()` vs Node `setImmediate`
   - 并发模型差异：Node 单线程事件循环 vs Go M:N 调度
   - channel 类比 Node 的 EventEmitter / async iterator
7. **Context**
   - `context.Context` 传递取消信号、超时、请求级值
   - Gin 的 `*gin.Context` 包了一层
8. **常用包速览**
   - `net/http`、`fmt`、`encoding/json`（本项目用 `common.Marshal` 包了一层）
   - 第三方：gin、gorm、go-redis

## 🛠️ 动手任务

- [ ] 在 `relay/channel/openai/adaptor.go` 里找出 `Adaptor` struct 和它实现的接口方法
- [ ] 写一个 5 行 Go demo：`func divide(a, b int) (int, error)`，b=0 时返回 error
- [ ] 解释 `common/json.go` 里 `Marshal` 为什么要包一层（提示：和 Rule 1 相关）

---

> 本节内容待填充。学习时让 Claude 读 `process.md`，告诉它"继续第 2 节"，它会基于本提纲展开。
