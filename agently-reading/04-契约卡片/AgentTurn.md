# 契约卡片：AgentTurn

## 定位

`AgentTurn` 是一次 Agent 请求的独立 Prompt 草稿，也是链式 API 的主要承载者。

## 文件

- `agently/core/AgentTurn.py`
- `agently/core/Agent.py`

## 持有状态

| 字段 | 含义 |
|------|------|
| `_agent` | 能力与长期配置所有者 |
| `request` | 本轮 `ModelRequest` |
| `request_prompt` | 本轮 Prompt |

## 核心行为

| 方法 | 作用 |
|------|------|
| `input`、`info`、`instruct` | 写入本轮 Prompt slot |
| `output` | 声明输出 schema 和格式 |
| `start` / `async_start` | 创建 execution 并取得解析数据 |
| `get_response` | 返回可多方式消费的响应对象 |
| `create_execution` | 通过 `AgentOrchestrator` 插件创建执行 |

## 关键不变量

默认链式 Prompt 写入本轮 request；`always=True` 才写回 Agent 级配置。

## 待验证

- [ ] 同一个 Agent 连续两次 `.input(...).start()` 是否完全隔离。
- [ ] `__getattr__` 委托 Agent 会不会造成 API 所有权误判。
- [ ] one-turn orchestrator 如何消费本轮 Prompt。

