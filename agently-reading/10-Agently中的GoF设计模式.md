# Agently 中的 GoF 设计模式：基于源码的 23 模式审计

> 阅读对象：Agently `4.1.3.5`
>
> 本文不是按照类名猜模式，也不要求在框架里凑齐 GoF 23 种模式。判断依据是：
> **变化点、对象角色、控制权转移和运行时协作**是否与模式定义相符。

## 一、先说结论

Agently 最重要的设计模式不是平均分布的。

真正构成框架骨架的是：

1. **Facade**：`AgentlyMain` 收束整个框架入口。
2. **Strategy**：插件协议和注册表隔离模型、解析、Action、编排等变化。
3. **Adapter**：把供应商协议、Action、Skill 等异构接口转成统一调用契约。
4. **Observer**：`EventCenter` 把运行过程与日志、存储和外部观测解耦。
5. **Builder**：Prompt 和 TriggerFlow 都通过渐进式 API 组装运行描述。
6. **Bridge**：Workspace API 与 Workspace Backend 可以分别变化。
7. **Memento**：TriggerFlow execution 可以保存完整快照并恢复执行。

其次是若干 Python 化或数据化的模式变体：

- Action 是 **Command 的数据化变体**；
- `AttemptRunner` 是 **Template Method 的组合式变体**；
- TriggerFlow 是显式状态机，但不是经典 **State** 对象结构；
- response stream 使用 Python generator 实现 **Iterator**；
- output schema 接近一个小型 DSL，具有 **Interpreter-like** 特征。

最容易误判的地方是：

- 多重继承的 mixin 不是 Decorator；
- 插件注册表不是 Abstract Factory；
- `create_xxx()` 不自动等于 Factory Method；
- Execution Environment 的资源复用是 Object Pool，不是 Flyweight；
- 有状态字段不自动等于 State；
- 有父子流程不自动等于 Composite。

---

## 二、判断等级

| 等级 | 含义 |
|---|---|
| 明确 | 意图、角色和协作结构基本符合 GoF 定义 |
| 变体 | 解决的是同类问题，但借助 Python callable、Protocol 或数据对象实现 |
| 类似 | 局部使用了该模式思想，缺少经典结构中的关键角色 |
| 不认定 | 只有名字或表面形状相似，源码不足以支持该结论 |

---

## 三、23 种模式总表

| 类别 | GoF 模式 | Agently 结论 | 主要证据 |
|---|---|---|---|
| 创建型 | Abstract Factory | 不认定 | PluginManager 是分类注册表，不生产相互匹配的产品族 |
| 创建型 | Builder | 明确/变体 | `AgentTurn` Prompt API、`TriggerFlowProcess` |
| 创建型 | Factory Method | 类似 | `create_agent`、`create_execution` 更接近 Simple Factory |
| 创建型 | Prototype | 类似 | sub-flow blueprint 与状态深拷贝，但无统一 `clone()` 契约 |
| 创建型 | Singleton | 类似 | 模块级 `Agently` 和共享 managers，不限制再次实例化 |
| 结构型 | Adapter | 明确 | ModelRequester adapters、`ActionTaskAdapter`、`SkillTaskAdapter` |
| 结构型 | Bridge | 明确 | `Workspace` + `WorkspaceBackend` |
| 结构型 | Composite | 类似 | TriggerFlow 可嵌套 sub-flow，但叶子与组合对象接口不统一 |
| 结构型 | Decorator | 不认定 | Agent mixin/extension 是静态组合与钩子，不是对象包装链 |
| 结构型 | Facade | 明确 | `AgentlyMain` |
| 结构型 | Flyweight | 不认定 | Execution Environment 是带生命周期的资源池 |
| 结构型 | Proxy | 明确/局部 | `_DeprecatedActionManagerProxy` |
| 行为型 | Chain of Responsibility | 类似 | validator/prefix/finally handler pipeline |
| 行为型 | Command | 变体 | `ActionCall` + `ActionDispatcher` + `ActionExecutor` |
| 行为型 | Interpreter | 类似 | output schema DSL 到 Prompt/Pydantic contract 的解释 |
| 行为型 | Iterator | 明确/语言原生 | sync/async response generators、runtime stream |
| 行为型 | Mediator | 类似 | `ActionDispatcher` 集中协调多个 Action 子系统 |
| 行为型 | Memento | 明确 | `TriggerFlowExecutionPersistence.save/load` |
| 行为型 | Observer | 明确 | `EventCenter` + hook registrations/hookers |
| 行为型 | State | 类似 | TriggerFlow 的字符串状态机，不是状态对象多态 |
| 行为型 | Strategy | 明确 | Plugin Protocol、Session resize、Skills effort strategy |
| 行为型 | Template Method | 变体 | `AttemptRunner` 固定重试骨架 + 注入 handlers |
| 行为型 | Visitor | 不认定 | 没有稳定 element hierarchy 与 `accept(visitor)` 双分派 |

---

# 四、创建型模式

## 1. Abstract Factory：不认定

### GoF 要解决什么

抽象工厂提供一个创建相关产品族的接口。例如同一套 UI 主题下，工厂同时创建匹配的
Button、Menu 和 Dialog。

### 容易误判的 Agently 结构

`PluginManager` 按类型注册：

- `ModelRequester`
- `PromptGenerator`
- `ResponseParser`
- `ActionRuntime`
- `ActionFlow`
- `ActionExecutor`
- `ExecutionEnvironmentProvider`
- `AgentOrchestrator`

见：

- `agently/core/extension/PluginManager.py`
- `agently/_default_init.py`

### 为什么不认定

这些插件可以独立激活，并不由一个工厂一次性生产一套必须相互匹配的产品。

```text
PluginManager
  ├─ registry["ModelRequester"][name]
  ├─ registry["ResponseParser"][name]
  └─ registry["ActionFlow"][name]
```

它是 **Registry + Strategy selection**，不是 Abstract Factory。

### 训练意义

看到“很多接口、很多实现、统一注册”时，先问：

> 它是在选择独立策略，还是在创建一组必须配套的产品？

Agently 属于前者。

---

## 2. Builder：明确的 Python 变体

Agently 中有两类 Builder。

### 例一：Prompt/Request 的渐进组装

`AgentTurn` 暴露：

```python
turn = (
    agent.create_turn()
    .input({"question": "什么是策略模式？"})
    .info({"audience": "Python developer"})
    .instruct("给出源码级解释")
    .output({"summary": str, "examples": [str]})
)

result = await turn.async_start()
```

源码入口：

- `agently/core/AgentTurn.py`
  - `input()`
  - `info()`
  - `instruct()`
  - `output()`
  - `create_execution()`

这些方法逐步修改内部 `request.prompt`，最后由 `create_execution()` 或
`async_start()` 把描述转换为可执行对象。

对应角色：

```text
Builder         AgentTurn / ModelRequest fluent API
Product state   Prompt
Build trigger   create_execution / get_response / async_start
```

它与经典 Builder 的差异是：没有独立 Director，Builder 本身也兼任请求门面。

### 例二：TriggerFlow 的流程定义 Builder

```python
flow = TriggerFlow(name="review")

pipeline = (
    flow
    .to(load_document)
    .to(analyze_document)
    .match()
    .case("approved")
    .to(publish)
    .case_else()
    .to(request_revision)
    .end_match()
)
```

源码入口：

- `agently/core/orchestration/TriggerFlow/TriggerFlow.py`
- `agently/core/orchestration/TriggerFlow/process/BaseProcess.py`
- `agently/core/orchestration/TriggerFlow/process/MatchCaseProcess.py`
- `agently/core/orchestration/TriggerFlow/process/ForEachProcess.py`

`when()`、`to()`、`to_sub_flow()`、`match()`、`case()`、`for_each()` 不立即完成整个
业务，而是不断向 `TriggerFlowBlueprint.definition` 注册 operator、signal 和 handler。

### 为什么 Builder 很重要

用户面对的是声明式组装 API；真正的执行结构藏在 Blueprint 中。阅读 TriggerFlow 时，
必须把“定义阶段”和“执行阶段”分开。

---

## 3. Factory Method：只有相似，不是经典实现

### 相关入口

`AgentlyMain` 提供：

```python
Agently.create_prompt()
Agently.create_request()
Agently.create_dynamic_task(...)
Agently.create_agent(...)
```

`TriggerFlow` 提供：

```python
flow.create_execution(...)
```

`WorkspaceManager` 提供：

```python
workspace = manager.create(path_or_backend)
```

### 为什么更接近 Simple Factory

经典 Factory Method 的关键点是：父类定义创建接口，子类重写创建方法决定具体产品。

Agently 这些方法通常直接选择并实例化具体类：

```text
AgentlyMain.create_agent -> self._agent_factory(...)
WorkspaceManager.create  -> LocalWorkspaceBackend or supplied backend
TriggerFlow.create_execution -> blueprint.create_execution(...)
```

其中有可注入 factory，也有注册表查找，但没有稳定的 Creator 子类层次。因此更准确的说法是：

- Simple Factory；
- Registry-backed Factory；
- 可注入对象创建函数。

### 实际价值

虽然不是经典 Factory Method，它仍把“怎么创建”从调用方移走，使默认 Settings、
PluginManager 和上下文能够自动注入。

---

## 4. Prototype：局部相似

### 源码证据

TriggerFlow 创建隔离子流程时：

```text
instantiate_isolated_sub_flow()
  -> trigger_flow.save_blueprint()
  -> 创建新的 TriggerFlow
  -> deepcopy(settings)
  -> deepcopy(flow_data)
  -> deepcopy(runtime_resources)
```

见 `agently/core/orchestration/TriggerFlow/BluePrintSubFlow.py`。

### 为什么只是相似

它确实从现有对象状态派生新对象，但：

- 没有公共 `clone()` / `prototype()` 契约；
- 复制逻辑只服务于 sub-flow 隔离；
- Blueprint、Settings、flow data 的复制规则由调用方明确编排。

因此这是 **copy-based instantiation**，具有 Prototype 思想，但不是框架级 Prototype 模式。

### 不应使用的错误证据

`ModelResponse` 在创建时保存 Prompt 和 Settings 快照，是请求隔离，不是 Prototype。
“发生了复制”不等于“通过原型创建对象”。

---

## 5. Singleton：模块级共享实例

### 源码结构

`agently/base.py` 在模块加载时创建：

```text
settings
plugin_manager
event_center
action
execution_environment
policy_approval
skills_executor
workspace
```

`agently/__init__.py` 再创建全局 `Agently = AgentlyMain()`。

### 为什么不是经典 Singleton

经典 Singleton 通常保证一个类只能产生一个实例。Agently 没有禁止：

```python
another = AgentlyMain()
```

因此它是 Python 常见的 **module singleton / shared application context**，不是强制单例。

### 影响

优点是零配置入口简单；代价是：

- 测试需要恢复全局 Settings；
- 注册插件会影响共享状态；
- 多租户场景必须注意配置边界。

---

# 五、结构型模式

## 6. Adapter：明确

### 例一：模型供应商 Adapter

统一协议位于：

- `agently/types/plugins/ModelRequester.py`

具体适配器包括：

- `OpenAICompatible`
- `AnthropicCompatible`
- `OpenAIResponsesCompatible`

它们把两个方向的数据进行转换：

```text
Agently Prompt/Settings
        |
        v
generate_request_data()
        |
        v
Provider-specific request
        |
        v
broadcast_response()
        |
        v
Agently event stream
```

同一个类同时体现：

- Strategy：运行时可以替换供应商实现；
- Adapter：供应商协议被转换为框架协议。

### 例二：DynamicTask Adapter

`agently/core/orchestration/DynamicTask/DynamicTask.py` 明确定义：

```python
class ActionTaskAdapter:
    async def __call__(self, context):
        ...

class SkillTaskAdapter:
    async def __call__(self, context):
        ...
```

Task DAG executor 只需要统一的 `DynamicTaskHandler(context)`；Adapter 分别把它转换为：

```text
Action-like object -> async_call_action(action_id, kwargs)
Skills executor    -> async_run_skills_task(task, skills=[...])
```

这是非常标准的 Adapter 使用场景。

---

## 7. Bridge：Workspace 与 Backend

### 抽象与实现

抽象侧：

- `agently/core/session/Workspace/Workspace.py`

实现协议：

- `agently/types/plugins/Workspace.py::WorkspaceBackend`

具体实现：

- `agently/core/session/Workspace/LocalBackend.py::LocalWorkspaceBackend`

协作结构：

```text
Client
  |
  v
Workspace                  <- Abstraction
  |
  +-- backend: WorkspaceBackend
                    |
                    +-- LocalWorkspaceBackend
                    +-- future remote/database backend
```

`Workspace.put/get/search/link/checkpoint()` 保持稳定，并把存储操作委托给 backend。

### 为什么是 Bridge

两个维度可以独立变化：

1. Workspace 高层能力可以增加，如 `ingest()`、`build_context()`。
2. Backend 可以替换本地文件 + SQLite，也可以实现远程存储。

客户端依赖 Workspace 抽象，不需要知道实际存储结构。

### 与 Adapter 的区别

Backend 不是为了临时兼容一个既有外部接口，而是从设计开始就作为独立实现维度存在，
所以这里 Bridge 比 Adapter 更准确。

---

## 8. Composite：TriggerFlow 只有部分相似

### 相似点

父 TriggerFlow 可以通过 `to_sub_flow()` 嵌入子 TriggerFlow：

```python
parent.to(prepare).to_sub_flow(child).to(finish)
```

框架负责：

- 为子流程创建隔离实例；
- capture 父流程输入、状态和资源；
- 启动 child execution；
- 桥接 child runtime stream；
- write back 子流程结果；
- 投影暂停状态并支持恢复。

### 缺少的经典结构

Composite 要求 Leaf 与 Composite 共享统一 Component 接口，使客户端可以一致对待单个对象和组合对象。

Agently 中：

- 普通节点主要是 `TriggerFlowChunk`；
- 子流程是 `TriggerFlow`；
- Blueprint 用不同逻辑注册 chunk 与 sub-flow；
- 二者没有统一的 `execute()` Component 契约。

因此它是 **层次化工作流组合**，但不应直接认定为经典 Composite。

---

## 9. Decorator：不认定

### 容易误判一：ModelRequester mixin

例如 `OpenAICompatible` 由多个 mixin 组成：

```text
HandlersMixin
RequestBuilderMixin
TransportMixin
CredentialMixin
ResponseAdapterMixin
ModelRequester
```

这是类定义时的静态组合。Decorator 则要求 wrapper 与 wrapped object 共享接口，并在运行时层层包装。

### 容易误判二：Agent extensions

Agent 通过多重继承和 extension handlers 获得 Session、Action、Skills、Workspace 等能力。
这些 extension 会注册 Prompt prefix、response finally 等钩子，但没有形成：

```text
Decorator(Decorator(Component))
```

### 结论

Agently 大量使用：

- mixin；
- callback hook；
- extension registration；

但这些不能统称为 GoF Decorator。

---

## 10. Facade：明确

### 入口

`agently/base.py::AgentlyMain` 把以下子系统收束为统一 API：

```text
Settings
PluginManager
EventCenter
Action
ExecutionEnvironment
PolicyApproval
SkillsExecutor
WorkspaceManager
Agent / Request / Prompt / DynamicTask factories
```

用户通常只需要：

```python
from agently import Agently

agent = Agently.create_agent()
request = Agently.create_request()
task = Agently.create_dynamic_task("分析项目")
```

### 运行意义

Facade 不实现所有底层逻辑，它负责：

- 暴露高层入口；
- 注入共享 Settings 和 PluginManager；
- 隐藏默认插件与运行时组装；
- 保留高级用户直接访问子系统的可能。

### 阅读提醒

读到 `AgentlyMain` 后应立即下钻到被委托对象。只读 Facade 会知道“有什么”，
但不会知道“运行时怎么发生”。

---

## 11. Flyweight：不认定

### 最像的结构

`ExecutionEnvironment` 会按 provider、scope、owner 和 reuse key 复用资源 handle，
并维护 `ref_count`、健康检查和释放。

这解决的是：

- 浏览器、容器、解释器等资源创建成本高；
- 多次 Action 调用需要复用活资源；
- 最后一个使用者离开后才释放。

### 为什么不是 Flyweight

Flyweight 通过共享不可变的 intrinsic state 支撑大量细粒度逻辑对象。

Execution Environment 共享的是：

- 有身份的外部资源；
- 带健康状态和生命周期的 handle；
- 需要引用计数和释放的昂贵对象。

它更准确属于 **Object Pool / Resource Pool**，不是 GoF Flyweight。

---

## 12. Proxy：明确但使用范围局部

### 明确证据

`agently/core/execution/Action/Action.py` 定义：

```python
class _DeprecatedActionManagerProxy:
    def __getattr__(self, item):
        warn_deprecated_once(...)
        return getattr(self._action, item)
```

Action 把旧接口：

```text
action_manager
tool_manager
```

绑定为代理。访问旧对象时，代理先执行弃用控制，再把请求转发给真实 `Action`。

对应角色：

```text
Subject       Action public methods
RealSubject   Action
Proxy         _DeprecatedActionManagerProxy
Client        legacy code using action_manager/tool_manager
```

### `AgentTurn.__getattr__` 怎么看

`AgentTurn` 也会把未知属性委托给 Agent，但它的主要职责是一次 turn 的 Prompt 和执行组装。
这属于 delegation/proxy-like convenience，不是最有说服力的 Proxy 证据。

---

# 六、行为型模式

## 13. Chain of Responsibility：更接近 Pipeline

### 相关处理链

Model request/response 过程包含：

```text
request prefixes
  -> requester
  -> response parser
  -> materialize
  -> output constraint
  -> ensure
  -> validate handlers
  -> finally handlers
```

多个 validator 会按注册顺序执行；任一步失败可能触发 retry。

### 为什么不是典型 CoR

经典责任链中，每个 Handler 决定：

- 自己处理并停止；
- 或把请求传给下一个 Handler。

Agently 的大多数处理器没有持有 `next`，也不能自由改变链路。顺序和继续条件由
`ModelResponseDataFlow` 等中心对象控制。

因此更准确的结论是：

> 这是 ordered processing pipeline，具有 Chain of Responsibility 的解耦思想。

### 阅读价值

排查输出校验时，不应只找一个 handler；要沿着中心调度者确定：

1. handler 注册顺序；
2. 哪一步物化数据；
3. ensure 与 validate 谁先执行；
4. 异常在哪里转换为 retry。

---

## 14. Command：Action 的数据化变体

### 角色映射

类型位于 `agently/types/data/action.py`：

- `ActionSpec`
- `ActionCall`
- `ActionResult`

执行位于：

- `ActionDispatcher`
- `ActionExecutor` Protocol
- Local、MCP、Python、Bash、Browser 等 concrete executors

调用过程：

```text
Client / model decision
        |
        v
ActionCall {action_id, kwargs, call_id, ...}
        |
        v
ActionDispatcher
  -> resolve ActionSpec
  -> resolve ActionExecutor
  -> merge policy and approval
  -> ensure execution environments
  -> executor.execute(call)
  -> normalize ActionResult
```

### 为什么是变体

经典 Command 常把 `execute()` 放在 command object 上。Agently 的 `ActionCall`
是 TypedDict 数据，执行行为在 `ActionExecutor` 中。

这是一种适合分布式和可观测系统的 **data command**：

- 调用可以序列化；
- 调度、策略和执行后端分离；
- 结果可以统一归一化；
- Command 本身保持纯数据。

---

## 15. Interpreter：output schema DSL 的相似实现

### 被解释的“语言”

用户可以用 Python 值描述输出契约，例如：

```python
request.output({
    "title": (str, "文章标题"),
    "items": [{
        "name": str,
        "score": int,
    }],
})
```

PromptGenerator 会理解：

- dict：对象字段；
- list：列表元素；
- tuple：类型与说明；
- Python/Pydantic type：类型约束；
- ensure keys 和 output format：额外语义。

然后生成：

1. 给模型阅读的格式说明；
2. 结构化输出模型；
3. parser/validation 所需契约。

源码入口：

- `agently/types/plugins/PromptGenerator.py`
- `agently/builtins/plugins/PromptGenerator/AgentlyPromptGenerator.py`
- `agently/core/model/Prompt.py`

### 为什么只是 Interpreter-like

它确实解释一套小型声明语言，但没有 GoF 示例中常见的 Expression 类层次和
`interpret(context)` 递归对象树。这里更多依赖 Python 容器和类型作为 AST。

因此可以把它理解为 **schema DSL interpreter**，但不宜声称是标准 Interpreter 实现。

---

## 16. Iterator：Python 原生实现

### 源码表现

Model response 同时支持：

- 同步 generator；
- 异步 generator；
- instant/streaming data；
- runtime event stream。

供应商 adapter 也不断 `yield` 统一事件：

```text
message
delta
tool_calls
reasoning_delta
done
meta
error
```

### 为什么可以认定

Iterator 的核心目的是：不暴露聚合对象内部表示，而提供顺序访问协议。

调用方可以：

```python
async for item in response.get_async_generator():
    consume(item)
```

调用方不知道 HTTP SSE、供应商 chunk、parser buffer 和 retry 的内部结构。

### Python 化差异

Agently 不需要手写 `has_next()/next()` 类，因为 Python generator 和 async generator
已经是语言级 Iterator 协议实现。

---

## 17. Mediator：ActionDispatcher 只有类似性

### 中央协调行为

`ActionDispatcher.async_execute()` 同时协调：

- `ActionRegistry`
- Action spec
- Action executor
- policy/approval
- execution environment
- artifact and result normalization
- runtime observation

它显著减少了调用方与这些对象之间的直接耦合。

### 为什么不判为明确

经典 Mediator 强调多个 colleague 通过 mediator 相互协作，colleague 不直接引用彼此。

Action 子系统更像：

- Application Service；
- Dispatcher；
- Orchestrator。

各组件并不都把 Dispatcher 当作通信中介。因此可使用 Mediator 视角理解集中协调，
但源码角色不够完整。

---

## 18. Memento：明确

### 角色

```text
Originator   TriggerFlowExecution
Memento      save() 返回的 state dict / JSON / YAML
Serializer   TriggerFlowExecutionPersistence
Caretaker    调用 save/load 的应用或存储系统
```

源码：

- `agently/core/orchestration/TriggerFlow/Execution.py`
- `agently/core/orchestration/TriggerFlow/ExecutionPersistence.py`

### 快照包含什么

`save()` 保存：

- execution id、status、lifecycle；
- runtime data、flow data；
- run context；
- interrupts 和 interventions；
- sub-flow frames；
- last signal；
- result readiness/value；
- lease、owner、时间戳和 state version；
- 需要重新注入的 resource keys。

`load()` 恢复这些字段，并重建：

- result event；
- closed/open 状态；
- execution registry 映射；
- pause/resume 所需上下文；
- auto-close monitor。

### 示例

```python
execution = flow.create_execution()
await execution.async_start(initial_input)

snapshot = execution.save()

restored = flow.create_execution()
restored.load(snapshot, runtime_resources={"service": service})
await restored.async_continue_with(interrupt_id, approved_payload)
```

### 重要边界

快照只保存可序列化状态。数据库连接、浏览器 page、函数和 service client 等运行资源
不能作为 Memento 内容可靠恢复，必须通过 `runtime_resources` 重新注入。

---

## 19. Observer：明确

### 角色

```text
Subject/Publisher   EventCenter
Observer            registered hook callback / EventHooker plugin
Event               ObservationEvent / RuntimeEvent
```

源码：

- `agently/core/runtime/EventCenter.py`
- `agently/_default_init.py`
- `agently/builtins/hookers/`

### 运行过程

```text
ModelResponse / Action / TriggerFlow
          |
          v
event_center.async_emit(event)
          |
          +-- filter registration
          +-- callback observer
          +-- console hooker
          +-- storage hooker
          `-- external observation bridge
```

事件发布者不需要知道最终是打印、存储还是发送到外部系统。

### 与 Mediator 的区别

`EventCenter` 的主语义是“一对多通知”，所以 Observer 是准确结论。
它不是为了让多个业务 colleague 互相请求协作，因此不应主要标为 Mediator。

---

## 20. State：显式状态机，不是经典 State 对象

### 源码中的状态

TriggerFlow 有两组状态：

执行状态：

```text
created -> running -> waiting/completed/failed/cancelled
```

生命周期状态：

```text
open -> sealed -> closed
```

定义位于 `agently/core/orchestration/TriggerFlow/Control.py`。

`TriggerFlowExecution` 根据状态决定：

- 是否接受 signal；
- 是否可以 pause/resume；
- 是否继续调度；
- 是否允许 close；
- 是否启动 auto-close；
- save/load 后如何恢复事件与任务。

### 为什么不是经典 State

经典 State 通常是：

```text
Context.state: State
State.handle(context)
  ├─ CreatedState
  ├─ RunningState
  └─ ClosedState
```

Agently 使用字符串常量和条件分支，没有 `CreatedState/RunningState/...` 多态对象。

所以准确结论是：

> TriggerFlow 有成熟的 finite-state machine，但只体现 State 模式意图。

这一区分很重要，因为扩展一个新状态时，开发者需要搜索所有状态判断，而不是只新增一个 State 类。

---

## 21. Strategy：Agently 最核心的模式

### 第一层：Plugin Strategy

协议位于 `agently/types/plugins/`：

- `ModelRequester`
- `PromptGenerator`
- `ResponseParser`
- `ActionRuntime`
- `ActionFlow`
- `ActionExecutor`
- `ExecutionEnvironmentProvider`
- `AgentOrchestrator`
- `SkillsExecutor`
- `TaskDAGPlanner`

`PluginManager.register()` 按插件类型和名称保存实现，并通过：

```text
plugins.<PluginType>.activate
```

选择激活策略。

### 第二层：函数级 Strategy

Session 注册 resize strategy：

```python
session.register_resize_handler("summarize", summarize_handler)
```

Skills 注册 effort strategy：

```python
skills.register_effort_strategy("audit_plus", audit_handler)
```

内置 Skills effort strategies 包括：

- `single_shot`
- `runtime_chain`
- `staged`
- `react`

### 运行示例

```text
ModelResponse
  -> settings 读取 active ModelRequester 名称
  -> PluginManager 获取 concrete class
  -> concrete strategy 生成请求并广播响应

Session
  -> analysis handler 返回 strategy name
  -> _resize_handlers[strategy_name](...)

Skills
  -> effort name
  -> builtin/custom handler
  -> 执行对应策略
```

### 为什么它是框架主轴

Agently 的主要扩展方式不是继承一个巨大基类，而是：

1. 遵守 Protocol；
2. 注册实现；
3. 通过 Settings 或运行参数选择；
4. 由稳定 Context 调用。

这正是 Strategy 所解决的“算法和实现可替换”问题。

---

## 22. Template Method：组合式变体

### 核心例子：AttemptRunner

`agently/core/runtime/AttemptRunner.py` 固定执行骨架：

```text
初始化 AttemptState
  -> execute(state)
  -> 遍历 stream
  -> observe(item, state)
  -> 成功结束
  -> 发生异常
  -> handle_error(error, state)
  -> retry / raise / yield_error / stop
```

变化步骤通过 handlers 注入：

- `execute`
- `observe`
- `handle_error`

### 为什么是变体

经典 Template Method 通常由基类方法定义算法，子类覆盖 protected hook：

```python
class Base:
    def template_method(self):
        self.step1()
        self.step2()
```

Agently 没有要求继承 `AttemptRunner`，而是传入 callable handlers。

因此它是：

> Template Method 的控制流思想 + Strategy 的组合实现。

这种写法更适合 Python，也避免为了三个步骤制造子类。

---

## 23. Visitor：不认定

### 搜索结论

在 TriggerFlow definition、Task DAG、Prompt schema 和 Action spec 中，确实存在复杂数据结构遍历，
但没有发现稳定的：

```text
Element.accept(visitor)
Visitor.visit_xxx(element)
```

也没有依靠双分派把多个操作从 element hierarchy 中移出的结构。

### 容易误判的地方

- PromptGenerator 递归处理 schema：更接近 Interpreter/transformer。
- Mermaid 生成遍历 TriggerFlow definition：只是普通遍历。
- Response parser 遍历数据流：是 parser/pipeline。

因此 Visitor 不认定。

---

# 七、把模式放回真实执行链

只记模式名称没有意义。下面把它们放回两条最重要的运行链。

## 1. Model Request 执行链

```text
AgentlyMain                                  Facade
  -> create_agent/create_request             Simple Factory-like
  -> AgentTurn.input/info/instruct/output    Builder
  -> AgentOrchestrator                       Strategy
  -> ModelResponse
       -> request prefix pipeline            CoR-like Pipeline
       -> ModelRequester                     Strategy
            -> OpenAI/Anthropic/...          Adapter
            -> provider event generator      Iterator
       -> AttemptRunner                      Template Method variant
       -> ResponseParser                     Strategy
       -> constraint/ensure/validate         Pipeline
       -> EventCenter                        Observer
```

这条链解释了为什么官方文档通常重点讲 Prompt 如何组装、如何选择模型、如何拿到结果：
它主要覆盖 Builder 和执行链的入口。真正的扩展契约则藏在 Protocol、PluginManager 和默认实现中。

## 2. Action 执行链

```text
Model/Client decision
  -> ActionCall                              Command data
  -> ActionDispatcher                       Dispatcher/Mediator-like
       -> ActionRegistry
       -> policy + approval
       -> ExecutionEnvironment pool
       -> ActionExecutor                    Strategy
            -> Local/MCP/Sandbox/...         Adapter/backend implementation
       -> ActionResult normalization
       -> EventCenter                       Observer
```

这里最重要的边界是：

- `ActionCall` 描述“要做什么”；
- `ActionDispatcher` 决定“允许不允许、交给谁、需要哪些环境”；
- `ActionExecutor` 负责“实际执行一次”；
- `ActionFlow` 决定“多轮 Action 如何循环和恢复”。

---

# 八、按六层阅读法应该怎样使用本文

## ① 意图层

先从 README 和模块注释确认 Agently 的核心意图：

- 统一模型访问；
- Agent/Action/Skills/Workflow 组合；
- 可观测、可暂停、可恢复；
- 通过插件协议扩展。

此时只记录问题域，不急着贴模式。

## ② 边界层

重点看：

- `agently/base.py`
- `agently/__init__.py`
- `agently/core/`
- `agently/types/plugins/`
- `agently/builtins/plugins/`

这一层最容易识别 Facade、模块级 Singleton 和子系统边界。

`pyproject.toml` 的价值不是告诉你执行细节，而是确认：

- 包名与版本；
- Python 版本；
- 依赖能力；
- 可选 integration；
- 测试和工具入口。

看不出业务流程是正常的，它是边界证据，不是执行层文档。

## ③ 契约层

优先读 `agently/types/plugins/` 中的 Protocol。

这一层回答：

> 替换一个 ModelRequester、ActionExecutor、WorkspaceBackend，最少必须实现什么？

这里主要识别 Strategy、Adapter 和 Bridge。

## ④ 组装层

重点看：

- `agently/_default_init.py`
- `PluginManager.register()`
- `AgentlyMain.create_xxx()`
- `Action.__init__()`
- `DynamicTask.__init__()`
- TriggerFlow fluent definition API

这里识别 Registry、Builder、Factory-like 和 Facade。

## ⑤ 执行层

沿实际入口向下跟：

```text
AgentTurn.async_start
  -> create_execution
  -> AgentOrchestrator
  -> ModelResponse
  -> ModelRequester
  -> ResponseParser
```

以及：

```text
ActionDispatcher.async_execute
  -> policy
  -> environment
  -> executor.execute
  -> normalize result
```

这里识别 Command、Iterator、Template Method、Observer 和 Pipeline。

## ⑥ 扩展层

最后对照官方实现：

- OpenAI/Anthropic ModelRequester；
- Local/MCP/Sandbox ActionExecutor；
- ExecutionEnvironmentProvider；
- Skills effort strategies；
- LocalWorkspaceBackend。

不要只看 Protocol。默认实现会告诉你：

- 错误如何处理；
- 生命周期由谁负责；
- 事件应该在哪里发；
- 哪些字段必须归一化；
- 什么逻辑属于 adapter，什么逻辑必须留在 core。

---

# 九、最终认识

Agently 的架构并不是“GoF 23 种模式展览”。它更像一个围绕稳定契约建立的运行时：

```text
Facade 提供入口
  -> Builder 形成运行描述
  -> Registry 选择 Strategy
  -> Adapter 接入异构实现
  -> Dispatcher/Runner 控制执行
  -> Iterator 输出流
  -> Observer 发布过程
  -> Memento 保存可恢复状态
```

如果只记一个阅读结论，应当是：

> Agently 最核心的设计不是继承层次，而是
> `Protocol + Registry + Settings selection + runtime composition`。

GoF 模式在这里最有用的地方，不是给类命名，而是帮助我们追问：

1. 哪个变化被隔离了？
2. 谁拥有控制权？
3. 哪个对象只描述数据，哪个对象真正执行？
4. 扩展时应实现 Protocol、注册 Strategy，还是包装现有对象？
5. 当前结构是经典模式，还是 Python 语境下更轻量的变体？

回答完这些问题，才算真正读懂了模式在 Agently 中的作用。
