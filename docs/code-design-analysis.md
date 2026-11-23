# 当前项目代码设计分析

分析日期：2026-07-02

## 分析范围

本文按项目功能和模块，对当前代码设计做静态分析。分析重点包括：

- 包内代码落点和程序集层次。
- Entity / Component / System / Helper 的职责划分。
- 消息 Handler 组织方式。
- 配置、协议和生成代码的使用方式。
- 异步、`EntityRef<T>`、协程锁和跨 Fiber 调用设计。
- 测试代码和工具代码组织。
- 当前静态状态中的风险点。

本文只基于文件扫描和代表性源码阅读，未运行 Unity、未编译、未执行测试。

## 总体代码组织

项目主要代码位于 `Packages/cn.etetet.*` 下，符合 ET Package 化组织方式。各包通常包含：

- `Scripts/Model/`：运行期模型层，定义 Entity、Component、枚举、事件、配置绑定等。
- `Scripts/Hotfix/`：热更逻辑层，定义 System、Helper、Handler、运行时业务逻辑。
- `Scripts/ModelView/`：客户端表现模型层，承载 GameObject、UI、动画、输入等表现相关 Entity。
- `Scripts/HotfixView/`：客户端表现热更逻辑层，处理 UI、GameObject、动画、特效、相机等表现逻辑。
- `Scripts/Core/`：底层核心代码，主要存在于 `core`、`recast`、`truesync`、`lsentity`、`sourcegenerator` 等基础包。
- `Scripts/Editor/`：Unity Editor 工具代码，例如行为树编辑器、Spell Editor、HybridCLR、UnityBridge、PackageManager。

这种结构把“数据模型、业务逻辑、客户端表现、编辑器工具”拆到不同层，整体符合 ET 的热更新和双端共享设计。

## 包级代码落点

从当前静态扫描看：

- 大部分业务包都有 `packagegit.json` 和 `Scripts/Model/Share/PackageType.cs`。
- 主要业务包的代码均位于 `Packages/cn.etetet.*` 下。
- `cn.etetet.statesync` 是当前主 demo/整合包，更多客户端表现代码集中在 `ModelView` 和 `HotfixView`。
- `cn.etetet.core`、`cn.etetet.recast`、`cn.etetet.unitybridge`、`cn.etetet.yiuiframework`、`cn.etetet.btnode`、`cn.etetet.spell` 是代码量较大的包。

需要注意：

- 多个包缺少包内 `AGENTS.md`，与 harness 中“每个包都有 AGENTS.md”的规范不完全一致。
- 少数包没有标准位置的 `Scripts/Model/Share/PackageType.cs`，例如 `cn.etetet.lsentity`、`cn.etetet.truesync`、部分 YIUI 扩展包。这可能是历史包、纯工具包或第三方风格包，但从规范角度应单独确认。
- 根目录当前没有发现根级 `ET.sln`，只发现 `statesync`、`lockstep` 和 Luban 自身解决方案；正式构建前需要确认初始化或 Unity/Rider 工程生成状态。

## ECS 数据与逻辑分离

项目整体遵循 ET 的 ECS 风格：

- Entity / Component 负责保存状态。
- System 负责生命周期和局部逻辑。
- Helper 负责跨实体、跨组件、跨模块的业务编排。
- Handler 负责协议入口，通常做参数读取、组件获取、调用 Helper/System、填充 response。

以物品系统为例：

- `ItemComponent` 位于 `Scripts/Model/Server/ItemComponent.cs`，只保存 `Capacity` 和 `SlotItems`。
- `ItemComponentSystem` 位于 `Scripts/Hotfix/Server/ItemComponentSystem.cs`，负责 Awake、Destroy、Deserialize，以及查槽位、查物品、设置容量等局部逻辑。
- `ItemHelper` 位于 `Scripts/Hotfix/Server/ItemHelper.cs`，负责添加物品、移除物品、移动堆叠、整理背包、通知客户端等业务流程。
- `C2M_MoveItemHandler` 位于 `Scripts/Hotfix/Server/C2M_MoveItemHandler.cs`，协议入口只调用 `ItemHelper.MoveItem(...)` 并写回错误码。

这种分层有两个明显好处：

- 单个 Entity 不承载业务方法，便于序列化、转移和热重载。
- Helper 聚合消息通知、配置读取和跨组件修改，避免 System 无边界膨胀。

## Component 设计

组件通常通过接口声明生命周期和能力：

- `IAwake`：组件创建初始化。
- `IDestroy`：销毁清理。
- `IDeserialize`：反序列化后重建运行时结构。
- `ITransfer`：跨地图或跨进程转移时需要保留。
- `IUpdate`、`ILateUpdate`：需要帧更新的组件。
- `IScene`：可作为 Scene/Domain 根的对象。

例如 `ItemComponent` 实现了 `IDeserialize` 和 `ITransfer`，说明它既需要在反序列化后重建 `SlotItems`，也需要随玩家 Unit 在地图转移时保留。

组件内部引用 Entity 时大量使用 `EntityRef<T>`，例如：

- 背包槽位 `List<EntityRef<Item>>`。
- AOI Cell 中保存 `EntityRef<AOIEntity>`。
- 行为树输出可以声明 `List<EntityRef<Unit>>`。

这符合 ET 异步安全和实体生命周期设计，避免集合长期持有裸 Entity 引用。

## System 设计

System 采用静态 partial 扩展方法模式：

```csharp
[EntitySystemOf(typeof(ItemComponent))]
public static partial class ItemComponentSystem
{
    [EntitySystem]
    private static void Awake(this ItemComponent self)
    {
        ...
    }
}
```

这种设计的特点：

- 生命周期方法由分析器和框架发现。
- 业务方法以扩展方法形式挂在组件上。
- Entity 本身保持为数据容器。
- System 类可被 source generator 和 analyzer 约束。

项目中 `ItemComponentSystem`、`EquipmentComponentSystem`、`BuffComponentSystem`、`QuestComponentSystem`、`LocationProxyComponentSystem`、`ServiceDiscoveryAgentSystem` 都采用类似方式。

## Helper 设计

Helper 是当前项目里非常重要的业务编排层。典型职责包括：

- 读取配置。
- 修改多个 Entity / Component。
- 触发事件。
- 发送网络消息。
- 做跨服务或跨地图调用。
- 管理业务流程和异常分支。

典型例子：

- `ItemHelper`：背包变更和客户端同步。
- `QuestHelper`：接任务、交任务、更新目标进度、通知客户端。
- `SpellHelper`：施法检查、CD、目标选择、CostNode、Buff 创建。
- `BuffHelper`：Buff 生命周期、叠加规则、Tick、客户端同步。
- `TransferHelper`：申请地图、Location 加锁、Unit 序列化/迁移、解锁。
- `SceneChangeHelper`：客户端场景切换、NavMesh 加载、等待主角创建。

这种 Helper 设计适合 ET 项目，因为很多流程天然跨组件、跨包或跨 Fiber。比把逻辑塞进单个 Component System 更清晰。

## Handler 设计

消息入口主要分为：

- `MessageHandler<Scene, TMessage>`
- `MessageHandler<Scene, TRequest, TResponse>`
- `MessageSessionHandler<TRequest, TResponse>`
- `MessageLocationHandler<TEntity, TRequest, TResponse>`

业务消息通常按场景类型注册：

```csharp
[MessageHandler(SceneType.Map)]
public class C2M_MoveItemHandler : MessageLocationHandler<Unit, C2M_MoveItem, M2C_MoveItem>
```

当前业务 Handler 设计特点：

- Handler 类按消息名命名，入口清晰。
- Handler 文件放在对应包的 `Scripts/Hotfix/{Client|Server}` 下。
- Map 内玩家相关请求多使用 `MessageLocationHandler<Unit, ...>`，通过 Location 路由到 Unit 所在地图。
- Handler 通常薄封装，只做协议适配和调用 Helper/System。

需要注意：

- 很多 Handler override 是 `async ETTask`，但内部只同步处理后 `await ETTask.CompletedTask`。这可能是为了满足框架抽象接口，但从代码设计角度看，应避免在非框架强制场景继续扩散这种“形式异步”。
- 有真实异步资源的 Handler 和 Helper 应严格遵守 `EntityRef<T>` 规则。

## 配置驱动设计

项目强依赖 Luban/Excel 生成配置。当前生成代码和配置表覆盖：

- `ItemConfig`
- `EquipmentConfig`
- `MapConfig`
- `MapUnitConfig`
- `UnitConfig`
- `NumericTypeConfig`
- `QuestConfig`
- `QuestObjectiveConfig`
- `MapTransferRuleConfig`
- `TextConfig`
- `StartMachineConfig`
- `StartProcessConfig`
- `StartSceneConfig`
- `StartZoneConfig`
- `SpellConfig`
- `BuffConfig`

配置驱动在代码中的体现：

- `UnitFactory` 根据 `UnitConfig` 初始化 UnitType、基础数值、坐标和 AI。
- 非玩家 Unit 会叠加 `MapUnitConfig` 中的地图实例数值。
- `NumericType.AI` 不为 0 时，通过 `BuffHelper.CreateBuff(...)` 启动 AI。
- `SpellHelper` 根据 `SpellConfig` 执行 CD、目标选择和 CostNode。
- `BuffHelper` 根据 `BuffConfig` 决定持续时间、叠加规则、通知范围和效果树。
- `QuestHelper` 根据 `QuestConfig` 检查前置任务。
- `TransferHelper` 依赖地图名、地图 ID 和转场规则做地图申请与转移。

这种设计把策划数据和程序逻辑分离，便于扩展玩法。但配置字段的契约很强，新增配置时需要同步 Luban 导出、生成代码、业务读取和测试。

## 协议驱动设计

协议源文件分散在各业务包的 `Proto/` 目录，生成代码汇总到 `cn.etetet.proto/CodeMode`。这种设计有几个优点：

- 协议归属和业务包一致。
- opcode 段从文件名可见，例如 `Login_C_10000`、`Spell_C_10200`、`Quest_C_10400`。
- Client、Server、ClientServer 生成代码可分端使用。
- Handler 和协议命名基本能对应。

消息域包括登录、地图、移动、技能、任务、物品、装备、成就、状态同步、服务发现、ActorLocation、转场、机器人、UnityBridge 等。

## 行为树与技能代码设计

行为树是战斗、条件、技能和 AI 的核心抽象之一。

基础行为树包提供：

- `BTNode`
- `BTRoot`
- `BTComposite`
- `BTSequence`
- `BTSelector`
- `BTNot`
- `BTCondition`
- `BTAction`
- `BTEnv`
- `BTDispatcher`

`BTDispatcher` 使用 `[BTHandler]` 扫描并注册节点 Handler，运行时根据节点类型分发。

业务节点包 `cn.etetet.btnode` 提供大量节点 Handler，例如：

- `BTDamageHandler` 从 `BTEnv` 取 Caster、Target、Buff，再调用 `DamageHelper.Damage(...)`。
- 目标选择节点处理单体、圆形、扇形、矩形、施法者等范围。
- Buff 和技能节点处理加 Buff、删 Buff、创建技能、技能修正、数值消耗。
- 客户端表现节点处理动画参数、特效、施法条和假子弹。
- AI 节点处理怪物/宠物巡逻、追击、跟随、返回。

技能系统复用行为树：

- `SpellConfig.TargetSelector` 是行为树。
- `SpellConfig.Cost` 是行为树。
- Buff 的服务端添加、Tick、命中、移除效果也是行为树。
- 条件表达式最终也被编译成行为树运行。

这种设计的优点是技能、Buff、AI、条件表达式共用一套节点和运行时环境，扩展能力强。风险是节点契约依赖 `BTEnv` 中的字符串 key 和实体类型，配置和代码之间需要测试保障。

## 单位创建与组件装配设计

`UnitFactory` 是当前地图玩法的关键装配点：

- 创建 Unit。
- 根据 `UnitConfig` 设置 UnitType。
- 添加 `NumericComponent` 并灌入配置数值。
- 设置位置和旋转。
- 添加移动、转向、AOI、目标、技能、Buff 等基础组件。
- 玩家添加 `ItemComponent` 和 `QuestComponent`。
- 怪物添加 `ThreatComponent` 和 `PathfindingComponent`。
- 根据 AI 数值创建 Buff 启动 AI。
- 宠物通过 `CreatePet` 绑定 Owner 和 `UnitPetComponent`。

这体现了“组件契约集中装配”的设计。后续业务如果假设玩家一定有背包和任务组件，应该追溯到 UnitFactory，而不是在使用点到处 `GetComponent<T>() ?? AddComponent<T>()`。

## 客户端表现层设计

客户端表现代码主要集中在：

- `ModelView/Client`
- `HotfixView/Client`
- `statesync` 资源包
- YIUI 相关包

典型组件包括：

- `GameObjectComponent`
- `AnimatorComponent`
- `InputSystemComponent`
- `OperaComponent`
- `CinemachineComponent`
- `BindPointComponent`
- `BuffGameObjectComponent`
- `SpellIndicatorComponent`

表现层通过事件和消息同步驱动：

- Unit 创建后创建 GameObject。
- 位置变化同步 GameObject 坐标。
- 旋转变化同步 GameObject 旋转。
- 移动开始/停止修改动画参数。
- 数值变化驱动 UI 或动作表现。
- Spell/Buff 消息驱动特效、施法条、指示器。

这套设计把逻辑实体和 Unity 表现实体分离，符合服务端可运行、客户端可表现的双端架构。

## 异步与 EntityRef 设计

项目异步主要出现在：

- RPC 调用。
- 数据库操作。
- 配置加载。
- 资源加载。
- Timer 等待。
- 协程锁。
- 跨 Fiber 创建和初始化。
- 地图转场和 Unit 迁移。
- 后台注册、心跳、重试循环。
- 测试用例。

较好的设计例子：

- `TransferHelper` 在 `await` 前保存 `EntityRef<Unit>`、`EntityRef<Scene>`、`EntityRef<ServiceDiscoveryProxy>`，`await` 后重新取回。
- `SceneChangeHelper` 在等待主角创建、加载 NavMesh、获取协程锁前后使用 `EntityRef<Scene>` 和 `EntityRef<CurrentScenesComponent>`。
- `LocationProxyComponentSystem.CallPrimaryWithRetry` 使用 `EntityRef<LocationProxyComponent>` 和 `EntityRef<Scene>` 处理重试、等待和组件销毁。
- `ServiceDiscoveryAgentSystem` 后台循环通过 `EntityRef<ServiceDiscoveryAgent>` 防止 await 后访问已释放实体。

这说明项目核心异步流程基本围绕 ET 的实体生命周期安全设计。

需要注意：

- 一些方法只是为了接口返回 `ETTask` 而 `await ETTask.CompletedTask`，应在新增代码时控制扩散。
- `NewContext(...)` 不应承载 Entity 透传，新增独立协程时应使用 `EntityRef<T>` 或稳定 Id。
- 长生命周期集合应继续使用 `EntityRef<T>`，不要新增裸 Entity 字段或集合。

## 服务端权威设计

当前玩法模块普遍采用服务端权威：

- 物品添加、移除、堆叠、整理由服务端决定并推送客户端。
- 装备穿戴和卸下由服务端同步装备更新。
- 任务状态和目标进度由服务端维护并推送。
- 技能施法、CD、Buff 生命周期和伤害由服务端处理。
- 地图转场由服务端申请地图、加锁 Location、转移 Unit。
- AOI 进出视野由服务端通知客户端。

客户端负责请求、展示和本地等待流程，核心状态以服务端为准。这对 MMO 项目是合理设计。

## 测试代码设计

测试框架位于 `cn.etetet.test`，业务包也可以在 `Scripts/Hotfix/Test` 下放测试。

当前测试覆盖较明显的模块：

- `actorlocation`：锁、解锁、超时、主备、重试、缓存、多类型存储等。
- `servicediscovery`：高可用、故障恢复、并发、分区、订阅通知、主节点切换等。
- `conditionexpr`：编译、运行、Luban bean。
- `item`：堆叠、Fiber singleton。
- `spell`：Fiber singleton。
- `core`：Fiber、CoroutineLock、EntityPool。
- `unitybridge`：大量编辑器命令和消息测试。

测试用例继承 `ATestHandler`，通过 `TestFiberScope` 创建隔离测试纤程。测试命名规范由分析器约束。

从代码设计看，服务发现和 ActorLocation 这类基础设施的测试覆盖较强；玩法层如任务、装备、成就的测试覆盖还可以继续补齐。

## 静态状态风险与改进点

### 包规范不完全统一

部分包缺少包内 `AGENTS.md` 或标准位置 `PackageType.cs`。如果后续开发会修改这些包，应先明确它们是历史包、第三方包、纯工具包，还是需要补齐规范。

### 部分业务仍有简化或 TODO

例如任务接取中的等级检查仍是 TODO，成就处理里也有“实际应该从配置表获取”的简化说明。文档和后续开发应把这类能力视为半成品或示例逻辑。

### 形式异步较多

因为框架接口需要 `ETTask`，一些 Handler 和测试使用 `await ETTask.CompletedTask`。这不一定是 bug，但新增代码时应避免把同步逻辑包装成不必要的异步层。

### 配置契约强，需要测试兜底

技能、Buff、AI、条件表达式高度依赖配置、行为树和 `BTEnv` key。配置和代码不一致时可能到运行时才暴露问题，应通过导出校验、编辑器验证和测试覆盖降低风险。

### Helper 权限较大

Helper 是清晰的业务编排点，但也容易膨胀。后续扩展时应保持边界：

- 局部纯数据操作留在 ComponentSystem。
- 跨组件、带通知、跨 Fiber/RPC 的流程放 Helper。
- 大型流程可继续拆分成更具体的 Helper 或服务组件。

### 根工程状态需确认

当前没有根级 `ET.sln`。如果要执行规范要求的 `dotnet build ET.sln`，需要先确认初始化、Unity 工程生成或现有解决方案布局。

## 结论

当前项目代码设计整体符合 ET 的核心风格：包模块化、数据与逻辑分离、配置驱动、协议驱动、服务端权威、客户端表现分层、异步 EntityRef 安全、测试纤程隔离。

最核心的设计主线是：

1. `core` 提供 Entity/Fiber/Actor/Message/Timer/ETTask 等基础。
2. `startconfig`、`servicediscovery`、`netinner`、`actorlocation`、`router` 构成分布式服务基础。
3. `login`、`map`、`mapmanager`、`transfer`、`mapplay` 形成玩家从登录到地图内运行的主流程。
4. `numeric`、`unit`、`aoi`、`move`、`recast` 提供地图实体和移动基础。
5. `behaviortree`、`btnode`、`spell`、`conditionexpr` 构成技能、Buff、AI 和条件判断的可配置运行时。
6. `quest`、`item`、`equipment`、`achievement` 提供 MMO 玩法系统。
7. `statesync`、YIUI、YooAssets 和表现层组件提供客户端 demo 和 UI/资源表现。

后续如果继续扩展功能，建议优先保持现有分层：配置和协议先进包内源文件，数据落 Model，逻辑落 Hotfix System，跨流程进 Helper，表现进 ModelView/HotfixView，异步路径显式使用 `EntityRef<T>`。

