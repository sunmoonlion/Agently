# 设计笔记：Action 与 Execution Environment 为何分离

## 问题

为什么 Python、Shell、MCP 等能力不直接在 ActionExecutor 内启动和持有所有资源？

## 初步答案

Action 描述一次可调用操作；Execution Environment 描述操作依赖的 live resource 生命周期。二者变化频率、复用范围、审批和清理要求不同。

## 权衡

| 设计 | 好处 | 代价 |
|------|------|------|
| 分离 | 资源复用、统一健康检查、policy、scope 和 cleanup | 概念与接线更多 |
| Executor 自管 | 初始实现简单 | 生命周期重复、泄漏、审批分散 |

## 验证任务

- [ ] 对比 LocalFunction 和 MCP Action。
- [ ] 找出 environment requirement 的归一化位置。
- [ ] 跟踪 execution close 时的资源释放。

