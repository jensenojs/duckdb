# DuckDB 学习路线

## 0. DuckDB架构总览
### 0.1 核心特性
- 列式存储的嵌入式分析型数据库
- 向量化执行引擎
- 完整的SQL支持（包括复杂子查询）
- 高效的并行处理

### 0.2 关键模块
- 查询处理流程
  ```
  SQL Query -> Parser -> Binder -> Logical Planner -> Optimizer -> Physical Planner -> Execution Engine
  ```
- 重要组件
  - Binder：语义分析和名称解析
  - Logical Planner：生成逻辑查询计划
  - Optimizer：查询优化（规则和成本驱动）
  - Physical Planner：生成物理执行计划
  - Execution Engine：基于Pipeline的向量化执行

### 0.3 创新特点
- Pipeline执行模型
  - 通过Source和Sink算子构建数据流
  - 最小化中间结果物化
  - 高效的并行执行支持
- 高效的子查询处理
  - Mark Join处理相关子查询
  - 子查询重写优化
- 向量化执行优化
  - 批处理执行
  - SIMD加速
- ART索引优化

## 1. 查询执行框架
### 1.1 基础执行流程
- 从SQL到执行的完整路径
  - `ClientContext::Query()` 入口分析
  - Parser到Logical Plan的转换
  - Physical Plan的生成过程

### 1.2 Pipeline 执行模型
- Pipeline的设计理念
  - Source/Sink模型
  - Operator链接方式
- Pipeline的构建与优化
  - `src/execution/physical_plan_generator.cpp`中的实现
  - 并行执行策略

### 1.3 向量化执行实现
- 数据布局设计 (`src/common/vector_operations/`)
- 向量化表达式执行器 (`src/execution/expression_executor/`)
- SIMD优化实现

## 2. 查询优化技术
### 2.1 DuckDB优化器架构
- 优化器总体设计
  - 混合优化策略：规则驱动 + 代价驱动
  - 优化时机选择
  - 优化规则的组织方式

- DuckDB特有的优化创新
  - 自适应Join优化
    - 小表策略
    - 并行度自适应
  - 智能统计信息收集
    - 采样策略
    - 增量更新机制
  - Pipeline感知优化
    - 算子流水线化
    - 中间结果最小化

### 2.2 基础查询优化
- 逻辑优化
  - 谓词处理（下推/上拉）
  - 子查询处理
  - 公共表达式消除
  - 常量折叠
  
- 物理优化
  - 访问路径选择
  - Join策略选择
  - 并行计划生成

### 2.3 高级查询优化
- 子查询处理技术
  - 去相关化策略
  - Mark Join技术
    - 原理与优势
    - 适用场景
  - 子查询重写技术
    - EXISTS/IN优化
    - 聚合子查询处理

- Join顺序优化
  - 启发式策略
  - 代价模型
  - 大规模Join处理

### 2.4 OLAP场景特化优化
- 向量化执行优化
  - 批处理策略
  - SIMD利用
  - 表达式评估优化

- 并行执行优化
  - Pipeline并行
  - 任务调度
  - 负载均衡

- 内存管理优化
  - 列存布局优化
  - 压缩策略
  - 缓存管理

### 2.5 特色优化技术
- 自适应执行
  - 运行时统计收集
  - 计划调整机制

- 混合工作负载优化
  - 小查询快速路径
  - 大查询资源控制

- 扩展优化器
  - 用户定义优化规则
  - 外部数据源优化

## 3. 索引与存储
### 3.1 ART索引实现
- ART树结构 (`src/execution/index/art/`)
- 索引扫描优化
- 多索引应用策略

### 3.2 内存管理
- 内存分配策略
- 缓存管理
- 溢出处理

## 4. 实践学习路径
1. 跟踪简单查询执行
   ```sql
   SELECT * FROM table WHERE id > 10;
   ```
   分析从SQL到结果的完整流程

2. 分析复杂查询优化
   ```sql
   SELECT * FROM t1 
   JOIN t2 ON t1.id = t2.id 
   WHERE EXISTS (SELECT 1 FROM t3 WHERE t3.id = t1.id);
   ```
   研究Join顺序优化和子查询处理

3. 研究特定优化器实现
   - 选择一个优化器文件(如`filter_pushdown.cpp`)
   - 分析其工作原理
   - 编写测试用例验证 