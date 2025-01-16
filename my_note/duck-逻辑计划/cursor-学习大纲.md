# DuckDB优化器学习大纲

## 学习目标

通过代码走读的方式，验证查询优化器的理论知识在DuckDB中的具体实现。

## 第一阶段：基础流程走读

使用基准SQL进行代码追踪：
```sql
SELECT a, COUNT(*) 
FROM table 
WHERE a > 10 
GROUP BY a;
```

### 1. Logical Plan生成过程

#### 核心文件
- `src/planner/planner.cpp`
  - `LogicalPlanGenerator` 类的实现
  - `CreatePlan` 方法的处理流程
- `src/include/duckdb/planner/logical_operator.hpp`
  - 理解逻辑算子的基类设计
- `src/planner/operator/` 目录
  - 各种逻辑算子的具体实现

#### 学习步骤
1. 分析 `LogicalPlanGenerator` 如何将Bound Tree转换为逻辑计划
2. 追踪上述SQL中涉及的主要逻辑算子：
   - `LogicalGet`
   - `LogicalFilter`
   - `LogicalAggregate`
3. 记录完整的转换过程和关键代码片段

#### 输出要求
- 绘制Bound Tree到Logical Plan的转换流程图
- 记录关键方法的调用链
- 总结DuckDB的逻辑算子体系

### 2. 优化器框架分析

#### 核心文件
- `src/optimizer/optimizer.cpp`
  - 优化器的主体流程
  - 规则注册和应用机制
- `src/include/duckdb/optimizer/optimizer.hpp`
  - 优化器类的定义
- `src/include/duckdb/optimizer/rule.hpp`
  - 优化规则的基类设计

#### 学习步骤
1. 分析优化器的初始化过程
2. 理解规则的注册机制
3. 追踪优化规则的应用顺序
4. 分析优化器的整体架构

#### 输出要求
- 优化器框架类图
- 优化规则的调用链路图
- 优化器工作流程的时序图

## 第二阶段：具体优化规则验证

### 1. 谓词下推优化

#### 核心文件
- `src/optimizer/filter_pushdown.cpp`
  - 基本的谓词下推实现
- `src/optimizer/join_filter_pushdown_optimizer.cpp`
  - Join条件的下推处理
- `src/optimizer/pushdown/` 目录
  - 各种特殊场景的下推实现

#### 验证SQL
```sql
-- Case 1: 简单谓词下推
SELECT * FROM t1 JOIN t2 ON t1.id = t2.id WHERE t1.a > 10;

-- Case 2: 复杂条件下推
SELECT * FROM t1 JOIN t2 ON t1.id = t2.id 
WHERE t1.a > 10 AND t2.b < 20 OR t1.c = t2.c;

-- Case 3: 子查询场景
SELECT * FROM t1 WHERE a > (SELECT avg(b) FROM t2);
```

#### 学习步骤
1. 分析谓词下推的判断条件
2. 理解不同类型谓词的下推策略
3. 研究Join场景下的特殊处理
4. 验证子查询相关的下推优化

#### 输出要求
- 谓词下推前后的计划对比
- 下推条件的判断逻辑总结
- 特殊场景的处理策略

### 2. 列裁剪优化

#### 核心文件
- `src/optimizer/remove_unused_columns.cpp`
  - 未使用列的识别和移除
- `src/optimizer/column_lifetime_analyzer.cpp`
  - 列生命周期分析
- `src/include/duckdb/optimizer/column_lifetime_analyzer.hpp`
  - 相关数据结构定义

#### 验证SQL
```sql
-- Case 1: 简单列裁剪
SELECT a FROM (SELECT a,b,c FROM t1) sub;

-- Case 2: Join场景
SELECT t1.a FROM t1 JOIN t2 ON t1.id = t2.id;

-- Case 3: 聚合场景
SELECT sum(a) FROM (SELECT a,b,c FROM t1) sub GROUP BY b;
```

#### 学习步骤
1. 分析列引用的追踪机制
2. 理解列生命周期的判断逻辑
3. 研究不同场景下的裁剪策略
4. 验证优化效果

#### 输出要求
- 列裁剪的判断流程图
- 优化前后的内存占用对比
- 特殊场景的处理总结

### 3. 子查询处理

#### 核心文件
- `src/optimizer/in_clause_rewriter.cpp`
  - IN子句的重写优化
- `src/optimizer/unnest_rewriter.cpp`
  - 子查询展开处理
- `src/planner/subquery/` 目录
  - 子查询相关的计划处理

#### 验证SQL
```sql
-- Case 1: IN子句
SELECT * FROM t1 WHERE a IN (SELECT b FROM t2);

-- Case 2: EXISTS子句
SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.b = t1.a);

-- Case 3: 标量子查询
SELECT *, (SELECT max(b) FROM t2 WHERE t2.c = t1.c) FROM t1;
```

#### 学习步骤
1. 分析子查询的分类和识别
2. 理解重写规则的应用条件
3. 研究相关性的判断逻辑
4. 验证重写后的执行效率

#### 输出要求
- 子查询重写的决策树
- 重写前后的计划对比
- 相关性判断的逻辑总结

### 4. Join优化

#### 核心文件
- `src/optimizer/join_order/` 目录
  - Join顺序枚举
  - 代价估算
- `src/optimizer/statistics/` 目录
  - 统计信息处理
- `src/optimizer/filter_combiner.cpp`
  - 复杂过滤条件的处理

#### 验证SQL
```sql
-- Case 1: 多表Join
SELECT * FROM t1,t2,t3 
WHERE t1.id=t2.id AND t2.id=t3.id;

-- Case 2: 不同Join类型
SELECT * FROM t1 
LEFT JOIN t2 ON t1.id=t2.id 
INNER JOIN t3 ON t2.id=t3.id;

-- Case 3: 复杂条件
SELECT * FROM t1,t2,t3 
WHERE t1.id=t2.id AND t2.id=t3.id 
  AND t1.a>10 AND t2.b<20;
```

#### 学习步骤
1. 分析Join顺序枚举算法
2. 理解代价估算模型
3. 研究统计信息的使用
4. 验证不同场景的优化效果

#### 输出要求
- Join顺序枚举的算法描述
- 代价模型的计算公式
- 优化效果的实验数据

## 第三阶段：高级特性研究

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

### 2. 物理优化

#### 核心文件
- `src/optimizer/compressed_materialization.cpp`
  - 压缩存储优化
- `src/optimizer/late_materialization.cpp`
  - 延迟物化策略

#### 学习步骤
1. 分析物化策略的选择
2. 理解压缩算法的应用
3. 研究优化效果的评估

#### 输出要求
- 物化策略的决策逻辑
- 压缩方案的选择规则
- 性能对比数据

## 调试方法

### 1. 计划分析
```sql
-- 查看逻辑计划
EXPLAIN SELECT ...

-- 查看物理计划
EXPLAIN ANALYZE SELECT ...
```

### 2. 日志跟踪
- 在关键优化规则处添加日志
- 跟踪中间计划的变化
- 记录优化决策的依据

### 3. 性能分析
- 使用profile工具分析执行时间
- 对比优化前后的资源使用
- 验证优化规则的效果

## 验证方法

### 1. 测试用例设计
- 每个优化规则准备多个测试SQL
- 覆盖常见场景和边界情况
- 包含正向和反向测试

### 2. 效果度量
- 执行时间对比
- 资源使用分析
- 中间结果大小比较

### 3. 文档记录
- 优化规则的触发条件
- 具体的优化过程
- 实验数据和结论

## 预期输出

1. 每个优化规则的详细分析文档
2. 关键代码的实现思路和注释
3. 完整的测试用例集合
4. 性能优化效果的数据报告

## 时间规划

- 第一阶段：1-2周
- 第二阶段：3-4周
- 第三阶段：2-3周

建议每个阶段完成后进行总结和复盘，确保对每个优化规则都有深入理解。 