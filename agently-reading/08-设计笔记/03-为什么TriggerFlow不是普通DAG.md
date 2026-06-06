# 设计笔记：为什么 TriggerFlow 不是普通 DAG

## 问题

为什么 TriggerFlow 以 signal、execution lifecycle 和 interrupt 为中心，而不只保存节点与边？

## 初步答案

AI 工作流的触发源不只来自上一节点返回值，还可能来自模型的部分结构化输出、Action 结果、人工输入、外部事件和子流。signal 比固定边更适合表达这些来源。

## 待验证证据

- `when(...)`
- runtime stream
- pause / resume
- save / load execution
- sub-flow bindings

## 设计代价

灵活事件模型会增加关闭条件、并发 drain、幂等和持久化的一致性难度。

