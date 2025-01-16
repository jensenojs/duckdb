---
tags:
  - db/example/duck
  - db/plan/optimizer
---
> 如果你对优化器的部分的一些基础感觉到困惑, 这篇[[理论/数据库系统/Bottom Up/1-Plan/Optimizer/数据库查询优化器综述|笔记]]也许会有用




DuckDB 实现了两种谓词下推机制：静态谓词下推和运行时过滤。

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

1. Join 谓词下推
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

##### 2.1.2 运行时过滤 (Runtime Filter)

DuckDB 在 Hash Join 执行过程中实现了运行时过滤机制：

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
   - 在 Hash 表构建完成后，收集构建端的 Join 键值
   - 创建基于这些值的 IN 过滤条件

2. 探测阶段：
   - 在扫描 probe 端数据前应用过滤条件
   - 过滤掉一定不会匹配的记录

3. 优化效果：
   - 减少 probe 端需要处理的数据量
   - 降低 Hash 表探测成本
   - 特别适用于选择率较低的 Join

##### 2.1.3 示例：复杂查询的谓词下推

考虑以下 SQL 查询：
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
   - 在执行 Join 时，从 customers 表收集所有满足 `country='USA'` 的 `customer_id`
   - 创建 `IN (customer_ids)` 过滤条件
   - 在扫描 orders 表时应用此过滤条件

优化效果：
1. 静态下推：提前过滤无关数据
2. 运行时过滤：减少 Join 的探测代价
3. 整体性能提升：
   - 减少 I/O 和内存使用
   - 降低计算成本
   - 提高 Join 效率