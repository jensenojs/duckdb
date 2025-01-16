---
tags:
  - db/example/duck
  - db/plan/optimizer
---
> 如果你对优化器的部分的一些基础感觉到困惑, 这篇[[理论/数据库系统/Bottom Up/1-Plan/Optimizer/数据库查询优化器综述|笔记]]也许会有用


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