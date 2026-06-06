# 契约卡片：Prompt

## 定位

Prompt 是分层状态对象，不只是字符串。它把 system、info、instruct、input、output、附件、历史和工具描述转换成 Provider 可消费的表示。

## 文件

- `agently/core/model/Prompt.py`
- `agently/builtins/plugins/PromptGenerator/`

## 主要表示

| 方法 | 输出 |
|------|------|
| `get()` | Prompt 状态字典 |
| `to_prompt_object()` | 标准 Prompt 对象 |
| `to_messages()` | chat messages |
| `to_text()` | 文本 Prompt |
| `to_serializable_prompt_data()` | 观测与持久化数据 |

## 设计问题

1. 为什么 Prompt 继承 `StateData`？
2. 父 Prompt 与 request Prompt 如何合并？
3. output schema 在何处变成 Prompt 文本？
4. attachment 为什么需要独立 rich-content 通道？

## 测试入口

- `tests/test_plugins/test_prompt_generator/`
- `tests/test_cores/test_request.py`

