---
tags:
  - db/example/duck
  - db/plan/optimizer
---
> 如果你对优化器的部分的一些基础感觉到困惑, 这篇[[理论/数据库系统/Bottom Up/1-Plan/Optimizer/数据库查询优化器综述|笔记]]也许会有用


# 优化器框架分析


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


# 具体优化规则验证




1. 查询图(Query Graph)的构建:

- `QueryGraphManager`负责构建和管理查询图
- 主要步骤:
    1. 通过`ExtractJoinRelations`提取join关系
    2. 使用`ExtractEdges`提取边(filters and bindings)
    3. `CreateHyperGraphEdges` 创建超图边, 主要处理比较表达式作为join谓词




# 各类优化规则实现

- `src/optimizer/` ：各类优化规则实现

1. 基础转换优化
   - 谓词下推
   - 列裁剪
   - 常量折叠


2. 子查询处理
   - 子查询去相关
   - 子查询展平
   - Mark Join 转换

3. Join 优化
   - Join 顺序优化
   - Join 类型选择
   - 小表优化

4. 聚合和窗口函数优化
   - 分组优化
   - 窗口函数重写
   - Having 子句处理