# 当前项目功能分析

分析日期：2026-07-02

## 分析基准

- 已读取根入口规则，并确认 `Packages/cn.etetet.harness/` 存在。
- 已读取 `Packages/cn.etetet.harness/AGENTS.md`。
- 已读取 `Packages/cn.etetet.harness/skills/index.md`，本次按 `et-code` 口径做静态架构与功能分析。
- 本文基于当前仓库文件做静态分析，覆盖 `README.md`、各包 `package.json`、`Proto/`、`Luban/`、`Scripts/`、`CodeMode/`、资源目录和关键源码入口。
- 本次未运行 Unity、未编译、未执行测试。

## 项目定位

当前项目是基于 ET10 的 Unity + .NET 多人在线游戏框架项目，采用 `Packages/cn.etetet.*` 模块化架构。项目目标是支持大型在线游戏开发，核心方向包括：

- 双端 C# 开发。
- ET Fiber/Actor 运行时。
- 分布式多进程服务器。
- 客户端与服务端热更新。
- Luban/Excel 配置驱动。
- YooAssets 资源管理。
- YIUI 客户端 UI 框架。
- 自动化测试与机器人测试。
- UnityBridge 编辑器桥接。
- AI Harness 开发规范与分析器约束。

从当前资源和代码看，主业务 demo 更接近一个状态同步 MMO/RPG 原型，包含登录、进地图、单位同步、移动、技能、Buff、任务、物品、装备、成就、宠物、怪物 AI、地图转场和 UI 表现。

## 顶层结构

- `Assets/`：Unity 项目资源，当前主要包含 YIUI 设置、TextMesh Pro、DOTween、Odin Inspector、项目设置等基础资源。
- `Packages/`：ET 模块化包，是主要代码、配置、协议和资源所在地。
- `Book/`：运行指南、教程和项目说明。
- `docs/`：项目文档，目前已有 `superpowers/specs` 下的设计文档。
- `Scripts/`：项目初始化、启动、发布、Aspire 运行等 PowerShell 脚本。
- `ProjectSettings/`：Unity 项目配置。

## 核心框架能力

### `cn.etetet.core`

ET 核心运行时。主要能力包括：

- Entity、Component、System、Scene 等 ECS 基础。
- Fiber、ActorId、进程/纤程地址、调度器。
- ETTask、ETVoid、异步状态机封装。
- EntityRef、ETCancellationToken、协程锁。
- 事件系统、定时器、对象池、日志。
- 网络抽象、KCP/TCP 通道、消息序列化。
- MemoryPack 和 Bson 相关序列化支持。

### `cn.etetet.sourcegenerator`

源码生成器和分析器包。用于：

- ET System 模板生成。
- 架构规则检查。
- 包边界和依赖约束。
- 测试命名等规则检查。

### `cn.etetet.memorypack`

集成 MemoryPack，用于高性能序列化。

### `cn.etetet.loader`

加载器包，负责：

- Model、Hotfix、ModelView、HotfixView 加载。
- 服务端启动初始化。
- 配置加载。
- 运行时代码热重载相关入口。

### `cn.etetet.hybridclr`

Unity C# 热更新方案，服务于客户端热更、AOT 补充元数据和打包流程。

### `cn.etetet.packagemanager`

ET Package 管理工具。

### `cn.etetet.harness`

AI Harness 分发包，包含 AI 开发规范、技能路由、分析器约束和工作流说明。该包不承载业务运行时代码。

## 分布式服务端能力

### `cn.etetet.startconfig`

服务器起服配置包。当前存在多套配置：

- `Localhost`
- `Mmo`
- `Moba`
- `Release`

`Localhost` 配置中包含：

- `ServiceDiscovery`
- `Location`
- `RouterManager`
- 多个 `Router`
- `Realm`
- `Gate1`
- `Gate2`
- `MapManager`

`Moba` 配置中还能看到 `Match` 场景。整体说明项目支持按环境拆分多进程、多场景、多区服拓扑。

### `cn.etetet.servicediscovery`

服务发现系统。主要能力包括：

- 服务注册。
- 服务注销。
- 服务查询。
- 服务订阅和取消订阅。
- 心跳与租约。
- 服务变更通知。
- ServiceDiscoveryAgent 转发请求到当前主节点。
- 主节点切换、故障恢复、并发访问和网络分区相关测试。

### `cn.etetet.actorlocation`

Actor Location 机制。主要能力包括：

- 对象位置注册、查询、移除。
- Location 加锁和解锁。
- 带 token 的锁保护。
- Lock 超时、重试和补偿。
- LocationProxy 和 MessageLocationSender。
- 通过 `ILocationMessage`、`ILocationRequest` 等协议将消息路由到目标实体所在场景。

### `cn.etetet.netinner`

内网消息模块。主要能力包括：

- 进程内和进程间消息发送。
- `MessageSender`、`ProcessOuterSender`。
- `NetInner` 场景初始化。
- `A2NetInner_Request`、`A2NetInner_Message` 转发。

### `cn.etetet.router`

软路由模块。用于网络入口、连接路由和攻击防护。协议中包含 `HttpGetRouterResponse`。

### `cn.etetet.http`

轻量 HTTP 模块。

### `cn.etetet.db`

数据库基础模块，目前包描述为数据库基础占位和后续扩展脚手架。

### `cn.etetet.aspire`

.NET Aspire 集成包。根据 StartConfig 为 ET 进程和副本创建 Aspire 服务，并传递场景名、StartConfig、端口等启动参数，同时接入 OpenTelemetry。

## 登录与地图主流程

### `cn.etetet.login`

登录系统，覆盖客户端、Realm、Gate 和会话管理。主要能力包括：

- 客户端 `LoginHelper.Login`。
- 客户端独立网络纤程。
- Realm 登录。
- Gate 登录验证。
- Session、Player、SessionPlayer 绑定。
- Ping。
- Logout。
- EnterMap。
- Router 地址获取与检查。

相关协议包括：

- `C2R_Login` / `R2C_Login`
- `C2G_LoginGate` / `G2C_LoginGate`
- `C2G_EnterMap` / `G2C_EnterMap`
- `C2G_Ping` / `G2C_Ping`
- `C2G_Logout` / `G2C_Logout`

### `cn.etetet.map`

地图基础系统。主要能力包括：

- 当前场景管理。
- 地图配置模型。
- 客户端 `CurrentScenesComponent`。
- 服务端地图消息通知。
- AOI 位置变更通知。
- 地图文本辅助。
- 地图资源和 NavMesh 资源加载。

当前地图资源包括：

- `Packages/cn.etetet.map/Bundles/Scenes/Map1.unity`
- `Packages/cn.etetet.map/Bundles/Scenes/Lordaeron.unity`

### `cn.etetet.mapmanager`

地图管理器。主要能力包括：

- 地图实例分配。
- 地图池管理。
- 地图生命周期管理。
- 玩家进入地图后的确认。
- 地图退出通知。
- 多地图协调。

### `cn.etetet.transfer`

地图转场和跨地图 Unit 转移模块。主要能力包括：

- 客户端请求地图传送。
- 服务端向 MapManager 申请地图实例。
- Location 加锁保护 Unit 迁移。
- 同进程直接移动 Unit。
- 跨进程通过 Bson 序列化转移 Unit 和可转移组件。
- 客户端场景切换流程。
- `MapTransferRule` Luban 配置。

相关协议包括：

- `C2M_TransferMap` / `M2C_TransferMap`
- `G2Map_EnterMap` / `Map2G_EnterMap`
- `M2M_UnitTransferRequest` / `M2M_UnitTransferResponse`
- `A2MapManager_GetMapRequest` / `A2MapManager_GetMapResponse`

### `cn.etetet.mapplay`

地图玩法组合层。主要能力包括：

- `UnitFactory` 创建玩家、怪物、NPC、宠物等单位。
- 单位创建后添加数值、移动、转向、AOI、目标、技能、Buff 等组件。
- 玩家额外挂载物品和任务组件。
- 怪物额外挂载仇恨和寻路组件。
- 根据 `NumericType.AI` 启动 AI Buff/行为树。
- AOI 进出视野同步。
- 数值变化广播。
- 宠物攻击。
- 客户端单位创建、移除、表现同步。
- 输入系统、相机、动画、GameObject 绑定、主界面事件。

相关协议包括：

- `M2C_CreateMyUnit`
- `M2C_CreateUnits`
- `M2C_RemoveUnits`
- `M2C_NumericChange`
- `C2M_PetAttack`
- `M2C_StartSceneChange`
- `M2C_Error`

## MMO 玩法模块

### `cn.etetet.unit`

基础 Unit 系统，提供包含位置、旋转等信息的场景实体。

### `cn.etetet.numeric`

数值系统。主要能力包括：

- `NumericComponent`。
- 基于 Luban 的 `NumericType` 配置。
- HP、MP、Speed、AI、坐标、Yaw 等 KV 数值。
- 数值变化事件和广播基础。

### `cn.etetet.aoi`

九宫格 AOI 实现，用于单位视野管理和进出视野事件。

### `cn.etetet.move`

MMO 移动系统。主要能力包括：

- 路径寻路结果上报。
- 停止移动。
- 转向同步。
- 速度变化影响移动速度。
- 客户端等待 Unit 停止。

相关协议包括：

- `C2M_PathfindingResult`
- `C2M_Stop`
- `M2C_PathfindingResult`
- `M2C_Stop`
- `M2C_Turn`

### `cn.etetet.recast`

3D Recast 寻路库。当前地图资源中已有 NavMesh 数据：

- `NavMesh-Map1.asset`
- `NavMesh-pvp_lordaeron_arena.asset`

### `cn.etetet.spell`

技能与 Buff 系统。主要能力包括：

- 技能施放。
- 主技能 CD 检查。
- 指定目标施法。
- 行为树目标选择。
- CostNode 消耗检查与扣除。
- 施法打断当前技能。
- Buff 创建、初始化、超时移除。
- Buff 叠加规则：加层数、加时间、替换等。
- Buff Tick。
- Buff 添加、更新、移除同步到客户端。
- 技能 CD 同步到客户端。
- 服务端 Buff 效果树和客户端 Buff 表现树。
- Spell Editor 编辑器。
- 技能指示器、技能特效、Buff 资源。

相关协议包括：

- `C2M_SpellCast`
- `M2C_SpellAdd`
- `M2C_SpellRemove`
- `M2C_BuffAdd`
- `M2C_BuffUpdate`
- `M2C_BuffRemove`
- `M2C_UpdateCD`

### `cn.etetet.behaviortree`

行为树基础包。主要能力包括：

- `BTRoot`
- `BTSequence`
- `BTSelector`
- `BTNot`
- `BTCondition`
- `BTAction`
- `BTDecorate`
- `BTEnv`
- `BTDispatcher`
- 行为树运行辅助。
- Unity 行为树编辑器。
- 节点绘制、复制粘贴、参数下拉、验证、运行路径调试、代码导出。

### `cn.etetet.btnode`

业务行为树节点扩展包。当前包含大量面向战斗和 AI 的节点：

- 技能目标获取。
- Buff 添加、移除、叠层、延长时间。
- 伤害。
- 数值消耗与数值变化。
- 技能修正。
- 创建宠物。
- 驯服野兽。
- 转向目标或位置。
- 地图传送。
- 服务端目标选择器：单体、圆形、扇形、矩形、施法者周围等。
- 客户端动画参数设置。
- 客户端创建技能特效、Buff 特效、假子弹。
- 怪物巡逻、追击、返回。
- 宠物待机、跟随、追击、返回。
- 仇恨判断。

### `cn.etetet.conditionexpr`

条件表达式包。用于把 Excel/Luban 中填写的条件表达式编译成行为树并运行。典型用途：

- 副本进入条件。
- 奖励领取条件。
- 任务接取条件。
- 功能开启条件。

支持能力包括：

- `>`、`>=`、`<`、`<=`、`==`、`!=`。
- `&&`、`||`、`!`、括号。
- 叶子条件绑定错误码。
- 默认错误码。
- `OwnerKey.Variable` 多 owner 读取。
- 专用条件节点参数。
- 变量注册和运行时检查。

### `cn.etetet.quest`

任务系统。主要能力包括：

- 接取任务。
- 提交任务。
- 放弃任务。
- 同步任务数据。
- 查询可接任务。
- 获取任务详情。
- 多目标任务和进度追踪。
- 点击 NPC。
- 击杀怪物事件触发任务进度。
- 客户端任务数据同步和事件通知。

相关协议包括：

- `C2M_AcceptQuest`
- `C2M_SubmitQuest`
- `C2M_AbandonQuest`
- `C2M_SyncQuestData`
- `C2M_QueryAvailableQuests`
- `C2M_GetQuestDetail`
- `C2M_ClickUnitRequest`
- `M2C_CreateQuest`
- `M2C_UpdateQuestObjective`
- `M2C_UpdateQuest`

### `cn.etetet.item`

物品与背包系统。主要能力包括：

- 添加物品。
- 移除物品。
- 使用物品。
- 丢弃物品。
- 背包容量管理。
- 槽位管理。
- 相同物品自动堆叠。
- 物品移动、交换和部分堆叠。
- 一键整理背包。
- 服务端权威变更。
- 服务端主动同步客户端。
- 客户端主动同步完整背包。

相关协议包括：

- `C2M_UseItem`
- `C2M_SyncBagData`
- `C2M_MoveItem`
- `C2M_DiscardItem`
- `C2M_SortBag`
- `M2C_UpdateItem`
- `M2C_UpdateBagCapacity`

### `cn.etetet.equipment`

装备系统。基于 Item 扩展。主要能力包括：

- 装备作为特殊 Item。
- `EquipmentItemComponent` 标记装备并记录强化等级。
- 9 个装备槽位：头、胸、腰、腿、脚、主手、副手、饰品 1、饰品 2。
- 穿戴装备。
- 卸下装备。
- 装备强化。
- 装备栏和背包中的 `SlotIndex` 语义区分。
- 装备更新独立于道具更新。

协议中当前可见 `EquipmentData` 和 `M2C_UpdateEquipment`。README 中还描述了穿戴、卸下、同步、强化相关协议能力。

### `cn.etetet.achievement`

成就系统。主要能力包括：

- 获取成就列表。
- 领取成就奖励。
- 成就进度推送。
- 成就完成通知。
- 获取成就详情。
- 获取成就分类。
- 获取成就统计。
- 处理击杀、等级、任务完成、道具收集、地图探索、进入地图等事件。
- 成就前置条件检查。
- 成就奖励预览。

相关协议包括：

- `C2M_GetAchievements`
- `C2M_ClaimAchievement`
- `C2M_GetAchievementDetail`
- `C2M_GetAchievementCategories`
- `C2M_GetAchievementStats`
- `M2C_AchievementProgress`
- `M2C_AchievementComplete`

### `cn.etetet.robot`

机器人系统。主要能力包括：

- 机器人登录。
- 控制台触发机器人登出。
- 共享客户端逻辑，用于自动化测试和压测。

相关协议包括：

- `Robot_LoginRequest`
- `Robot_LoginResponse`
- `Console2Robot_LogoutRequest`
- `Console2Robot_LogoutResponse`

### `cn.etetet.lockstep`、`cn.etetet.truesync`、`cn.etetet.lsentity`

帧同步相关能力：

- `cn.etetet.lockstep`：预测回滚帧同步 demo。
- `cn.etetet.truesync`：帧同步定点数实现。
- `cn.etetet.lsentity`：帧同步实体相关基础。

协议中包含匹配、房间、帧输入、Hash 校验、重连、房间初始化等。

### `cn.etetet.statesync`

当前主 demo/整合包。依赖大量业务包，包含：

- `Scenes/Init.unity` 默认运行场景。
- 登录、Loading、Lobby、Main、HUD、ActionBar、Minimap、血条、目标信息等 UI 资源。
- 技能行为树资源。
- 技能特效、Buff 特效、技能指示器。
- 角色、武器、动画、战斗 HUD、浮动文字、Nameplate 等资源。
- 状态同步相关协议，如热重载、Benchmark、测试 Hotfix 消息。

## 客户端 UI 与资源模块

### YIUI 系列

- `cn.etetet.yiui`：YIUI 框架生成入口。
- `cn.etetet.yiuiframework`：YIUI 核心框架，包含 UI 生命周期、面板、组件、模板 Prefab 等。
- `cn.etetet.yiuiinvoke`：独立 Invoke 功能。
- `cn.etetet.yiuiyooassets`：YIUI 与 YooAssets 加载集成。
- `cn.etetet.yiuiluban`：YIUI Luban 配置工具。
- `cn.etetet.yiuigm`：GM 命令 UI。
- `cn.etetet.yiuitips`：Tips 弹窗。
- `cn.etetet.yiuiloopscrollrectasync`：异步无限循环列表。
- `cn.etetet.yiuireddot`：红点系统。
- `cn.etetet.yiui3ddisplay`：UI 中展示 3D 模型。
- `cn.etetet.yiuieffect`：UI 特效资源。

### 资源管理

- `cn.etetet.yooassets`：Unity 资源管理系统。
- 项目使用 YooAssets 进行资源打包和加载。
- 打包流程中需要设置 YooConfig、构建 AssetBundle。

### 当前发现的主要资源

地图：

- `Packages/cn.etetet.map/Bundles/Scenes/Map1.unity`
- `Packages/cn.etetet.map/Bundles/Scenes/Lordaeron.unity`

单位预制体：

- `Bear`
- `BearBoss`
- `Boar`
- `Character_HumanMale`
- `Mage`
- `Skeleton_Warrior`
- `TrainingDummy`

UI 资源：

- Loading
- Login
- Lobby
- MainPanel
- HUDPanel
- ActionBar
- CastSlider
- HPView
- Minimap
- PlayerInfo
- TargetInfo
- UnitInfo
- Tips
- GM
- RedDot

战斗与技能资源：

- Frost Bolt
- Frost Nova
- Blizzard
- AutoAttack
- TameBeast
- PhaseAdd
- PhaseRemove
- MonsterAutoAttack
- MonsterBattle
- Transfer
- 多种技能指示器和粒子特效。

## 配置、协议和生成代码

### Luban 配置

当前存在 Luban 配置的包包括：

- `cn.etetet.achievement`
- `cn.etetet.conditionexpr`
- `cn.etetet.config`
- `cn.etetet.equipment`
- `cn.etetet.item`
- `cn.etetet.map`
- `cn.etetet.numeric`
- `cn.etetet.quest`
- `cn.etetet.spell`
- `cn.etetet.startconfig`
- `cn.etetet.statesync`
- `cn.etetet.transfer`

主要配置表包括：

- `Achievement`
- `AchievementCategory`
- `Equipment`
- `Item`
- `Map`
- `MapUnit`
- `Unit`
- `NumericType`
- `Quest`
- `QuestObjective`
- `Text`
- `MapTransferRule`
- `StartMachine`
- `StartProcess`
- `StartScene`
- `StartZone`

### `cn.etetet.config`

配置包主要能力：

- 聚合 Luban 配置。
- 生成 Client、Server、ClientServer 多端配置代码。
- 内置 `ET.ExcelExporter`。
- 内置 `ET.ExcelMcp`，基于 EPPlus 提供 Excel MCP 能力。

`ET.ExcelMcp` 支持：

- 创建和转换 Excel 文件。
- 工作表创建、删除、重命名、列出。
- 单元格读写、编辑、清空。
- 公式。
- 类型识别。
- 中文字符。
- 图表、数据透视表、条件格式等增强能力。

### `cn.etetet.proto`

协议生成工具和协议代码聚合包。当前协议域包括：

- Achievement
- ActorLocation
- Equipment
- Item
- LockStep
- Login
- Map
- MapManager
- MapPlay
- Move
- Quest
- Robot
- Router
- ServiceDiscovery
- Spell
- StateSync
- Test
- Transfer
- UnityBridge

## UnityBridge 与 AI 开发辅助

### `cn.etetet.unitybridge`

Unity Editor 本地桥接与命令行工具。根据 README 和 proto，能力包括：

- 状态与生命周期：Ping、HostState、Compile、Refresh、RegenProject、EnterPlay、ExitPlay、Reload、EditorGetState、Pause、Undo、Redo。
- 资源操作：AssetSearch、AssetFind、AssetGetPath、AssetLoad、AssetImport、AssetRefresh、AssetReadText。
- 场景操作：SceneGetHierarchy、SceneGetActive、SceneLoad、SceneSave、SceneNew。
- 选择集：SelectionGet、SelectionSet、SelectionAdd、SelectionRemove、SelectionClear。
- GameObject：Create、Destroy、Duplicate、Find、GetInfo、Rename、SetActive。
- Transform：Get、SetPosition、SetRotation、SetScale、SetParent、LookAt、Reset、SetSiblingIndex。
- Inspector：GetComponents、GetProperties、GetProperty、FindProperty、SetProperty、SetProperties、AddComponent、RemoveComponent。
- Prefab：Instantiate、Save、Unpack、GetInfo、GetHierarchy、Apply。
- GameView：GetResolution、ListResolutions、SetResolution。
- 其他：ConsoleGetLogs、EditorLog、ScreenshotCapture、UnityTestRun、MenuItemExecute、BatchExecute、TestEcho。

该模块让 AI 或脚本可以直接操作 Unity Editor，而不是只改文件。

### `cn.etetet.test`

自动化测试框架。主要能力：

- 基于 `ConsoleMode.Test` 运行。
- 按 `[Test]` 特性自动发现测试处理器。
- 支持按包名和用例名正则过滤。
- 每个用例可创建独立 `TestCase` 子纤程。
- 适合做完整客户端和服务端交互测试。
- 测试命名由分析器强制约束，格式为 `{PackageType}_{TestName}_Test`。

运行入口示例：

```powershell
dotnet ./Bin/ET.App.dll --Console=1 --SceneName=Test
```

交互命令示例：

```text
Test
Test --Name=CreateRobot
```

## 构建、运行与发布能力

运行指南中说明：

- Unity 版本要求为 `2022.3.62`。
- 初始化命令为 `pwsh ./Scripts/Initialize-Project.ps1`。
- 运行主场景为 `Packages/ET.StateSync/Scenes/Init`，当前实际路径为 `Packages/cn.etetet.statesync/Scenes/Init.unity`。
- 服务端可通过 Unity 菜单启动单进程服务器。
- 服务端独立启动时工作目录应为 Unity 项目根目录，而不是 `Bin`。
- 热重载可通过 Unity 菜单 `ET -> Reload` 或 F7。
- 打包流程包含 HybridCLR 安装、Excel/Luban 导出、行为树/技能/Buff/AI ScriptableObject 导出、F6 编译、HybridCLR Generate All、复制 AOT DLL、YooAssets AssetBundle 构建和最终 Build。

根目录脚本包括：

- `Scripts/Initialize-Project.ps1`
- `Scripts/Launch.ps1`
- `Scripts/Publish.ps1`
- `Scripts/PublishAspire.ps1`
- `Scripts/RunAspire.ps1`
- `Scripts/Export-OpenSource.ps1`
- `Scripts/Verify-FiberConfigSingletonMigration.ps1`

## 当前静态状态观察

- 当前仓库根目录没有发现根级 `ET.sln`。
- 当前发现的解决方案文件有：
  - `Packages/cn.etetet.statesync/ET.sln`
  - `Packages/cn.etetet.lockstep/ET.sln`
  - `Packages/cn.etetet.yiuiluban/DontNet~/luban/src/Luban.sln`
- 按项目规范，正式编译仍应使用初始化或 Unity/Rider 生成后的根级 `ET.sln`，命令为 `dotnet build ET.sln`。
- `Assets/` 根资源较少，主要业务资源集中在 `Packages/cn.etetet.statesync`、`Packages/cn.etetet.map` 和 YIUI 相关包内。
- 大量功能是通过 Package 组合出来的，`cn.etetet.statesync` 是当前主整合 demo 包。
- 部分包描述比实际代码更宽泛，例如 `db` 当前更像脚手架，具体业务能力应以源码和协议为准。

## 功能总览

按最终用户或开发者能感知的能力归纳，当前项目具备以下功能：

1. 分布式 MMO 服务端基础。
2. 服务发现与多进程场景拓扑。
3. Actor Location 路由与实体位置管理。
4. 软路由网络入口。
5. 登录、Gate、Realm、Session 管理。
6. 进入地图与地图实例管理。
7. 跨地图、跨进程 Unit 转移。
8. 地图单位创建、同步和销毁。
9. AOI 视野同步。
10. MMO 移动、停止、转向同步。
11. Recast 寻路。
12. 数值系统。
13. 技能施放、CD、目标选择和技能消耗。
14. Buff 生命周期、叠加、Tick、同步和表现。
15. 行为树运行时与 Unity 编辑器。
16. 战斗/技能/AI 业务行为树节点。
17. 怪物 AI 和宠物 AI。
18. 宠物创建和宠物攻击。
19. 任务接取、进度、提交、放弃和查询。
20. 物品背包、堆叠、移动、整理、丢弃和使用。
21. 装备穿戴、卸下、强化和同步。
22. 成就进度、完成、奖励、分类和统计。
23. 机器人自动化测试和压测基础。
24. 状态同步主 demo。
25. 预测回滚帧同步 demo。
26. YIUI UI 框架。
27. 登录、大厅、Loading、主界面、HUD、ActionBar、Minimap 等 UI 资源。
28. 3D 模型展示、Tips、GM、红点、循环列表、UI 特效等 UI 扩展。
29. YooAssets 资源管理和打包流程。
30. HybridCLR 客户端热更新。
31. Luban/Excel 配置生成。
32. Proto 协议生成。
33. Excel MCP 工具。
34. UnityBridge 编辑器自动化操作。
35. 自动化测试框架。
36. AI Harness 技能路由、规范和分析器约束。

