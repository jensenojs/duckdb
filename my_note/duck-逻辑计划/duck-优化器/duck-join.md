---
tags:
  - db/example/duck
  - 道
  - db/plan/optimizer
---
> 如果你对优化器的部分的一些基础感觉到困惑, 这篇[[理论/数据库系统/Bottom Up/1-Plan/Optimizer/数据库查询优化器综述|笔记]]也许会有用



#### 2.2 Join 顺序优化
DuckDB 的 Join 顺序优化器 ( `JoinOrderOptimizer` ) 基于动态规划算法实现，主要包含以下核心组件：

##### 2.2.1 查询图表示
```cpp
// src/optimizer/join_order/query_graph_manager.cpp
class QueryGraphManager {
    bool Build(JoinOrderOptimizer &optimizer, LogicalOperator &op) {
        // 1. 提取Join关系
        relation_manager.ExtractJoinRelations(optimizer, op, filter_operators);
        // 2. 提取边(filters and bindings)
        filters_and_bindings = relation_manager.ExtractEdges(op, filter_operators, set_manager);
        // 3. 创建超图边
        CreateHyperGraphEdges();
    }
};
```
- 使用超图 (Hypergraph) 结构表示查询
- 节点：基础表 (Base Relations)
- 边：Join 条件和过滤谓词
- 支持多表 Join 条件

##### 2.2.2 动态规划求解
```cpp
// src/optimizer/join_order/plan_enumerator.cpp
class PlanEnumerator {
    void SolveJoinOrder() {
        // 1. 初始化叶子节点计划(单表)
        InitLeafPlans();
        
        // 2. 根据查询规模选择求解策略
        if (query_graph_manager.relation_manager.NumRelations() <= EXHAUSTIVE_THRESHOLD) {
            SolveJoinOrderExactly();     // 精确求解
        } else {
            SolveJoinOrderApproximately(); // 启发式求解
        }
    }
};
```

###### 精确求解 (Exact)
- 基于论文"Dynamic Programming Strikes Back"
- 使用动态规划遍历所有可能的 Join 顺序
- 时间复杂度：O (2^n)，适用于小规模查询
- 保证找到最优解

###### 启发式求解 (Approximate)
- 使用贪心策略构建 Join 树
- 每次选择代价最小的 Join
- 时间复杂度：O (n^2)，适用于大规模查询
- 可能得到局部最优解


- `PlanEnumerator`实现了动态规划算法来优化连接顺序
- 基于论文"Dynamic Programming Strikes Back"
- 主要方法:
    1. `InitLeafPlans`: 初始化叶子计划(单表)
    2. `SolveJoinOrder`: 使用动态规划求解最优连接顺序
    3. 支持精确求解(`SolveJoinOrderExactly`)和近似求解(`SolveJoinOrderApproximately`)


3. 连接顺序优化器(JoinOrderOptimizer):

- 主要入口是`Optimize`方法
- 优化流程:
    1. 构建查询图(`query_graph_manager.Build`)
    2. 创建代价模型(`CostModel`)
    3. 初始化计划枚举器(`PlanEnumerator`)
    4. 求解最优连接顺序(`SolveJoinOrder`)
    5. 重构逻辑计划(`Reconstruct`)



- 在[cost_model.cpp](cci:1://file:///Users/jensen/Projects/Databases/duckdb/src/optimizer/join_order/cost_model.cpp)中实现
- 主要方法是`ComputeCost`
- 代价计算:
    - 基于基数估计(`cardinality_estimator.EstimateCardinalityWithSet`)
    - 代价 = 连接基数 + 左子树代价 + 右子树代价



5. 基数估计器(CardinalityEstimator):

- 在[cardinality_estimator.cpp](cci:1://file:///Users/jensen/Projects/Databases/duckdb/src/optimizer/join_order/cardinality_estimator.cpp)中实现
- 主要功能:
    1. 处理过滤条件:
        - `EmptyFilter`: 检查空过滤条件
        - `SingleColumnFilter`: 处理单列过滤
    2. 等价类管理:
        - `AddRelationTdom`: 添加关系到等价域
        - `DetermineMatchingEquivalentSets`: 确定匹配的等价集
    3. 基数估计:
        - `EstimateCardinalityWithSet`: 估计基数
        - 基于分子分母计算:
            - `GetNumerator`: 计算分子
            - `GetDenominator`: 计算分母

6. 总体优化流程:
7. 查询图构建:
    - 提取join关系和过滤条件
    - 构建超图结构表示查询
8. 连接顺序枚举:
    - 使用动态规划算法
    - 支持精确和近似求解
9. 代价估计:
    - 基于基数估计
    - 考虑等价类和过滤条件
10. 计划重构:
    - 根据最优连接顺序重构逻辑计划

这就是DuckDB查询优化器的核心实现。它采用了经典的动态规划方法来求解连接顺序优化问题,同时使用了基于等价类的基数估计方法来评估查询计划的代价。