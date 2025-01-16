# DuckDB逻辑计划与优化器学习大纲

## 第一阶段：逻辑计划生成与基础优化

### 1. DuckDB的逻辑算子体系

#### 1.1 基础数据源算子
- `LogicalGet`：表扫描算子
  ```cpp
  // src/planner/operator/logical_get.cpp
  class LogicalGet : public LogicalOperator {
      TableFunction function;           // 数据源函数
      unique_ptr<FunctionData> bind_data; // 绑定数据
      vector<string> names;            // 列名
      vector<LogicalType> returned_types; // 返回类型
      TableFilterSet table_filters;    // 下推的过滤条件
  };
  ```
  - 支持多种数据源：基表、索引、外部表等
  - 集成了基础的过滤下推机制
  - 提供统计信息接口用于优化

- `LogicalDummyScan`：空数据源
  - 用于常量查询
  - 用于某些优化重写场景

#### 1.2 过滤和投影算子
- `LogicalFilter`：过滤算子
  ```cpp
  // src/planner/operator/logical_filter.cpp
  class LogicalFilter : public LogicalOperator {
      // 核心功能：谓词分解
      static bool SplitPredicates(vector<unique_ptr<Expression>> &expressions) {
          // 将AND连接的谓词拆分为独立谓词
          // 便于后续的谓词下推优化
      }
  };
  ```
  - 支持复杂谓词表达式
  - 内置谓词分解机制
  - 为优化提供重组机会

- `LogicalProjection`：投影算子
  ```cpp
  // src/planner/operator/logical_projection.cpp
  class LogicalProjection : public LogicalOperator {
      vector<unique_ptr<Expression>> expressions;
      vector<string> column_names;
  };
  ```
  - 处理列裁剪
  - 支持表达式计算
  - 维护列别名

#### 1.3 Join相关算子
- `LogicalJoin`：基类
  ```cpp
  // src/planner/operator/logical_join.cpp
  class LogicalJoin : public LogicalOperator {
      JoinType type;            // INNER, LEFT, RIGHT, FULL
      vector<JoinCondition> conditions;
      vector<unique_ptr<Expression>> expressions;
  };
  ```

- 具体实现：
  - `LogicalComparisonJoin`：基于比较的连接
  - `LogicalAnyJoin`：ANY/SOME语义
  - `LogicalCrossProduct`：笛卡尔积
  - `LogicalDelimJoin`：用于子查询处理

#### 1.4 聚合和窗口算子
- `LogicalAggregate`：聚合算子
  ```cpp
  // src/planner/operator/logical_aggregate.cpp
  class LogicalAggregate : public LogicalOperator {
      vector<unique_ptr<Expression>> groups;   // GROUP BY表达式
      vector<unique_ptr<Expression>> expressions; // 聚合表达式
      vector<unique_ptr<AggregateFunction>> functions; // 聚合函数
  };
  ```
  - 支持复杂分组表达式
  - 处理DISTINCT聚合
  - 优化机会：部分聚合

- `LogicalWindow`：窗口函数
  ```cpp
  class LogicalWindow : public LogicalOperator {
      vector<unique_ptr<Expression>> partitions; // PARTITION BY
      vector<unique_ptr<Expression>> orders;     // ORDER BY
      vector<unique_ptr<WindowFunction>> functions;
  };
  ```

### 2. 从SQL到逻辑计划的转换过程

#### 2.1 Planner的核心流程
```cpp
// src/planner/planner.cpp
class Planner {
    unique_ptr<LogicalOperator> CreatePlan(SQLStatement &statement) {
        // 1. 绑定语句
        auto bound_statement = Bind(statement);
        
        // 2. 生成初始逻辑计划
        LogicalPlanGenerator generator(*this);
        auto plan = generator.CreatePlan(std::move(bound_statement));
        
        // 3. 进行基础优化
        Optimizer optimizer(context);
        return optimizer.Optimize(std::move(plan));
    }
};
```

#### 2.2 逻辑计划生成器
```cpp
// src/planner/logical_plan_generator.cpp
class LogicalPlanGenerator {
    unique_ptr<LogicalOperator> CreatePlan(BoundSQLStatement &statement) {
        switch (statement.type) {
            case StatementType::SELECT:
                return CreateSelect(statement.Cast<BoundSelectStatement>());
            case StatementType::INSERT:
                return CreateInsert(statement.Cast<BoundInsertStatement>());
            // ...其他语句类型
        }
    }
};
```

#### 2.3 完整示例分析
以下面的SQL为例：
```sql
SELECT a, COUNT(*) 
FROM t1 
WHERE a > 10 
GROUP BY a;
```

生成过程：
1. `LogicalGet`生成
   ```cpp
   auto get = make_uniq<LogicalGet>();
   get->table_index = t1_index;
   get->column_ids = {a_column_id};
   ```

2. `LogicalFilter`生成
   ```cpp
   auto filter = make_uniq<LogicalFilter>();
   filter->expressions.push_back(make_uniq<BoundComparisonExpression>(
       ExpressionType::COMPARE_GREATER_THAN,
       make_uniq<BoundColumnRefExpression>("a", LogicalType::INTEGER, 0),
       make_uniq<BoundConstantExpression>(Value::INTEGER(10))
   ));
   ```

3. `LogicalAggregate`生成
   ```cpp
   auto aggregate = make_uniq<LogicalAggregate>();
   // 添加分组键
   aggregate->groups.push_back(make_uniq<BoundColumnRefExpression>(
       "a", LogicalType::INTEGER, 0
   ));
   // 添加COUNT(*)
   aggregate->expressions.push_back(make_uniq<BoundAggregateExpression>(
       AggregateFunction::COUNT_STAR,
       vector<unique_ptr<Expression>>(),
       false
   ));
   ```

### 3. 类型系统与表达式

#### 3.1 类型系统
```cpp
// src/common/types.hpp
class LogicalType {
    PhysicalType physical_type;
    LogicalTypeId id;
    shared_ptr<ExtraTypeInfo> type_info;
};
```

#### 3.2 表达式体系
- `BoundColumnRefExpression`：列引用
- `BoundConstantExpression`：常量
- `BoundComparisonExpression`：比较
- `BoundConjunctionExpression`：AND/OR
- `BoundFunctionExpression`：函数调用
- `BoundAggregateExpression`：聚合

## 第二阶段：优化器专题研究

### 1. 优化器框架

#### 1.1 整体架构
```cpp
// src/optimizer/optimizer.cpp
class Optimizer {
    vector<unique_ptr<OptimizerRule>> rules;
    
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan) {
        // 1. 应用规则优化
        for (auto &rule : rules) {
            plan = rule->Apply(std::move(plan));
        }
        
        // 2. 代价优化
        return plan;
    }
};
```

#### 1.2 优化规则体系
```cpp
// src/optimizer/rule/
class OptimizerRule {
    virtual unique_ptr<LogicalOperator> Apply(unique_ptr<LogicalOperator> root);
};

// 具体规则示例
class FilterPushdown : public OptimizerRule {
    unique_ptr<LogicalOperator> Apply(unique_ptr<LogicalOperator> root) override;
};
```

### 2. 核心优化技术

#### 2.1 谓词下推优化

DuckDB实现了两种谓词下推机制：静态谓词下推和运行时过滤。

##### 2.1.1 静态谓词下推
```cpp
// src/optimizer/filter_pushdown.cpp
class FilterPushdown {
    // 核心数据结构
    struct Filter {
        unordered_set<idx_t> bindings;     // 过滤条件涉及的列绑定
        unique_ptr<Expression> filter;      // 过滤表达式
    };
    
    vector<unique_ptr<Filter>> filters;     // 待下推的过滤条件
    FilterCombiner combiner;                // 过滤条件合并器
    
    // 主入口函数
    unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> op) {
        switch (op->type) {
        case LOGICAL_FILTER:      return PushdownFilter(std::move(op));
        case LOGICAL_PROJECTION:  return PushdownProjection(std::move(op));
        case LOGICAL_JOIN:        return PushdownJoin(std::move(op));
        case LOGICAL_AGGREGATE:   return PushdownAggregate(std::move(op));
        // ...其他算子
        }
    }
};
```

主要优化规则：

1. Join谓词下推
```cpp
unique_ptr<LogicalOperator> PushdownJoin(unique_ptr<LogicalOperator> op) {
    auto &join = op->Cast<LogicalJoin>();
    
    // 1. 获取左右表的列绑定
    unordered_set<idx_t> left_bindings, right_bindings;
    LogicalJoin::GetTableReferences(*join.children[0], left_bindings);
    LogicalJoin::GetTableReferences(*join.children[1], right_bindings);
    
    // 2. 根据Join类型选择下推策略
    switch (join.join_type) {
    case JoinType::INNER:
        return PushdownInnerJoin(std::move(op), left_bindings, right_bindings);
    case JoinType::LEFT:
        return PushdownLeftJoin(std::move(op), left_bindings, right_bindings);
    // ...其他Join类型
    }
}
```

2. 聚合算子下推
```cpp
unique_ptr<LogicalOperator> PushdownAggregate(unique_ptr<LogicalOperator> op) {
    auto &aggr = op->Cast<LogicalAggregate>();
    
    // 区分可下推和不可下推的条件
    vector<unique_ptr<Expression>> pushdown_filters;
    vector<unique_ptr<Expression>> aggr_filters;
    
    for (auto &filter : filters) {
        if (CanPushdownThroughAggregate(*filter, aggr)) {
            pushdown_filters.push_back(std::move(filter->filter));
        } else {
            aggr_filters.push_back(std::move(filter->filter));
        }
    }
    
    // 递归下推到子节点
    op->children[0] = AddLogicalFilter(std::move(op->children[0]), 
                                     std::move(pushdown_filters));
    
    return AddLogicalFilter(std::move(op), std::move(aggr_filters));
}
```

##### 2.1.2 运行时过滤(Runtime Filter)

DuckDB在Hash Join执行过程中实现了运行时过滤机制：

```cpp
// src/execution/operator/join/physical_hash_join.cpp
class JoinFilterPushdownInfo {
public:
    void PushInFilter(const JoinFilterPushdownFilter &info, JoinHashTable &ht,
                     const PhysicalOperator &op, idx_t filter_idx, idx_t filter_col_idx) {
        // 1. 扫描Hash表构建段获取所有值
        Vector build_vector(ht.layout.GetTypes()[build_idx], key_count);
        data_collection.Gather(tuples_addresses, build_vector);
        
        // 2. 生成IN过滤条件
        value_set_t unique_ht_values;
        for (idx_t k = 0; k < key_count; k++) {
            unique_ht_values.insert(build_vector.GetValue(k));
        }
        
        // 3. 创建运行时过滤条件
        auto in_filter = make_uniq<InFilter>(unique_ht_values, true);
        
        // 4. 应用到probe端
        info.table_filters.push_back(std::move(in_filter));
    }
};
```

运行时过滤的工作流程：

1. 构建阶段：
   - 在Hash表构建完成后，收集构建端的Join键值
   - 创建基于这些值的IN过滤条件

2. 探测阶段：
   - 在扫描probe端数据前应用过滤条件
   - 过滤掉一定不会匹配的记录

3. 优化效果：
   - 减少probe端需要处理的数据量
   - 降低Hash表探测成本
   - 特别适用于选择率较低的Join

##### 2.1.3 示例：复杂查询的谓词下推

考虑以下SQL查询：
```sql
SELECT c.name, SUM(o.amount) as total
FROM customers c 
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'USA' 
  AND o.amount > 100 
  AND EXTRACT(YEAR FROM o.order_date) = 2024
GROUP BY c.name
HAVING SUM(o.amount) > 1000;
```

优化过程：

1. 静态谓词下推：
   ```
   LogicalFilter(HAVING SUM(o.amount) > 1000)
    └─ LogicalAggregate(GROUP BY: c.name)
        └─ LogicalJoin(c.id = o.customer_id)
            ├─ LogicalFilter(c.country = 'USA')
            │   └─ LogicalGet(customers)
            └─ LogicalFilter(o.amount > 100 AND EXTRACT(YEAR FROM o.order_date) = 2024)
                └─ LogicalGet(orders)
   ```

2. 运行时过滤：
   - 在执行Join时，从customers表收集所有满足`country='USA'`的`customer_id`
   - 创建`IN (customer_ids)`过滤条件
   - 在扫描orders表时应用此过滤条件

优化效果：
1. 静态下推：提前过滤无关数据
2. 运行时过滤：减少Join的探测代价
3. 整体性能提升：
   - 减少I/O和内存使用
   - 降低计算成本
   - 提高Join效率


#### 2.2 Join顺序优化
DuckDB的Join顺序优化器(`JoinOrderOptimizer`)基于动态规划算法实现，主要包含以下核心组件：

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
- 使用超图(Hypergraph)结构表示查询
- 节点：基础表(Base Relations)
- 边：Join条件和过滤谓词
- 支持多表Join条件

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

###### 精确求解(Exact)
- 基于论文"Dynamic Programming Strikes Back"
- 使用动态规划遍历所有可能的Join顺序
- 时间复杂度：O(2^n)，适用于小规模查询
- 保证找到最优解

###### 启发式求解(Approximate)
- 使用贪心策略构建Join树
- 每次选择代价最小的Join
- 时间复杂度：O(n^2)，适用于大规模查询
- 可能得到局部最优解

##### 2.2.3 代价模型
```cpp
// src/optimizer/join_order/cost_model.cpp
class CostModel {
    double ComputeCost(DPJoinNode &left, DPJoinNode &right) {
        // 1. 获取Join后的关系集合
        auto &combination = query_graph_manager.set_manager.Union(left.set, right.set);
        // 2. 估算Join基数
        auto join_card = cardinality_estimator.EstimateCardinalityWithSet<double>(combination);
        // 3. 计算总代价 = Join代价 + 左子树代价 + 右子树代价
        return join_card + left.cost + right.cost;
    }
};
```

##### 2.2.4 基数估计
```cpp
// src/optimizer/join_order/cardinality_estimator.cpp
class CardinalityEstimator {
    double EstimateCardinalityWithSet(JoinRelationSet &new_set) {
        // 1. 计算分子(基数上界)
        auto numerator = GetNumerator(new_set);
        
        // 2. 计算分母(选择率)
        auto denom = GetDenominator(new_set);
        
        // 3. 基于等价类和过滤条件调整基数
        return numerator / denom.multiplier;
    }
    
    // 维护等价类信息
    void AddToEquivalenceSets(FilterInfo *filter_info) {
        // 处理等值Join条件
        // 构建和维护等价类
        // 用于提高基数估计准确性
    }
};
```

##### 2.2.5 计划重构
```cpp
// src/optimizer/join_order/query_graph_manager.cpp
class QueryGraphManager {
    unique_ptr<LogicalOperator> Reconstruct(unique_ptr<LogicalOperator> plan) {
        // 1. 根据最优Join顺序重构查询计划
        auto join_plan = GenerateJoins(extracted_relations, *final_set);
        
        // 2. 处理剩余的过滤条件
        for (auto &filter : remaining_filters) {
            join_plan = PushFilter(std::move(join_plan), std::move(filter));
        }
        
        return join_plan;
    }
};
```

##### 2.2.6 示例：三表Join优化

考虑查询：
```sql
SELECT * FROM orders o 
JOIN customers c ON o.customer_id = c.id
JOIN lineitem l ON o.order_id = l.order_id
WHERE o.total > 1000;
```

优化过程：
1. 构建查询图：
   - 节点：{orders, customers, lineitem}
   - 边：{(o,c), (o,l), filter(o.total)}

2. 枚举Join顺序：
   ```
   可能的顺序：
   - (orders ⋈ customers) ⋈ lineitem
   - (orders ⋈ lineitem) ⋈ customers
   - (customers ⋈ orders) ⋈ lineitem
   ```

3. 代价计算：
   ```cpp
   // 假设基数
   |orders| = 1000
   |customers| = 100
   |lineitem| = 5000
   
   // 计算每个顺序的代价
   cost1 = |orders ⋈ customers| + |orders ⋈ customers ⋈ lineitem|
   cost2 = |orders ⋈ lineitem| + |orders ⋈ lineitem ⋈ customers|
   cost3 = |customers ⋈ orders| + |customers ⋈ orders ⋈ lineitem|
   ```

4. 选择最优顺序：
   ```cpp
   // 假设cost1最小，生成最终计划
   LogicalJoin {
       left: LogicalJoin {
           left: LogicalGet(orders),
           right: LogicalGet(customers),
           condition: o.customer_id = c.id
       },
       right: LogicalGet(lineitem),
       condition: o.order_id = l.order_id
   }
   ```

#### 2.3 子查询处理
```cpp
// src/optimizer/in_clause_rewriter.cpp
class InClauseRewriter : public OptimizerRule {
    unique_ptr<LogicalOperator> RewriteInClause(unique_ptr<LogicalOperator> op) {
        // 1. 检测IN子句
        // 2. 分析相关性
        // 3. 选择重写策略：
        //    - 转换为semi join
        //    - 转换为EXISTS
        //    - 展开为OR
    }
};
```

### 3. 统计信息与代价估算

#### 3.1 基础统计信息
```cpp
// src/statistics/base_statistics.hpp
class BaseStatistics {
    Value min;              // 最小值
    Value max;             // 最大值
    idx_t distinct_count;  // 不同值数量
    bool has_null;        // 是否包含NULL
    bool is_valid;        // 统计信息是否有效
};
```

#### 3.2 直方图
```cpp
// src/statistics/histogram.hpp
class Histogram {
    vector<Value> bins;     // 直方图桶
    vector<double> counts;  // 每个桶的计数
    
    double EstimateSelectivity(const Value &value) {
        // 估算选择度
    }
};
```

#### 3.3 代价模型
```cpp
// src/optimizer/cost_model.hpp
class CostModel {
    double EstimateScan(const LogicalGet &get) {
        return get.EstimateCardinality() * SCAN_COST_PER_ROW;
    }
    
    double EstimateJoin(const LogicalJoin &join) {
        auto left_card = join.children[0]->EstimateCardinality();
        auto right_card = join.children[1]->EstimateCardinality();
        return left_card * right_card * JOIN_COST_PER_PAIR;
    }
};
```

## 输出要求

### 1. 第一阶段输出
- 逻辑算子继承体系的完整类图
- 示例SQL的详细转换过程分析
- 关键数据结构的内存布局分析

### 2. 第二阶段输出
- 每个优化规则的触发条件和转换过程
- 优化效果的性能测试报告
- 特殊场景的处理策略总结
