# 调用链 C：TriggerFlow 生命周期

> 进阶链。完成请求链和 Action 链后再读。

## 最小 API

```python
flow = TriggerFlow(name="hello")
flow.to(handler)
snapshot = await flow.async_start("World")
```

## 主链骨架

```text
TriggerFlow definition
  -> to(handler)
  -> create_execution()
  -> async_start(initial_value)
  -> signal scheduling
  -> handler receives runtime data
  -> state / stream / emitted signals
  -> seal and drain
  -> close snapshot
```

## 必须区分

- Definition 与 Execution
- state 与 runtime resource
- signal 与 runtime stream
- pause 与 process termination
- save/load state 与 Blueprint

## 证据

优先从以下测试反推：

- `test_trigger_flow_contract.py`
- `test_trigger_flow_execution_state.py`
- `test_trigger_flow_pause_resume.py`
- `test_trigger_flow_execution_result.py`

