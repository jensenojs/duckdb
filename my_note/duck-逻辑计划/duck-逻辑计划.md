---
tags:
  - db/example/duck
  - db/plan
---
在之前[[实践/examples/Database/duckdb/duckdb-前端|之前的文章]]我们已经梳理了 `Parser` 和 `Binder` 的流程, 这里再给一个浓缩的调用链路回顾一下

![[实践/examples/Database/duckdb/duckdb-前端#^70af6f|duckdb-前端]]

对于逻辑计划的学习, 不能像之前那样把整体的粗粗地过一遍就完了, 因为有很多值得学习的机制不是一条 SQL 的流程就能走完的. 因此本篇笔记只负责基础流程的走读, [[实践/examples/Database/duckdb/duck-逻辑计划/duck-优化器/duck-优化器|优化器]] 的走读由其他的笔记完成. 对于每个优化方案的具体学习会拆分到不同的笔记中, 链接在这里, 方便后续与其他的数据库实现进行对比

# Logical Plan 生成过程



- `Planner` 类：负责生成逻辑查询计划
- `LogicalOperator` ：逻辑算子的基类
- `BoundQueryNode` 到 `LogicalOperator` 的转换过程


```cpp
// 主要入口
Planner::CreatePlan(BoundQueryNode &node)
-> CreatePlan(BoundSelectNode &statement) // SELECT查询
-> CreatePlan(BoundSetOperationNode &node) // UNION/INTERSECT等
-> CreatePlan(BoundRecursiveCTENode &node) // WITH RECURSIVE
```


```c++ title:""

```

```c++ title:""

```

```c++ title:""

```

```c++ title:""

```

```c++ title:""

```

```c++ title:""

```

```c++ title:""

```

```c++ title:""

```

```c++ title:""

```

```c++ title:""

```

```c++ title:""

```

```c++ title:""

```
