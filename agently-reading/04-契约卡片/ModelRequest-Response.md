# 契约卡片：ModelRequest / ModelResponse

## 定位

- `ModelRequest`：组装请求级 Settings、Prompt 和 handlers。
- `ModelResponse`：冻结请求快照，选择 Provider，并暴露结果读取接口。
- `ModelResponseResult`：消费流、解析、校验并缓存结果。

## 文件

- `agently/core/model/ModelRequest.py`
- `agently/core/model/ModelResponse.py`
- `agently/core/model/ModelResponseResult.py`

## 生命周期

```text
ModelRequest
  -> get_response()
  -> ModelResponse(snapshot)
  -> ModelRequester.generate_request_data()
  -> request_model()
  -> ModelResponseResult consumes stream
  -> text / data / meta
```

## 快照边界

`ModelResponse.__init__` 会复制 Settings、Prompt 和 ExtensionHandlers。请求开始后的外部修改不应改变本次响应。

## 输出接口

| 接口 | 输出 |
|------|------|
| `get_text()` | 文本 |
| `get_data()` | 解析后的结构化数据 |
| `get_meta()` | Provider 与运行元数据 |
| `get_generator()` | 同步流 |
| `get_async_generator()` | 异步流 |

## 测试入口

- `tests/test_cores/test_request.py`
- `tests/test_cores/test_response.py`
- `tests/test_cores/test_model_request_validate.py`
- `tests/test_cores/test_model_request_observation.py`

