---
tags:
  - db/plan/optimizer
---


Join 优化是查询优化中最具挑战性的问题之一，它涉及多个方面：
1. Join 顺序的选择
2. Join 策略的选择
3. 特殊 Join 类型的处理

# Join 顺序重排

Join 顺序优化是查询优化中最关键的问题之一，原因在于：
1. **执行代价差异巨大**：不同的 Join 顺序可能导致中间结果大小相差数个数量级
2. **搜索空间庞大**： $n$ 个表的 Join 有 $n!$ 种可能的顺序
3. **影响深远**：Join 顺序影响后续操作的代价和可选择的执行策略

## 例子

示例 1 : 考虑一个简单的电商查询：
```sql
SELECT c.name, o.order_date, p.product_name
FROM customers c, orders o, order_items oi, products p
WHERE c.id = o.customer_id 
  AND o.id = oi.order_id
  AND oi.product_id = p.id
  AND c.country = 'China'
  AND o.order_date >= '2023-01-01'
```

假设数据规模：
```
customers:    1,000,000 行
orders:      10,000,000 行
order_items: 30,000,000 行
products:       100,000 行
```

两种 Join 顺序的对比：
1. **次优顺序** (不考虑过滤条件)：
```sql
-- 执行顺序：从左到右
(((customers ⋈ orders) ⋈ order_items) ⋈ products)

-- 中间结果分析：
Step 1: customers ⋈ orders
  - 1,000,000 × 10,000,000
  - 产生约1,000,000行（假设每客户平均10个订单）
  
Step 2: 中间结果 ⋈ order_items
  - 1,000,000 × 30,000,000
  - 产生约3,000,000行（假设每订单平均3个商品）
  
Step 3: 最终 ⋈ products
  - 3,000,000 × 100,000
  - 最终约3,000,000行
```

2. **优化顺序** (考虑过滤条件)：
```sql
-- 先应用过滤条件
-- c.country = 'China': 假设筛选后剩1%
-- o.order_date >= '2023-01-01': 假设筛选后剩10%

((customers[filtered] ⋈ (orders[filtered] ⋈ order_items)) ⋈ products)

-- 中间结果分析：
Step 1: customers过滤后只剩10,000行
Step 2: orders过滤后只剩1,000,000行
Step 3: orders[filtered] ⋈ order_items
  - 1,000,000 × 3 = 3,000,000行
Step 4: customers[filtered] ⋈ (orders-items结果)
  - 10,000 × 3,000,000 → 300,000行
Step 5: 最终 ⋈ products
  - 300,000 × 100,000 → 300,000行
```

性能差异分析：
- 次优顺序的最大中间结果：3,000,000 行
- 优化顺序的最大中间结果：300,000 行
- 内存使用差异：约 10 倍
- I/O 开销差异：约 10 倍


示例 2：索引利用

考虑带索引的查询场景：
```sql
SELECT o.order_id, c.name, s.status_name
FROM orders o, customers c, order_status s
WHERE o.customer_id = c.id
  AND o.status_id = s.id
  AND o.order_date BETWEEN '2023-01-01' AND '2023-12-31'
  AND c.vip_level > 2
```

索引情况：
```
orders: (order_date) 索引
customers: (id) 索引
order_status: 小表(<100行)
```

不同 Join 顺序的影响：
1. **索引友好顺序**：
```sql
-- 执行顺序
(orders[date_filtered] ⋈ order_status) ⋈ customers[vip_filtered]

优点：
- 利用orders表的日期索引做范围扫描
- 使用customers表的id索引做点查询
- 避免了全表扫描
```

2. **索引不友好顺序**：
```sql
-- 执行顺序
(customers[vip_filtered] ⋈ orders) ⋈ order_status

缺点：
- 无法高效利用orders表的日期索引
- 可能导致大量随机I/O
- 中间结果集较大
```

从上面的例子我们能看到
   - 错误的顺序可能产生巨大的中间结果
   - 大中间结果导致更多的内存消耗
   - 可能需要频繁的磁盘 I/O

所以如果不解决好这个问题, 会带来一系列很严重的影响
## 方法

###### 动态规划方法
对于小规模查询（通常是 10 个表以内），动态规划是最常用的方法：
```python
def find_optimal_join_order(tables):
    # dp[subset]存储最优子计划
    dp = {}
    
    # 初始化单表计划
    for table in tables:
        dp[{table}] = Plan(table)
    
    # 逐步构建更大的计划
    for size in range(2, len(tables) + 1):
        for subset in combinations(tables, size):
            best_cost = float('inf')
            best_plan = None
            
            # 枚举所有可能的分割方式
            for split in find_valid_splits(subset):
                left = dp[split[0]]
                right = dp[split[1]]
                
                # 估算代价
                cost = estimate_join_cost(left, right)
                if cost < best_cost:
                    best_cost = cost
                    best_plan = create_join_plan(left, right)
            
            dp[subset] = best_plan
    
    return dp[frozenset(tables)]
```

- 优点：
	- 保证找到最优解
	- 避免重复计算
	- 容易实现和维护
- 缺点：
	- 时间复杂度 $O(n·2^n)$
	- 空间复杂度 $O(2^n)$
	- 不适用于大规模查询
- 关键优化点
	- **提前剪枝**
	   - 利用统计信息估算下界
	   - 跳过明显次优的方案

### 启发式方法

对于大规模查询，使用启发式方法：

小表驱动大表
   - 高选择性条件优先
   - 考虑数据倾斜
```python
def greedy_join_order(tables):
    result = tables[0]
    remaining = tables[1:]
    
    while remaining:
        best_next = None
        best_cost = float('inf')
        
        # 选择下一个最优的表
        for table in remaining:
            cost = estimate_join_cost(result, table)
            if cost < best_cost:
                best_cost = cost
                best_next = table
        
        result = join(result, best_next)
        remaining.remove(best_next)
    
    return result
```


### 贪心算法 ans so on

1. **贪心算法**
	- 每次选择局部最优的连接顺序
	- 考虑因素：表大小、选择率、索引可用性

```python
def greedy_join_order(tables):
    result = tables[0]
    remaining = tables[1:]
    
    while remaining:
        best_next = None
        best_cost = float('inf')
        
        # 选择下一个最优的表
        for table in remaining:
            cost = estimate_join_cost(result, table)
            if cost < best_cost:
                best_cost = cost
                best_next = table
        
        result = join(result, best_next)
        remaining.remove(best_next)
    
    return result
```


**遗传算法**
- 将 Join 顺序编码为染色体
- 通过交叉和变异探索搜索空间
- 适合并行优化

### 机器学习

[SkinnerDB: Regret-Bounded Query Evaluation via Reinforcement Learning, 2019](zotero://select/library/items/LAJH9WHJ)

# Join 策略选择
不同的 Join 策略适用于不同场景：

1. **Nested Loop Join**
```sql
-- 适用场景
-- 1. 内表很小
-- 2. 外表有索引
-- 3. 需要第一条结果快速返回
for each row r in outer_table:
    for each row s in inner_table:
        if join_condition(r,s):
            emit(r,s)
```

2. **Hash Join**
```sql
-- 适用场景
-- 1. 两表大小差异大
-- 2. Join key没有排序
-- 3. 内存足够存放小表
build_phase:
    H = empty hash table
    for each row r in small_table:
        insert r into H
        
probe_phase:
    for each row s in large_table:
        find matching rows in H
        emit matches
```

3. **Merge Join**
```sql
-- 适用场景
-- 1. 输入已经排序
-- 2. 输出需要排序
-- 3. 内存受限
while both_inputs_have_rows:
    if r.key == s.key:
        emit matches
    elif r.key < s.key:
        advance r
    else:
        advance s
```

策略选择的考虑因素：
1. **数据特征**
   - 表大小
   - 数据分布
   - 是否排序
   
2. **系统状态**
   - 可用内存
   - 并行度
   - I/O 带宽

3. **查询需求**
   - 结果顺序
   - 响应时间
   - 吞吐量

# 特殊 Join 处理
1. **外连接转换**
```sql
-- 左外连接转内连接
SELECT * FROM R LEFT JOIN S ON R.id = S.id
WHERE S.val IS NOT NULL
-- 可以转换为
SELECT * FROM R INNER JOIN S ON R.id = S.id

-- 但要注意，不是所有外连接都能转换
SELECT * FROM R LEFT JOIN S ON R.id = S.id
WHERE S.val > 10 OR S.val IS NULL
-- 这个就不能转换为内连接
```

2. **半连接优化**
```sql
-- 原始查询
SELECT * FROM R 
WHERE EXISTS (SELECT 1 FROM S WHERE S.id = R.id)

-- 可以优化为
SELECT DISTINCT R.* 
FROM R SEMI JOIN S ON R.id = S.id
```

3. **多表 Join 消除**
```sql
-- 原始查询
SELECT R.* FROM R 
JOIN S ON R.id = S.id 
JOIN T ON S.id = T.id

-- 如果S.id是外键，指向R.id
-- 且T.id是外键，指向S.id
-- 可以消除不必要的Join
SELECT R.* FROM R
```


