# 调用链 B：Agent 调用 Action

## 目标

跟踪一个本地函数从注册、暴露给模型、被调用到形成 `ActionResult` 的全过程。

## 主链骨架

```text
register_action(function)
  -> ActionRegistry
  -> agent.use_actions(...)
  -> AgentTurn / execution
  -> AgentlyActionRuntime
  -> planning response
  -> normalized action call
  -> ActionDispatcher
  -> LocalFunctionActionExecutor
  -> ActionResult
  -> action logs / next model request
```

## 跟踪表

| 步 | 文件:行 | 输入类型 | 输出类型 | 状态所有者 |
|----|---------|----------|----------|------------|
| 注册 | | function/schema | Action | Registry |
| 规划 | | Prompt | action calls | Runtime |
| 归一化 | | provider call | normalized call | Runtime |
| 分发 | | normalized call | executor | Dispatcher |
| 执行 | | arguments | ActionResult | Executor |
| 回填 | | ActionResult | prompt/log | Execution |

## 关键分叉

- 普通函数是否需要 Execution Environment？
- MCP Action 在哪一步切换 Executor？
- Action 失败是返回结果、重试还是中止？
- `extra.action_logs` 由谁写入？

## 过关

- [ ] 能解释 Runtime、Dispatcher、Executor 的差异。
- [ ] 能新增一个函数 Action，不修改 Core。
- [ ] 能根据日志定位一次失败调用。

