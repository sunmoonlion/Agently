# 契约卡片：TriggerFlow

## 定位

`TriggerFlow` 是定义，`TriggerFlowExecution` 是一次运行。它处理 signal、state、stream、interrupt 和持久化生命周期。

## 文件

- `agently/core/orchestration/TriggerFlow/TriggerFlow.py`
- `agently/core/orchestration/TriggerFlow/Execution.py`
- `agently/core/orchestration/TriggerFlow/Definition.py`
- `agently/core/orchestration/TriggerFlow/Contract.py`

## 生命周期

```text
definition
  -> create_execution()
  -> open
  -> start / emit / resume
  -> sealed
  -> closed
  -> result or snapshot
```

## 状态分类

| 状态 | 要求 |
|------|------|
| execution state | 可序列化，可保存恢复 |
| runtime resources | live object，不进入持久化 |
| runtime stream | 面向消费者的实时数据 |
| interrupts | 等待外部输入的控制状态 |

## 测试入口

- `tests/test_cores/test_trigger_flow_contract.py`
- `tests/test_cores/test_trigger_flow_execution_state.py`
- `tests/test_cores/test_trigger_flow_pause_resume.py`
- `tests/test_cores/test_trigger_flow_runtime_resources.py`

