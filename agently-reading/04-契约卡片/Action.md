# 契约卡片：Action

## 定位

Action 描述模型可调用的原子能力。Action Registry、Dispatcher、Runtime、Executor 和 Execution Environment 各自负责不同阶段。

## 文件

- `agently/core/execution/Action/Action.py`
- `agently/core/execution/Action/ActionRegistry.py`
- `agently/core/execution/Action/ActionDispatcher.py`
- `agently/builtins/plugins/ActionRuntime/`
- `agently/builtins/plugins/ActionExecutor/`

## 分工

| 对象 | 回答的问题 |
|------|------------|
| Action schema | 模型能调用什么？ |
| Registry | 已注册哪些 Action？ |
| Dispatcher | 这次调用交给谁？ |
| Executor | 原子调用如何执行？ |
| Runtime | 模型如何规划、循环和结束？ |
| Execution Environment | 调用前需要什么长期资源？ |

## 最小扩展

优先注册普通 Python function Action；只有执行方式不同才新增 Executor，只有存在资源生命周期才新增 Environment Provider。

## 测试入口

- `tests/test_cores/test_builtin_actions_v2.py`
- `tests/test_extensions/test_action_extension.py`
- `tests/test_plugin_protocol_contracts.py`

