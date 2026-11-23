# 当前项目架构设计分析

分析日期：2026-07-02

## 分析范围

本文按项目功能和模块，对当前架构设计做静态分析。分析重点包括：

- Package 分层和依赖方向。
- 客户端、服务端、共享代码边界。
- 分布式服务端拓扑。
- 登录、进地图、地图转场、地图玩法主链路。
- 消息、Actor、Location 和 ServiceDiscovery 的协作方式。
- 配置、协议、资源和热更新架构。
- 当前架构风险与改进点。

本文只基于当前仓库文件做静态分析，未运行 Unity、未编译、未执行测试。

本文还整合了一个 subagent 的只读辅助分析结果。subagent 重点从 `package.json` 依赖、StartConfig 生成代码、Proto 消息域、Scripts 分层和主 demo 包 `cn.etetet.statesync` 的角度补充了架构链路与风险点；其中关键结论已在本地复核。

## 总体架构定位

当前项目是 ET10 风格的模块化在线游戏工程，整体架构目标是支撑大型多人在线游戏开发。它不是单一 Unity 客户端项目，而是一个客户端、服务端、工具链、资源、配置、测试、编辑器桥接共同组成的全栈游戏框架。

总体可以概括为六层：

1. 基础运行时层：`core`、`memorypack`、`sourcegenerator`、`loader`。
2. 基础服务层：`startconfig`、`netinner`、`router`、`http`、`db`、`servicediscovery`、`actorlocation`。
3. 地图实体层：`unit`、`numeric`、`aoi`、`move`、`recast`、`map`。
4. 地图玩法层：`login`、`mapmanager`、`transfer`、`mapplay`、`spell`、`quest`、`item`、`equipment`、`achievement`、`btnode`、`conditionexpr`。
5. 客户端表现层：`statesync`、YIUI 系列、YooAssets、ModelView/HotfixView 代码和 Unity 资源。
6. 工具与验证层：`config`、`proto`、`test`、`robot`、`unitybridge`、`harness`、`hybridclr`、`aspire`。

## Package 架构

项目以 `Packages/cn.etetet.*` 为核心组织单元。每个功能基本独立成包，通过 `package.json` 显式声明依赖。

架构上可以看到几个特点：

- 基础包被大量复用，例如 `core`、`config`、`proto`。
- 高层玩法包依赖底层能力包，例如 `mapplay` 依赖 `map`、`move`、`numeric`、`spell`、`item`、`quest`。
- 服务基础包形成链路，例如 `actorlocation` 依赖 `netinner`，`router` 依赖 `startconfig` 和 `http`，`servicediscovery` 依赖 `netinner`、`loader`、`db`。
- 主 demo 包 `statesync` 依赖大量玩法和工具包，是当前项目的整合入口。

这种 Package 化架构的好处是：

- 功能边界清晰。
- 包可独立演进、分发和组合。
- AI 和 analyzer 可以按包边界约束代码修改。
- 业务模块可以按依赖方向逐层扩展。

风险是：

- 高层整合包依赖很多模块，后续容易成为“全依赖入口”。
- 如果跨包调用没有严格通过 package 依赖和模块边界管理，容易产生隐性耦合。
- 当前部分包缺少包内 `AGENTS.md` 或标准 `PackageType.cs`，规范一致性需要补齐或记录例外。

静态依赖检查还有两个值得记录的点：

- 包含 `cn.etetet.test` 时存在强连通分量：`actorlocation`、`item`、`login`、`map`、`mapplay`、`move`、`quest`、`spell`、`test`。排除 `cn.etetet.test` 后未检测到业务包强连通环。说明测试包的依赖位置需要谨慎，避免测试依赖影响“包依赖单向”的架构判断。
- `cn.etetet.statesync` 的 `package.json` 中大量依赖声明为 `4.0.0`，但本地对应包版本分布在 `0.0.7`、`1.0.0`、`2.0.x`、`3.x`、`4.0.3` 等。当前使用本地 `file:` 或 Unity 本地包时不一定立即出问题，但发布 registry、升级包或做严格版本解析时会成为风险。

## 客户端、服务端和共享代码边界

代码目录按 ET 双端架构划分：

- `Model/Share`：客户端和服务端共享的数据、枚举、事件、协议补充类型、配置绑定。
- `Model/Server`：服务端 Entity、Component 和服务端专有数据。
- `Model/Client`：客户端逻辑层 Entity 和等待结构。
- `Hotfix/Server`：服务端业务逻辑、消息 Handler、Helper、System。
- `Hotfix/Client`：客户端逻辑层消息处理和业务 Helper。
- `ModelView/Client`：客户端 Unity 表现层数据。
- `HotfixView/Client`：客户端 Unity 表现层逻辑。
- `Editor/Share`：Unity Editor 工具。

这种分法支持：

- 服务端不依赖 Unity 表现。
- 客户端逻辑和客户端表现可以拆开。
- 共享代码减少协议、配置、枚举和基础数据重复。
- 热更新代码可分 Model/Hotfix/ModelView/HotfixView 管理。

以地图玩法为例：

- 服务端 `mapplay` 创建 Unit、挂组件、处理宠物攻击、AOI 通知和数值广播。
- 客户端 `mapplay` 接收 Unit 创建/移除/数值变化消息并维护逻辑实体。
- `HotfixView` 根据逻辑事件创建 GameObject、同步位置/旋转、播放动画、显示 UI。

这说明逻辑状态和 Unity 表现之间有明确隔离。

## 服务端拓扑架构

`startconfig` 生成代码显示项目支持多套拓扑：

- `Localhost`
- `Mmo`
- `Moba`
- `Release`

`Localhost` 包含：

- `ServiceDiscovery`
- `Location`
- `RouterManager`
- 多个 `Router`
- `Realm`
- `Gate1`
- `Gate2`
- `MapManager`

`Moba` 还包含：

- `Match`
- `MapManager`
- `Realm`
- `Gate`
- `Location`

这说明服务端不是单进程单场景假设，而是以 StartScene 为单位拆分服务。每个场景可以作为 Fiber 运行，并通过 Actor 消息通信。

典型职责：

- `ServiceDiscovery`：服务注册、发现、订阅和主节点切换。
- `RouterManager` / `Router`：客户端连接路由入口。
- `Realm`：账号登录入口。
- `Gate`：会话、玩家、进入地图入口。
- `Location`：维护实体位置和 Location 路由。
- `MapManager`：分配和管理地图实例。
- `Map`：承载地图内 Unit、AOI、移动、技能、任务等玩法。

从 subagent 的启动链路分析看，服务端启动路径可概括为：

1. `Program` / `Entry` 初始化。
2. `ConfigLoader` 加载配置。
3. `FiberManager.CreateMainFiber` 创建主 Fiber。
4. 主 Fiber 进入 `StateSync` 后发布 `EntryEvent1/2/3`。
5. 服务端 `EntryEvent2_InitServer` 先创建进程级 `NetInner` 和 `ServiceDiscoveryAgent` Fiber。
6. 再根据 `StartSceneConfig` 为当前进程创建具体业务 Scene Fiber。

`Localhost` 配置是单进程开发拓扑：进程 `1`、内网端口 `40001`，同一进程内包含 `ServiceDiscovery`、`Location`、`RouterManager`、4 个 `Router`、`Realm`、2 个 `Gate`、`MapManager`。`Release`、`Moba` 等配置则更接近拆进程部署形态。

## Fiber / Actor 架构

`core` 中的 `FiberManager` 表明项目以 Fiber 为运行单元，并支持：

- 主线程调度。
- 独立线程调度。
- 线程池调度。
- 父 Fiber 继承 Singleton。
- 根据 SceneType 分发 `FiberInit`。

架构含义：

- 每个服务场景可以运行在不同 Fiber。
- Fiber 内保持类似单线程编程体验。
- 跨 Fiber 通过 Actor 消息通信。
- Entity 挂在 Scene/Fiber 上，由该 Fiber 负责生命周期。

这种架构适合 MMO 服务端，因为它把并发隔离在 Fiber/Actor 边界，业务代码多数情况下不需要直接处理锁。

## 消息架构

项目消息架构由三部分组成：

1. `Proto/` 源协议：各业务包维护自己的协议文件。
2. `cn.etetet.proto/CodeMode`：生成后的 Client、Server、ClientServer 协议代码。
3. Handler：按 SceneType 注册消息入口。

消息类型大致分为：

- Session 请求：客户端到 Realm/Gate。
- Location 请求：客户端到具体 Unit 所在 Map。
- 内网请求：服务间 RPC，例如 Gate 到 MapManager、Map 到 MapManager。
- 普通广播/通知：服务端推送客户端。
- UnityBridge 请求：外部工具到 Unity Editor。

命名上通常通过前缀表达方向：

- `C2G`：Client 到 Gate。
- `G2C`：Gate 到 Client。
- `C2M`：Client 到 Map。
- `M2C`：Map 到 Client。
- `G2Map`：Gate 到 Map。
- `Map2G`：Map 到 Gate。
- `A2MapManager`：Actor/服务 到 MapManager。
- `M2M`：Map 到 Map。

这种命名让消息链路和服务边界比较直观。

## ServiceDiscovery 架构

`servicediscovery` 是当前分布式架构的服务目录层。它提供：

- 服务注册。
- 服务注销。
- 服务查询。
- 服务订阅。
- 心跳。
- 变更通知。
- Agent 转发。
- 主节点切换。
- 故障恢复。

`ServiceDiscoveryAgentSystem` 中可以看到：

- Agent Awake 后保存 `MessageSender`。
- 自动添加代理心跳组件。
- 启动后台注册循环。
- 使用状态位控制 Bootstrapping、MutationSyncing 等流程。
- 背景循环通过 `EntityRef<ServiceDiscoveryAgent>` 处理 await 后实体安全。
- 注册和脏数据同步采用重试策略。

架构上，业务场景不直接硬编码目标服务地址，而是通过 `ServiceDiscoveryProxy` 查询目标 SceneName 或 ActorId。这让地图、Gate、Location、MapManager 等服务可以动态发现。

## ActorLocation 架构

`actorlocation` 解决“某个实体当前在哪个 Actor/Fiber 上”的问题。典型用于玩家 Unit 的 Location 路由。

关键能力：

- `LocationComponent` 保存对象位置。
- `LocationProxyComponent` 面向业务侧提供 Add/Get/Remove/Lock/Unlock。
- `MessageLocationSender` 根据 Location 把消息送到目标实体所在 Actor。
- `LockWithToken` 和 `UnLock(..., lockToken)` 保护实体迁移期间的一致性。
- Location 主节点由服务发现结果选择，支持主节点切换、重试和清空缓存。

架构意义：

- 客户端发送 `C2M_*` 到 Unit，不需要知道 Unit 当前在哪个 Map 进程。
- 地图转场期间通过 Location Lock 防止旧 Actor 和新 Actor 同时处理同一 Unit 消息。
- Unit 迁移完成后，用新 ActorId 解锁，使后续消息路由到新地图。

## 登录到进地图主链路

从当前代码和协议看，主链路大致是：

1. 客户端 `LoginHelper.Login` 创建 `ClientSenderComponent`。
2. 客户端向 Realm 发送 `C2R_Login`。
3. Realm 返回 Gate 地址、Key、玩家 Id 等信息。
4. 客户端向 Gate 发送 `C2G_LoginGate`。
5. Gate 创建或绑定 Player/Session。
6. 客户端发送 `C2G_EnterMap`。
7. Gate 与 Map/MapManager 协作，让玩家进入地图。
8. Map 创建 Unit，并发送 `M2C_StartSceneChange`、`M2C_CreateMyUnit`、`M2C_CreateUnits` 等消息。
9. 客户端 `SceneChangeHelper` 创建 CurrentScene，等待主角 Unit 创建，加载 NavMesh，触发 `SceneChangeFinish`。

这个链路体现了三层职责：

- Realm 负责登录入口。
- Gate 负责会话和玩家连接。
- Map/MapManager 负责地图实例和 Unit 生命周期。

更细的服务间消息链路是：

1. 客户端先访问 `RouterManager` 的 `/get_router`，拿到 `Realm` 和 `Router` 地址。
2. 客户端通过 Router 建到 `Realm` 的会话，发送 `C2R_Login`。
3. `Realm` 通过 `ServiceDiscoveryProxy` 按账号 hash 和 zone 选择 `Gate`，调用 `R2G_GetLoginKey`。
4. 客户端再连 `Gate`，使用 key 执行 `C2G_LoginGate`。
5. 客户端发送 `C2G_EnterMap`。
6. Gate 创建临时 `GateMap` Fiber，发送 `G2Map_EnterMap`。
7. Map 侧创建 Unit 后调用 `TransferHelper.TransferLock`。
8. Transfer 通过服务发现找 `MapManager`，申请地图副本 ActorId，锁 ActorLocation，把 Unit 迁移到目标 Map Fiber。
9. 目标 Map 处理 `M2M_UnitTransferRequest`，恢复 Unit 并通知客户端 `M2C_StartSceneChange` 和 `M2C_CreateMyUnit`。
10. 后续客户端对 Unit 的 `ILocationMessage` / `ILocationRequest` 由 Gate 通过 ActorLocation 路由到 Unit 所在 Map。

## 地图转场架构

`transfer` 包提供地图转场的核心编排。

服务端转场流程：

1. 等待帧末或获取 Mailbox 协程锁。
2. 判断是否需要切换客户端场景。
3. 通过 `ServiceDiscoveryProxy` 找到 `MapManager`。
4. 向 MapManager 申请目标地图实例。
5. 对 Unit 做 Location 加锁。
6. 从 AOI 中移除 Unit。
7. 同进程时直接移交 Unit 对象。
8. 跨进程时将 Unit 和 `ITransfer` 组件序列化为 Bson。
9. 向目标 Map 发送 `M2M_UnitTransferRequest`。
10. 目标 Map 反序列化或接收 Unit，返回新 ActorId。
11. Location 解锁并写入新 ActorId。
12. 通知 MapManager 玩家已进入地图。

客户端转场流程：

1. 收到 `M2C_StartSceneChange`。
2. 判断是否真正切换场景。
3. 创建新的 CurrentScene。
4. 等待 `M2C_CreateMyUnit`。
5. 加载 NavMesh。
6. 发布 SceneChangeStart / SceneChangeFinish。
7. 唤醒等待 `Wait_SceneChangeFinish` 的逻辑。

这个设计把 Unit 迁移、Location 路由、客户端场景加载、地图实例管理拆成清晰的协作边界。

## 地图内玩法架构

地图内玩法由多个包组合：

- `unit`：基础单位。
- `numeric`：数值。
- `aoi`：视野。
- `move`：移动。
- `recast`：寻路。
- `spell`：技能和 Buff。
- `quest`：任务。
- `item`：物品。
- `equipment`：装备。
- `achievement`：成就。
- `mapplay`：地图玩法整合。

`UnitFactory` 是地图内组件装配中心：

- 所有 Unit 都会挂数值、移动、转向、AOI、目标、技能、Buff。
- 玩家额外挂背包和任务。
- 怪物额外挂仇恨和寻路。
- AI 通过配置中的 NumericType.AI 触发 Buff/行为树。

架构上，`mapplay` 不把所有玩法写死在一个大系统里，而是作为组合层调用各基础玩法包。这样能够继续增加新玩法包，但要求包依赖方向保持清晰。

## 战斗、技能、AI 架构

战斗系统的核心抽象是行为树 + Buff：

- `behaviortree` 提供行为树运行时和编辑器。
- `btnode` 提供战斗、AI、客户端表现节点。
- `spell` 把技能配置、Buff 配置、目标选择、消耗、服务端效果和客户端表现串起来。
- `conditionexpr` 把配置表条件表达式编译成行为树。

技能施放链路：

1. `C2M_SpellCast` 进入 Map。
2. `SpellHelper.Cast` 读取 `SpellConfig`。
3. 主技能检查 CD。
4. 创建 Buff 作为技能运行实体。
5. 行为树选择目标。
6. CostNode 行为树检查和扣除消耗。
7. 更新 CD。
8. 初始化 Buff。
9. Buff Add/Update/Remove 通过消息同步客户端。
10. 客户端表现层根据 Buff/Spell 消息播放特效、动画、施法条或指示器。

这个架构让技能、Buff 和 AI 更偏配置化。它的扩展点是新增行为树节点、配置新的 Spell/Buff 资产和 Luban 数据。

## 配置架构

配置架构以 Luban/Excel 为核心：

- 各业务包维护自己的 `Luban/Config`。
- 聚合生成代码集中在 `cn.etetet.config/CodeMode`。
- StartConfig 单独生成服务端拓扑配置。
- Spell/Buff 部分有独立生成配置。

配置被分成：

- Client
- Server
- ClientServer

架构意义：

- 服务端和客户端可以只加载自己需要的配置。
- 配置可以由策划维护。
- 代码中通过 `Fiber().GetSingleton<XXXConfigCategory>()` 获取配置。
- 行为树、条件表达式、技能效果和地图单位都可以配置化。

风险：

- 配置和代码契约强绑定。
- 行为树节点字段、`BTEnv` key、Luban bean 字段不一致时容易运行时出错。
- 导出链路和生成代码需要作为开发流程的一部分严格执行。

## 客户端表现与资源架构

客户端表现架构由三部分组成：

1. Unity 资源：场景、Prefab、材质、动画、特效、UI。
2. YooAssets：资源加载和打包。
3. YIUI + HotfixView：UI 和表现逻辑。

资源集中在：

- `Packages/cn.etetet.map/Bundles`
- `Packages/cn.etetet.statesync/Assets`
- `Packages/cn.etetet.statesync/Bundles`
- YIUI 相关包的 `Assets/GameRes`

`statesync` 包中包含：

- `Scenes/Init.unity`
- 登录、大厅、Loading、Main、HUD、ActionBar、Minimap、血条、目标信息等 UI。
- 技能行为树资产。
- 技能特效和 Buff 特效。
- 技能指示器。
- 角色、武器、动画、战斗 HUD、浮动文字、Nameplate。

架构上，客户端表现不直接决定服务端状态，只消费服务端消息和本地逻辑事件。这样符合服务端权威设计。

## 热更新架构

项目热更新由多层组成：

- `loader`：加载 Model/Hotfix/ModelView/HotfixView。
- `hybridclr`：客户端 C# 热更新和 AOT 补充元数据。
- `CodeMode`：生成配置和协议代码。
- `YooAssets`：资源热更新和 AssetBundle 打包。
- Unity 菜单/脚本：编译、Reload、导出配置、导出 ScriptableObject。

运行指南中说明：

- F6 编译。
- F7 Reload。
- HybridCLR Generate All。
- CopyAotDlls。
- YooAsset AssetBundle Builder。

架构含义是：代码、配置、资源、ScriptableObject 行为树资产都有各自导出/热更链路。

当前入口文件状态需要注意：

- `Packages/manifest.json` 只直接引用了 `cn.etetet.mapplay`，没有直接引用 `cn.etetet.statesync`。
- 仓库内未发现 `MainPackage.txt`。
- 当前主 demo 包从资源和依赖看是 `cn.etetet.statesync`，但 Unity manifest 和发布脚本仍有不一致痕迹，需要在实际初始化、打开 Unity 或发布前确认主包入口策略。

## 测试与验证架构

测试架构由 `cn.etetet.test`、`robot`、`unitybridge` 和业务包测试组成。

特点：

- 测试通过 `ConsoleMode.Test` 运行。
- `ATestHandler` 按特性发现。
- `TestFiberScope` 为每个用例创建隔离测试 Fiber。
- 支持按包名和用例名正则筛选。
- Robot 可以复用客户端逻辑模拟玩家。
- UnityBridge 可以执行编辑器和 PlayMode 相关操作。

这套验证架构适合 AI 开发闭环：写代码、编译、运行测试、检查日志、必要时操作 Unity。

## AI 与编辑器自动化架构

`harness` 和 `unitybridge` 是这个项目区别于普通 ET 项目的关键扩展：

- `harness` 负责 AI 技能路由、规范和行为约束。
- `unitybridge` 把 Unity Editor 暴露为可命令化的宿主。
- `test` 提供游戏逻辑验证入口。
- `config/ExcelMcp` 提供 Excel 配置操作入口。

组合起来后，AI 可以：

- 根据技能规则修改代码。
- 操作 Unity 刷新、编译、进出 PlayMode。
- 查询场景、Prefab、Inspector、GameObject。
- 执行 Unity 测试和截图。
- 修改或检查 Excel 配置。
- 执行业务测试。

这是一套面向 AI 参与开发的架构，而不只是运行时框架。

## 架构优点

### 模块边界清晰

功能按包拆分，包依赖显式声明，适合大型项目多人协作和 AI 辅助开发。

### 服务端扩展性强

ServiceDiscovery、ActorLocation、Fiber、Router、MapManager 让服务端具备多进程、多地图、多区服扩展基础。

### 逻辑和表现分离

服务端逻辑、客户端逻辑、客户端表现、编辑器工具分层明确，便于热更新和服务端独立运行。

### 配置化能力强

技能、Buff、AI、任务、物品、地图、转场、数值等大量能力可通过 Luban/行为树/ScriptableObject 配置扩展。

### 自动化验证能力强

测试框架、Robot、UnityBridge、AI Harness 组合后，具备从代码到运行时验证的闭环基础。

### 异步安全意识明确

核心异步流程中大量使用 `EntityRef<T>`，跨 await 后重新获取实体，符合 ET 的 Entity 生命周期安全规则。

## 架构风险与注意点

### 主整合包依赖过重

`cn.etetet.statesync` 依赖大量业务和工具包，适合作为 demo 整合入口，但不宜继续无边界承载新业务逻辑。新功能应优先拆独立包，再由整合包引用。

### 测试包导致依赖环观感

静态依赖分析显示，包含 `cn.etetet.test` 时存在跨多个业务包的强连通分量；排除 `test` 后业务包未检测到强环。测试包通常会依赖被测对象，容易形成反向依赖，因此应避免让生产包依赖测试包。当前 `actorlocation` 的 package 依赖里包含 `cn.etetet.test`，需要重点确认是否只是历史遗留或开发期依赖。

### 包规范不完全统一

部分包缺少 `AGENTS.md` 或标准 `PackageType.cs`。如果严格按 harness 规范推进，需要明确哪些是例外包，哪些需要补齐。

### 根级解决方案当前缺失

当前静态扫描未发现根级 `ET.sln`。项目规范要求唯一编译入口是 `dotnet build ET.sln`，因此正式构建前需要通过初始化或 Unity/Rider 工程生成确认根解决方案。

### 配置与行为树契约复杂

行为树节点、技能配置、Buff 配置、条件表达式、Luban bean、ScriptableObject 之间有多处契约。如果缺少导出校验和测试，运行时错误定位成本会较高。

### 服务基础模块复杂度较高

ServiceDiscovery 和 ActorLocation 已经处理主备、重试、分区、锁 token、缓存清理等问题。扩展这些模块时必须优先补测试，避免引入一致性问题。

### Helper 层可能继续膨胀

当前 Helper 是合理编排层，但如果新功能继续把所有流程塞进 Helper，后续会形成大类。建议按业务子域拆 Helper，或为长期状态引入专门组件。

### Demo 硬编码入口

`C2G_EnterMapHandler` 当前把进图目标写为 `"Map2"`、`mapId = 0`、`unitConfigId = 1001`。这符合 demo 快速进图，但不适合作为正式角色出生、选线、选地图或副本入口策略。正式化时应改为配置、角色数据或服务端分配策略驱动。

### 发布脚本迁移残留

`Scripts/Publish.ps1` 仍复制 `Packages/cn.etetet.wow/Bundles/Bson`，但当前仓库没有发现 `cn.etetet.wow` 包，主 demo 包是 `cn.etetet.statesync`。这说明发布脚本存在迁移残留，实际发布前需要修正资源源路径和主包路径。

## 建议的后续架构治理

1. 为缺少 `AGENTS.md` 的业务包补齐包内规则，或记录为明确例外。
2. 确认根级 `ET.sln` 的生成/初始化流程，并在文档里标注当前环境状态。
3. 为任务、装备、成就等玩法补更完整的测试，服务端权威逻辑尤其需要覆盖。
4. 为行为树和技能配置增加导出后校验清单，确保节点字段、Env key、配置字段一致。
5. 控制 `statesync` 的职责，把它保留为 demo/整合包，不把新增业务直接沉积进去。
6. 对跨地图转场、Location、ServiceDiscovery 的关键链路保留高覆盖测试，因为这些模块是分布式一致性的核心。
7. 新增异步流程时继续强制使用 `EntityRef<T>`，并明确异步边界来自 RPC、IO、资源、Timer、协程锁或跨 Fiber 结果。
8. 清理或确认 `cn.etetet.test` 引入的依赖环观感，避免生产包依赖测试包。
9. 修正 `statesync` 依赖版本声明和本地包版本不一致的问题，至少在发布前明确版本策略。
10. 修正 `Packages/manifest.json`、`MainPackage.txt` 缺失状态和 `Scripts/Publish.ps1` 中的 `cn.etetet.wow` 残留。
11. 把 `C2G_EnterMapHandler` 中的 demo 进图参数改为配置或角色数据驱动。

## 结论

当前项目架构是一个典型的 ET10 分布式在线游戏框架架构：底层以 Fiber、Entity、Actor 和消息系统为基础；服务端通过 ServiceDiscovery、Router、Location、Gate、MapManager 和 Map 组成可扩展拓扑；玩法通过独立 Package 叠加；客户端通过 ModelView/HotfixView、YIUI 和 YooAssets 做表现；工具层通过 Luban、Proto、HybridCLR、UnityBridge、Test 和 Harness 支撑开发闭环。

这套架构的优势在于边界清楚、扩展点多、自动化能力强。主要风险在于包规范一致性、配置契约复杂度、整合包依赖膨胀，以及根级工程文件状态需要进一步确认。
