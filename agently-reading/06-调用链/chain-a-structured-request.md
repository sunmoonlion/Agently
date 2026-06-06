# 调用链 A：结构化请求

## 目标 API

```python
agent.input("分析反馈").output({"category": (str, "分类", True)}).start()
```

## 主链

| 步 | 文件 | 符号 | 输入 | 输出 |
|----|------|------|------|------|
| 1 | `core/Agent.py` | quick prompt API | Agent + value | AgentTurn |
| 2 | `core/AgentTurn.py` | `input` / `output` | slot values | request Prompt |
| 3 | `core/AgentTurn.py` | `start` | execution options | parsed data |
| 4 | AgentOrchestrator plugin | `create_execution` | AgentTurn | execution |
| 5 | `core/model/ModelRequest.py` | `get_response` | Prompt | ModelResponse |
| 6 | `core/model/ModelResponse.py` | `_get_response_generator` | snapshot | message stream |
| 7 | ModelRequester plugin | request methods | prompt/settings | provider stream |
| 8 | `ModelResponseResult.py` | data readers | stream | validated object |

## 数据形变

```text
Python prompt values
  -> Prompt slots
  -> PromptObject
  -> messages + request options
  -> provider request data
  -> response message stream
  -> text fragments
  -> parsed structured data
  -> ensure / validate
  -> dict
```

## 观测链

至少验证：

- `agent_turn.started`
- `request.started`
- `model.request_started`
- `prompt.built`
- `model.requesting`
- response / validation 相关事件

## 待跟栈

- [ ] 补全准确文件与行号。
- [ ] 确认 retry 的控制所有者。
- [ ] 记录 Prompt snapshot 前后的对象 identity。
- [ ] 使用测试替身运行，无真实 API key。

