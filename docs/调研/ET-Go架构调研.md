# ET-Go 架构调研

调研日期：2026-07-14

## 调研范围

本文调研如何结合当前 ET10 项目的设计思想，实现一个 Go 版本的在线游戏服务端框架。目标不是逐文件、逐语法翻译现有 C# 代码，而是识别 ET 中最有价值的架构约束，并使用 Go 的语言特性重新实现。

本文重点回答以下五类问题：

1. 项目定位：Go 版应该解决什么问题，不应该覆盖什么范围。
2. 核心运行时：Entity、System、Fiber、异步和定时器如何映射到 Go。
3. 网络与分布式：消息、RPC、Actor、Location 和服务发现如何设计。
4. 工程化：协议、配置、代码生成、静态分析、测试和热更新如何处理。
5. 实施策略：第一阶段做什么、主要风险是什么、如何判断原型成功。

本文基于当前仓库的静态代码阅读和 Go 官方资料调研形成。未修改运行时代码，未执行 Unity、编译或测试。

## 核心结论

可以实现一个具有 ET 思想的 Go 版本，但推荐将其定位为：

> Go MMO 服务端框架 + Go Robot/Test，Unity 客户端继续使用 C#。

Go 版应保留 ET 的 Entity/System 分离、Fiber 单线程逻辑域、Actor 消息通信、Location 路由、协议生成、包依赖约束和自动化验证；不应机械复制 C# 的 `async/await`、DLL 热更新、运行时反射和对象池实现。

推荐暂定项目名为 `ET-Go`，在当前仓库中先以原型方式验证。原型成熟后，再决定是否拆成独立仓库。

---

## 一、项目定位类问题

### 问题 1：这是一次代码翻译，还是一次架构重写？

结论：应当是架构重写，不是代码翻译。

当前项目的核心能力并不只来自 C# 语法，而是由以下部分共同组成：

- Unity 客户端和 .NET 服务端共享代码。
- Entity 生命周期和 Entity/System 分离。
- Fiber、同步上下文和 `ETTask`。
- Actor、RPC、Location 和 ServiceDiscovery。
- MemoryPack、Proto、Luban 和代码生成器。
- Roslyn Analyzer 对架构规则的强制检查。
- HybridCLR、机器人测试和 UnityBridge。

如果逐文件翻译，会把大量 C#/.NET 特有实现生硬搬到 Go，最后得到既不符合 Go 使用习惯、也无法保持 ET 原有优势的代码。

Go 版应该保留行为契约和架构不变量，重新设计具体实现。

### 问题 2：Go 版应该包含客户端吗？

结论：第一阶段不包含正式游戏客户端。

当前客户端使用 Unity 2022.3，约有大量 C# 文件直接依赖 Unity API。Go 无法直接替代 Unity 的 C# 脚本体系。如果连客户端一起迁移，就需要更换游戏引擎或自研客户端运行时，范围会从“实现 Go 版 ET”扩大成“重新开发完整游戏引擎和框架”。

推荐范围：

- Unity 正式客户端：继续使用 C#。
- MMO 服务端：使用 Go。
- Robot 和协议测试客户端：使用 Go。
- Web GM、运营后台：可以使用 TypeScript。
- AI、数据分析和构建辅助：可以使用 Python。

### 问题 3：Go 版最应该保留 ET 的哪些思想？

结论：优先保留八个架构不变量。

1. Entity 负责状态，System 负责逻辑。
2. 一个逻辑域只有一个状态所有者。
3. 跨逻辑域只能通过消息通信。
4. Entity 引用必须能够识别对象已销毁或已复用。
5. Actor 的物理位置对业务调用方透明。
6. 协议、Handler 和 System 注册尽量由代码生成完成。
7. 模块依赖必须单向，并由工具强制检查。
8. 测试能够在单进程中模拟多服务和多客户端。

当前代码依据包括：

- `Packages/cn.etetet.core/Scripts/Core/Share/Entity/Entity.cs`
- `Packages/cn.etetet.core/Scripts/Core/Share/Entity/EntityRef.cs`
- `Packages/cn.etetet.core/Scripts/Core/Share/World/Fiber/Fiber.cs`
- `Packages/cn.etetet.core/Scripts/Model/Share/Actor/MessageDispatcher.cs`
- `Packages/cn.etetet.core/Scripts/Hotfix/Share/Actor/ProcessInnerSenderSystem.cs`
- `Packages/cn.etetet.core/Scripts/Core/Share/Timer/TimerComponentSystem.cs`

---

## 二、核心运行时类问题

### 问题 4：Go 中如何表示 Entity？

结论：Entity 仍然是树状数据节点，但不照搬 C# 继承体系。

建议每个 Entity 至少包含：

- 业务 ID。
- 运行期 InstanceID。
- Generation，用于识别已销毁或复用对象。
- Parent。
- Children。
- Components。
- 所属 Scene/Fiber 标识。
- 生命周期状态。

Go 不适合构建很深的类继承结构，因此建议使用组合、接口和生成代码。

组件保持纯数据，例如：

```go
type Transform struct {
    X float32
    Y float32
}

type MoveComponent struct {
    TargetX float32
    TargetY float32
    Speed   float32
}
```

逻辑使用 System 函数承载：

```go
func MoveUpdate(ctx *et.Context, unit et.Ref[Unit], delta time.Duration) {
    entity, ok := et.Resolve(ctx, unit)
    if !ok {
        return
    }

    move, ok := et.Get[MoveComponent](entity)
    if !ok {
        return
    }

    transform := et.MustGet[Transform](entity)
    _ = move
    _ = transform
}
```

### 问题 5：Go 中如何替代 `EntityRef<T>`？

结论：使用不长期保存裸指针的代际引用。

建议结构：

```go
type Ref[T any] struct {
    ID         uint64
    Generation uint32
}
```

解析引用时必须同时满足：

1. Entity 仍存在。
2. ID 相同。
3. Generation 相同。
4. 当前代码正在 Entity 所属 Fiber 中执行。

当 Entity 被销毁或对象槽位被复用时，Generation 变化，旧引用自动失效。这对应现有 `EntityRef<T>` 使用 `InstanceId` 检查生命周期的设计。

### 问题 6：Go 中 Fiber 应该是什么？

结论：一个 Fiber 对应一个状态所有者 goroutine，而不是一个 Entity 对应一个 goroutine。

Fiber 建议拥有：

- 一个输入 mailbox。
- 一棵 Entity 树。
- Entity registry。
- Timer heap。
- RPC pending 表。
- Handler/System registry。
- Scene、Zone 和服务类型信息。
- 停止和取消上下文。

所有 Entity 状态只能由所属 Fiber goroutine 修改。网络线程、数据库协程和其他 Fiber 不得直接持有并修改 Entity 指针，只能发送消息或异步结果。

Go 官方内存模型明确要求并发访问通过 channel、锁或原子操作进行同步；单 goroutine 持有 Fiber 状态与 ET 的单线程逻辑域高度一致：

- [Go Memory Model](https://go.dev/ref/mem)
- [Share Memory By Communicating](https://go.dev/doc/codewalk/sharemem/)

### 问题 7：为什么不能为每个 Entity 创建 goroutine？

结论：Entity 和并发执行单元不是同一个概念。

一个地图可能包含大量 Unit、Buff、Item、Quest、AOI Cell 和行为树节点。如果每个 Entity 都创建 goroutine 和 channel，会带来：

- 大量 mailbox 和调度开销。
- Entity 之间交互全部异步化。
- 很难实现确定性的帧更新顺序。
- 生命周期和背压管理更加复杂。
- 很容易出现 goroutine 泄漏。

建议一个 Fiber 管理一组具有强一致性关系的 Entity，例如：

- 一个地图实例。
- 一个战斗房间。
- 一个 Gate 分片。
- 一个 Location 分片。
- 一个独立后台服务场景。

### 问题 8：Go 中如何替代 `ETTask` 和 `await`？

结论：不能在 Fiber 内直接阻塞等待，应使用 Future + continuation 回投 Fiber。

错误实现是：

1. Fiber Handler 发出 RPC。
2. Handler 在 Fiber goroutine 内等待 response channel。
3. Fiber 停止处理 mailbox。
4. 同 Fiber 响应或环形 RPC 无法继续执行，形成死锁。

推荐模型：

```go
ref := et.RefOf[Unit](unit)

et.Await(ctx, ctx.Call(target, request), func(
    ctx *et.Context,
    response *pb.Response,
    err error,
) {
    unit, ok := et.Resolve(ctx, ref)
    if !ok {
        return
    }

    // continuation 已重新进入原 Fiber，可以安全访问 unit。
})
```

执行流程：

1. 发出 RPC 并注册 pending future。
2. 当前 Handler 返回，Fiber 继续处理其他消息。
3. RPC 返回后，由 RPC Hub 完成 future。
4. continuation 被包装成内部消息并投递到原 Fiber。
5. continuation 在原 Fiber 中重新解析 Entity Ref。

取消和超时使用 `context.Context` 表达，不额外复制一套 CancellationToken。

### 问题 9：Go 中如何实现定时器？

结论：每个 Fiber 持有自己的定时器容器，第一版使用最小堆。

不建议为每个业务定时器创建一个长期 goroutine 或 `time.Ticker`。建议 Fiber 维护：

- 最小堆，按触发时间排序。
- TimerID 到 TimerAction 的索引。
- 一次性、重复、等待型定时器。
- 虚拟时钟接口，方便测试。

第一版使用标准库 `container/heap` 即可。只有基准测试证明定时器数量成为瓶颈后，才升级为分层时间轮。

### 问题 10：是否需要对象池？

结论：第一版不池化 Entity，只池化高频临时对象。

建议第一版仅对以下对象考虑 `sync.Pool`：

- 网络读写缓冲区。
- 消息 Envelope。
- 编解码临时缓冲区。
- 高频短生命周期的内部事件。

Entity 先使用普通分配和 GC。等性能基准确认 Entity 分配是主要瓶颈后，再引入对象池。这样可以减少早期生命周期错误，也能降低 `Ref.Generation` 实现复杂度。

---

## 三、网络与分布式类问题

### 问题 11：消息如何进入 Fiber？

结论：网络 IO 与业务状态完全隔离。

推荐链路：

```text
Socket Reader
    -> Frame Decoder
    -> Message Codec
    -> Router
    -> Fiber Mailbox
    -> Handler
    -> Entity/System
```

网络 goroutine 只负责：

- 读取数据。
- 处理粘包和分帧。
- 解码消息。
- 构造 Envelope。
- 根据 ActorID/FiberID 路由。

网络 goroutine 绝对不能直接调用业务 Entity/System。

### 问题 12：RPC 如何设计？

结论：保留 ET 的 RpcID、pending callback、超时和错误传播模型。

建议 Envelope 至少包含：

- Opcode。
- RpcID。
- Source ActorID。
- Target ActorID。
- TraceID。
- ErrorCode。
- Payload。

每个 Fiber 或 Node 维护 pending RPC 表。RPC 完成、超时或取消时只能成功结束一次，并清理 pending 记录。

必须覆盖以下测试：

- 正常返回。
- 超时。
- 主动取消。
- 目标 Actor 不存在。
- 重复 response。
- response 晚于 timeout。
- Fiber 销毁时批量取消 pending RPC。

### 问题 13：ActorID 如何设计？

结论：ActorID 同时表达物理位置和实例代际。

建议包含：

```text
NodeID + FiberID + InstanceID + Generation
```

其中：

- NodeID：进程或节点地址。
- FiberID：节点内逻辑执行域。
- InstanceID：Fiber 内 Entity 实例。
- Generation：防止 Entity 槽位复用后旧消息命中新对象。

同进程消息直接进入目标 Fiber mailbox；跨进程消息通过 Transport 编码发送。

### 问题 14：Location 服务如何设计？

结论：继续保留逻辑 ID 到 ActorID 的间接寻址层。

Location 记录建议包含：

- LocationType。
- EntityKey。
- 当前 ActorID。
- Version/Epoch。
- LockToken。
- Lease/ExpireAt。

必须支持：

- Add、Get、Remove。
- Lock、Unlock。
- 迁移期间防止旧节点覆盖新节点。
- 幂等重试。
- 缓存失效和主节点切换。

第一阶段可以实现单节点内存版；第二阶段再加入持久化、复制和故障恢复。

### 问题 15：网络协议选择什么？

结论：新协议优先使用 Protobuf；兼容现有客户端时单独评估 MemoryPack。

Protobuf 的优势：

- C# 和 Go 都有成熟生成器。
- 支持字段演进。
- 可以自动生成消息类型和 Handler/RPC 注册代码。
- 便于 Robot、后台工具和跨语言服务接入。

官方资料：

- [Protocol Buffers Go Reference](https://protobuf.dev/reference/go/)
- [Go Generated Code Guide](https://protobuf.dev/reference/go/go-generated/)

如果要求 Go 服务端直接兼容当前 MemoryPack 客户端，则需要实现并验证 Go MemoryPack Codec。该工作应作为独立兼容性课题，不与第一版内核同时进行。

### 问题 16：是否使用现成 Actor 框架？

结论：先抽象，再通过原型和基准决定，不立即绑定。

Proto.Actor Go 已提供 Actor、PID、Mailbox、Future、Remote、Cluster、Virtual Actor、Protobuf 和 gRPC：

- [Proto.Actor Documentation](https://proto.actor/docs/protoactor/)
- [Proto.Actor Go Package](https://pkg.go.dev/github.com/asynkron/protoactor-go/actor)

但 ET 的核心模型是“一个 Fiber 管理很多 Entity”，并包含确定性 Tick、Entity 树和特定的 Location 语义。直接把每个 Entity 映射为 Proto.Actor Actor，可能改变原有并发边界。

推荐方案：

1. 自研最小的本地 Fiber/mailbox 内核。
2. 定义 `Transport`、`RemoteRouter` 和 `LocationStore` 接口。
3. 实现一个 Proto.Actor 适配原型。
4. 用顺序性、吞吐、P99 延迟、内存、迁移能力和故障恢复进行比较。
5. 根据测试结果决定是否将 Proto.Actor 用作远程通信层。

---

## 四、工程化类问题

### 问题 17：Handler、System 和 Opcode 如何注册？

结论：通过 `go generate` 生成显式注册表，不依赖大量运行时反射。

建议提供 `etgen` 工具，扫描约定注释或声明文件，例如：

```go
//et:handler scene=Map request=C2M_Move response=M2C_Move
func HandleMove(ctx *et.Context, req *pb.C2M_Move) (*pb.M2C_Move, error) {
    return nil, nil
}
```

生成内容包括：

- Opcode 常量。
- Request/Response 关系。
- SceneType 到 Handler 的映射。
- Component TypeID。
- Entity 生命周期 System 注册表。
- 配置类型注册表。

生成代码应当可重复、可审查，并在 CI 中检查生成结果是否最新。

### 问题 18：如何在 Go 中实现 ET Analyzer 思想？

结论：实现自定义 `etvet`，接入 `go vet`、IDE 和 CI。

Go 官方 `go/analysis` 支持 AST、类型信息、控制流和 SSA，可用于实现模块化静态检查：

- [Go Analysis Package](https://pkg.go.dev/golang.org/x/tools/go/analysis)

第一批建议规则：

1. Component 不应包含业务方法。
2. System 必须操作声明的 Component/Entity 类型。
3. 禁止业务包保存可变全局状态。
4. 禁止跨层导入和反向依赖。
5. 禁止在 Fiber 外解析或修改 Entity。
6. 异步 continuation 捕获裸 Entity 指针时报错。
7. Handler 的 Request/Response 必须匹配生成协议。
8. 所有 goroutine 必须有明确 owner、退出条件或 Context。
9. 忽略 Future/Send 错误时给出诊断。
10. Timer、RPC 和 Session 必须在销毁路径释放。

Go 语言本身禁止 import cycle，可以天然承担一部分包依赖检查；业务层级仍需要 `etvet` 进一步约束。

### 问题 19：配置系统如何处理？

结论：继续使用 Excel/Luban，生成 Go 配置代码。

当前仓库已经包含 Luban Golang 生成模块：

`Packages/cn.etetet.yiuiluban/DontNet~/luban/src/Luban.Golang/Luban.Golang.csproj`

因此可以继续保留：

- Excel 策划工作流。
- Luban Bean、Enum 和 Table 定义。
- 客户端 C# 配置输出。
- 服务端 Go 配置输出。

需要额外统一：

- 数值类型映射。
- 枚举值。
- ID 和外键检查。
- 时间和时区表达。
- 客户端/服务端配置裁剪规则。

### 问题 20：Go 版如何处理热更新？

结论：不复制 DLL 热重载，采用滚动更新和状态迁移。

Go 官方 `plugin` 存在明显限制：

- 仅支持 Linux、FreeBSD 和 macOS，不支持 Windows。
- 插件不能卸载。
- 主程序和插件需要完全一致的工具链、Build Tags 和依赖。
- 对 race detector 支持较差。

官方说明：[Go plugin package](https://pkg.go.dev/plugin)

推荐替代方案：

1. 开发期快速编译和自动重启。
2. 配置和脚本数据热加载。
3. 服务端滚动部署。
4. Actor 停服、快照、恢复和重新注册 Location。
5. 长连接由 Gateway/Router 做短时保活或迁移。
6. 协议按向前、向后兼容方式演进。

### 问题 21：如何做测试和性能验证？

结论：测试框架应当与内核同时开发，而不是最后补充。

测试分层建议：

- Entity 生命周期单元测试。
- Fiber mailbox 顺序测试。
- RPC 超时、取消和重复响应测试。
- 虚拟时钟定时器测试。
- Location 锁、迁移和幂等测试。
- 内存 Transport 多节点集成测试。
- TCP/WebSocket 端到端测试。
- 多 Robot 登录、进地图和移动测试。
- Fuzz 测试协议帧和异常消息。
- Benchmark 测试消息吞吐、P99 和内存。

CI 至少执行：

```text
go test ./...
go test -race ./...
go vet ./...
etvet ./...
```

Go race detector 只能发现实际执行路径中的竞态，因此需要让压力测试和集成测试覆盖真实并发路径：

- [Go Data Race Detector](https://go.dev/doc/articles/race_detector)

### 问题 22：使用哪个 Go 版本？

结论：开发工具链使用 Go 1.26 最新补丁，模块兼容基线可先设为 Go 1.25。

截至调研日期，Go 1.26 是当前稳定系列，Go 1.26.5 发布于 2026-07-07。Go 官方维持 Go 1 兼容性承诺，并支持最近两个主要版本：

- [Go Release History](https://go.dev/doc/devel/release)
- [Go 1.26 Release Notes](https://go.dev/doc/go1.26)

---

## 五、实施范围与风险类问题

### 问题 23：原型放在哪里？

结论：先放在当前仓库的独立 ET Package 中，稳定后再拆仓库。

建议目录：

```text
Packages/cn.etetet.goet/
├── AGENTS.md
├── package.json
└── Go~/
    ├── go.mod
    ├── api/
    │   └── proto/
    ├── cmd/
    │   ├── allinone/
    │   ├── login/
    │   ├── map/
    │   └── robot/
    ├── kernel/
    │   ├── entity/
    │   ├── fiber/
    │   ├── actor/
    │   ├── rpc/
    │   ├── timer/
    │   ├── transport/
    │   └── codec/
    ├── game/
    │   ├── login/
    │   ├── unit/
    │   └── map/
    └── tools/
        ├── etgen/
        └── etvet/
```

使用 `Go~` 可以避免 Unity 把 Go 源码作为普通资源导入。创建正式 Package 时仍需遵守当前项目的 Package ID、依赖、`packagegit.json` 和包内 `AGENTS.md` 规则。

### 问题 24：第一阶段最小闭环是什么？

结论：先验证单进程多 Fiber，再进入跨进程。

第一阶段建议包含：

1. Entity、Component、Ref 和生命周期。
2. Fiber mailbox 和串行调度。
3. 本地 Send、Call、Reply。
4. RPC 超时和 Context 取消。
5. Fiber 私有定时器和虚拟时钟。
6. Protobuf 协议生成。
7. TCP Session 和基础帧协议。
8. Login、Gate、Map 三类逻辑 Fiber。
9. Go Robot 完成登录、进地图和移动。
10. `go test -race`、Benchmark 和故障场景测试。

第二阶段再加入：

- 跨进程 Transport。
- ServiceDiscovery。
- ActorLocation。
- 地图动态创建和销毁。
- Actor 状态快照和迁移。
- OpenTelemetry 和运行监控。

### 问题 25：原型成功的验收标准是什么？

结论：以行为正确性和可验证性为主，不只看代码是否能运行。

建议验收项：

1. 同一 Fiber 中的消息严格按约定顺序执行。
2. 业务 Entity 没有数据竞争，`go test -race` 通过。
3. 已销毁 Entity 的旧 Ref 无法解析到新对象。
4. RPC 正常返回、超时、取消和目标不存在均有确定结果。
5. continuation 一定回到原 Fiber 执行。
6. Fiber 销毁后 goroutine、timer 和 pending RPC 全部释放。
7. 虚拟时钟下定时器测试稳定、无真实时间依赖。
8. Robot 能完成完整登录和进地图链路。
9. 单进程可以启动多个逻辑服务进行集成测试。
10. `etgen` 输出可重复，`etvet` 能阻止典型架构违规。

### 问题 26：最大的技术风险是什么？

结论：风险主要集中在异步语义、协议兼容和分布式状态，而不是 Go 语法。

主要风险：

1. Future continuation API 比 C# `await` 更啰嗦，容易影响业务开发体验。
2. 如果错误地阻塞 Fiber 等待 RPC，会产生环形调用死锁。
3. 如果允许异步 goroutine 持有裸 Entity 指针，会破坏状态所有权。
4. MemoryPack 与 Protobuf 的协议选择会影响现有 Unity 客户端兼容性。
5. Location 的迁移、锁和主节点切换容易出现陈旧写入。
6. 自研 Actor Remote 的故障恢复成本较高。
7. Go 没有与 HybridCLR 等价的代码热重载能力。
8. 过早使用对象池、无锁队列和复杂调度器会增加正确性风险。

风险控制原则：

- 先单进程，后跨进程。
- 先正确性，后无锁优化。
- 先普通分配，后对象池。
- 先显式 continuation，后优化开发语法。
- 所有并发设计必须配套 race、故障和超时测试。

### 问题 27：在正式编码前还需要确认哪些决策？

需要进一步确认：

1. 第一版是否必须兼容现有 Unity/MemoryPack 协议。
2. ET-Go 是当前仓库内的实验包，还是独立仓库。
3. 是否允许第一版依赖 Proto.Actor。
4. 数据库首选 MongoDB、PostgreSQL，还是仅定义存储接口。
5. 第一版网络只做 TCP，还是同时支持 WebSocket/KCP。
6. Fiber 与 Scene 是一对一，还是允许一个 Fiber 承载多个 Scene。
7. Update/Tick 是固定帧、事件驱动，还是两者并存。
8. 是否要求 Actor 在节点间在线迁移。

在没有额外要求时，推荐默认选择：

- Protobuf 新协议。
- 当前仓库内原型。
- 自研最小本地 Fiber，Proto.Actor 仅作为对比实现。
- 存储接口 + 内存实现。
- 第一版只做 TCP。
- 一个 Fiber 一个根 Scene。
- 事件驱动为主，地图/房间支持固定 Tick。
- 第一版只做重启恢复，不做在线迁移。

## 最终建议

ET 的 Fiber/Actor 思想与 Go 的 goroutine/channel 非常契合，因此 Go 版本具有可行性。最合理的实现不是复制 C# 类型系统，而是围绕“状态所有权、串行逻辑域、消息通信、代际引用、代码生成和自动验证”建立一个新的 Go 内核。

推荐下一步先编写 ET-Go 的架构决策记录和最小内核原型，只实现 Entity、Ref、Fiber、Mailbox、Timer 和本地 RPC，并用 race test 验证核心并发不变量。核心模型稳定后，再增加网络、Location、ServiceDiscovery 和游戏 Demo。
