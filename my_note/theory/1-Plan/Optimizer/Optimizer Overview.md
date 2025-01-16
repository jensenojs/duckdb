---
tags:
  - db/plan/optimizer
---
https://zhuanlan.zhihu.com/p/464717139

## Query Optimizer
查询优化器的工作是将内部查询表示转换成执行查询的高效查询计划。
### 它能做什么
特别地，优化器的查询计划可以指定
1. 查询将访问的磁盘设备，以及每个设备的随机和顺序 I/O 次数的估计
2. 根据查询计划中的操作符和需要处理的元组数量，估计查询的 CPU 负载
3. 对查询数据结构的内存占用进行估计，包括在连接和其他查询执行任务期间对大型输入进行排序和哈希所需的空间

查询计划可以被看作是一个将表数据通过查询操作符图的数据流图。在许多系统中，查询首先被分解成 SELECT-FROM-WHERE 查询块。

然后使用 Selinger 等人在著名论文中描述的技术对每个单独的查询块进行优化。

> P. G. Selinger, M. Astrahan, D. Chamberlin, R. Lorie, and T. Price, “Access path selection in a relational database management system,” in Proceedings of ACM-SIGMOD International Conference on Management of Data, pp. 22–34, Boston, June 1979.

完成后，通常会在每个查询块的顶部添加一些操作符作为后处理，以计算如果存在的话的 GROUP BY、ORDER BY、HAVING 和 DISTINCT 子句。然后将各个块以直接的方式拼接在一起。

为了实现跨平台可移植性，现在每个主要的数据库管理系统（DBMS）都将查询编译成某种可解释的数据结构。它们之间唯一的区别是中间形式的抽象级别。

虽然 Selinger 的论文被广泛认为是查询优化的“圣经”，但它是初步研究。所有系统都在多个维度上显著扩展了这项工作。主要的扩展包括：

### 一些扩展

#### Plan space
The System R optimizer constrained its plan space somewhat by focusing only on “left-deep” query plans (where the right-hand input to a join must be a base table),  
And by “postponing Cartesian products” (ensuring that Cartesian products appear only after all joins in a dataflow).

In commercial systems today, it is well known that “bushy” trees (with nested right-hand inputs) and early use of Cartesian products can be useful in some cases. Hence both options are considered under some circumstances by most systems.

System R 优化器通过只关注“左深”查询计划（其中连接的右手输入必须是基表），  
以及“推迟笛卡尔积”（确保笛卡尔积只在数据流中的所有连接之后出现）来限制其计划空间。

在今天的商业系统中，众所周知“灌木状”树（带有嵌套的右手输入）和早期使用笛卡尔积在某些情况下可能是有用的。因此，大多数系统在某些情况下都考虑了这两个选项。

#### 选择性估计
大多数系统现在通过直方图和其他摘要统计分析和总结属性中值的分布。由于这涉及到访问每个列中的每个值，  
这可能相对昂贵。因此，一些系统使用抽样技术来估计分布，而不需要全面扫描的费用。

可以通过“连接”连接列上的直方图来计算基表连接的选择性估计。为了超越单列直方图，最近提出了更复杂的方案来纳入列之间的依赖性问题。
>  A. Desphande, M. Garofalakis, and R. Rastogi, “Independence is good:  
Dependency-based histogram synopses for high-dimensional data,” in Proceed-  
Ings of the 18th International Conference on Data Engineering, San Jose, CA,  
February 2001.
V. Poosala and Y. E. Ioannidis, “Selectivity estimation without the attribute  
Value independence assumption,” in Proceedings of 23rd International Confer-  
Ence on Very Large Data Bases (VLDB), pp. 486–495, Athens, Greece, August  
1997.

#### 搜索算法
一些商业系统，特别是 Microsoft 和 Tandem 的系统，放弃了 Selinger 的动态规划优化方法，转而使用基于 Cascades 中使用的技术的目标导向“自顶向下”搜索方案。
>  G. Graefe, “The cascades framework for query optimization,” IEEE Data Engi-  
Neering Bulletin, vol. 18, pp. 19–29, 1995.

不幸的是, 不管是自顶向下还是动态规划, 它们的运行时间和内存需求都于一次查询中所涉及表的数量成指数关系. 针对设计太多表的查询, 一些系统会采用启发式查询的机制. 
> K. P. Bennett, M. C. Ferris, and Y. E. Ioannidis, “A genetic algorithm for database query optimization,” in Proceedings of the 4th International Conference on Genetic Algorithms, pp. 400–407, San Diego, CA, July 1991.

The heuristics used in commercial systems tend to be proprietar, An educational exercise is to examine the query “optimizer” of the open-source MySQL engine, which at last check was entirely heuristic and relied mostly on exploiting indexes and key/foreign-key constraints.
- This is reminiscent of early (and infamous) versions of Oracle. In some systems, a query with too many tables in the FROM clause can only be executed if the user explicitly directs the optimizer how to choose a plan (via so-called optimizer “hints” embedded in the SQL).

#### 并行性
今天每个主要的商业DBMS都对并行处理有一些支持。大多数也支持“查询内部”并行处理：通过使用多个处理器来加快单个查询的能力。

查询优化器需要介入确定如何在多个CPU上调度操作符——以及并行化操作符——以及（在无共享或共享磁盘情况下）如何在多台独立电脑上调度。

##### Two phases
Hong and Stonebraker [42] chose to avoid the parallel optimization complexity problem and use two phases: 
1. First a traditional single-system optimizer is invoked to pick the best single-system plan, 
2. And then this plan is scheduled across multiple processors or machines. 

>  W. Hong and M. Stonebraker, “Optimization of parallel query execution plans  
In xprs,” in Proceedings of the First International Conference on Parallel  
And Distributed Information Systems (PDIS), pp. 218–225, Miami Beach, FL,  
December 1991.


#### Auto-Tuning

工业研究中正在进行的各种努力试图提高数据库管理系统自动做出调整决策的能力。这些技术的一些基于收集查询工作负载，然后使用优化器通过各种“如果”分析来找到计划成本。例如，如果存在其他索引或者数据布局不同会怎样？优化器需要进行一些调整以支持这项活动， as described by Chaudhuri and Narasayya [12]. The Learning Optimizer (LEO) work of Markl et al. [57] is also in this vein.

>S. Chaudhuri and V. R. Narasayya, “Autoadmin ‘what-if’ index analysis util-  
Ity,” in Proceedings of ACM SIGMOD International Conference on Manage-  
Ment of Data, pp. 367–378, Seattle, WA, June 1998.

>  V. Markl, G. Lohman, and V. Raman, “Leo: An autonomic query optimizer  
For db2,” IBM Systems Journal, vol. 42, pp. 98–106, 2003.

### A Note on Query Compilation and Recompilation
SQL supports the ability to “prepare” a query: to pass it through the parser, rewriter and, optimizer, store the resulting query execution plan, and use it in subsequent “execute” statements. This is even possible for dynamic queries (e. G., from web forms) that have program variables in the place of query constants. 
The only wrinkle is that during selectivity estimation, the variables that are provided by the forms are assumed by the optimizer to take on “typical” values. When non-representative “typical” values are chosen, extremely poor query execution plans can result.
Query preparation is especially useful for form-driven, canned queries over fairly predictable data: the query is  prepared when the application is written, and when the application goes live, users do not experience the overhead of parsing, rewriting, and optimizing.

Although preparing a query when an application is written can improve performance, this is a very restrictive application model. Many application programmers, as well as toolkits like Ruby on Rails, build SQL statements dynamically during program execution, so precompiling is not an option. Because this is so common, DBMSs store  
These dynamic query execution plans in the query plan cache. If the same (or very similar) statement is subsequently submitted, the cached version is used. This technique approximates the performance of precompiled static SQL without the application model restrictions and is heavily used.

As a database changes over time, it often becomes necessary to reoptimize prepared plans. At a minimum, when an index is dropped, any plan that used that index must be removed from the stored plan cache, so that a new plan will be chosen upon the next invocation.

Other decisions about re-optimizing plans are more subtle, and expose philosophical distinctions among the vendors, This philosophical distinction arises from differences in the historical customer base for these products.


SQL支持“准备”查询的能力：通过解析器、重写器和优化器，存储生成的查询执行计划，并在随后的“执行”语句中使用它。即使是动态查询（例如来自网络表单）那些在查询常量的位置有程序变量的也可以这样做。  

唯一的问题在于，在进行选择性估计时，表单提供的变量被优化器假定为“典型”值。当选择了不具代表性的“典型”值时，可能会导致非常糟糕的查询执行计划。  
查询准备对于由表单驱动的、针对相对可预测数据的预设查询特别有用：查询在编写应用程序时被准备，在应用程序上线时，用户不会体验到解析、重写和优化的开销。

尽管在编写应用程序时准备查询可以提高性能，但这是一个非常受限的应用模型。许多应用程序程序员，以及像Ruby on Rails这样的工具包，在程序执行期间动态构建SQL语句，因此预编译不是一个选项。因为这种做法非常普遍，数据库管理系统在查询计划缓存中存储这些动态查询执行计划。如果随后提交了相同（或非常相似）的语句，就会使用缓存的版本。这种技术在没有应用程序模型限制的情况下近似地实现了预编译静态SQL的性能，并且在工业上被广泛使用。

随着数据库随时间变化，经常需要重新优化准备的计划。至少，当索引被删除时，任何使用该索引的计划都必须从存储计划缓存中移除，以便在下一次调用时选择新计划。

其他关于重新优化计划的决定更加微妙，并且暴露了供应商之间的哲学区别。这种哲学区别源于这些产品的历史客户基础的差异。