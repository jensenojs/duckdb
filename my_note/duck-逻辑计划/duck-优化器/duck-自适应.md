---
tags:
  - db/example/duck
  - 道
  - db/plan/optimizer
---

### 1. 自适应优化

#### 核心文件
- `src/optimizer/statistics_propagator.cpp`
  - 统计信息的传播
- `src/optimizer/expression_heuristics.cpp`
  - 表达式优化策略

#### 学习步骤
1. 分析统计信息的收集和更新
2. 理解自适应优化的触发条件
3. 研究优化策略的调整机制

#### 输出要求
- 自适应优化的工作流程
- 优化策略的调整规则
- 实验效果分析