---
tags:
  - db/plan/optimizer
---

> [Unnesting Arbitrary Queries](zotero://select/library/items/5D8RVWIZ)

在2015年那篇著名的Unnesting Arbitrary Queries出现之前，并没有一种方法能解开所有的子查询，因此各家数据库对无法解开的子查询仍然保留了以嵌套方式执行的算子，一般称作apply join。

https://www.zhihu.com/search?type=content&q=Unnesting%20Arbitrary%20Querie

# 背景与动机

在SQL查询中，嵌套子查询（nested subqueries）虽然可以简化复杂查询的编写，但常常导致依赖连接（dependent joins）

corelated subquery, 关联子查询

- 嵌套子查询在许多 SQL 语句中会导致性能问题，尤其是相关子查询。因为对于外部查询的每一行，相关子查询可能需要重新执行，这会产生大量的额外计算，降低性能。
- 在复杂查询中，可能会出现多层嵌套，使得查询执行计划复杂，难以优化。


## 什么是依赖连接
依赖连接是一种特殊的连接操作，其中右侧的关系（即子查询）依赖于左侧关系的当前元组。具体来说，对于左侧关系中的每一个元组，右侧关系都需要重新计算以生成与之匹配的结果。

依赖连接可以形式化为：
$$
T_1 \bowtie_C p T_2 := \{t_1 \cdot t_2 | t_1 \in T_1 \land t_2 \in T_2(t_1) \land p(t_1 \cdot t_2)\}
$$
其中
- $T_1$ 和 $T_2$ 是两个关系
- $t1$ 是 $T_1$ 中的一个元组, $t_2$ 同理, 且后者的生成依赖于前者
- $p$ 是一个谓词，用于筛选满足条件的元组对

在依赖连接中，右侧关系 $T_2$ 的计算是动态的，需要为左侧关系 $T_1$ ​ 中的每一个元组重新进行, 所以会导致较高的时间复杂度, 例如，以下查询Q1：
```sql
SELECT
    s.name,
    e.course
FROM
    students s,
    exams e
WHERE
    s.id = e.sid
    AND e.grade = (
        SELECT
            MIN(e2.grade)
        FROM
            exams e2
        WHERE
            s.id = e2.sid
    );
```

子查询 `(select min(e2.grade) from exams e2 where s.id = e2.sid)` 用于获取每个学生的最低成绩, 它依赖于外部查询中的 `s.id` 。

现有的数据库系统虽然采用了一些启发式方法来解嵌套 (unnest)，但这些方法通常只能处理特定类型的查询，无法处理更复杂的嵌套子查询。因此，本文提出了一种通用的解嵌套方法，能够处理任意类型的嵌套子查询，从而显著提升查询性能。这里有非常多的关系代数的表达式的内容, 可以参阅[[理论/数据库系统/Bottom Up/1-Plan/逻辑计划综述#关系代数基础|关系代数基础]], 这里的演练相当于是加餐了

## 论文讨论时用到的DDL

示例中所用到的 `students` 和 `exams` 的表, 可以把它理解为下面的样子
```sql
-- 创建 students 表
CREATE TABLE students (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    major VARCHAR(255),  -- 专业
    year INT             -- 入学年份
);

-- 创建 exams 表
CREATE TABLE exams (
    sid INT,
    course VARCHAR(255),     -- 考试科目名称
    curriculum VARCHAR(255), -- 专业
    date DATE,
    grade INT,
    FOREIGN KEY (sid) REFERENCES students(id)
);
```

## 一般原则

实际上, 基本的原理就是通过谓词上移来做到的, 然后对于其他更复杂的情况, 将其转换为谓词上移能够处理的方式, 下面先来简单地看一下什么是谓词上移动

### 谓词上移

这种方法适用于那些因语法原因而引入依赖性的查询, 我们以 TPC-H Query 21 为例子
```sql
SELECT
...
FROM
    lineitem l1
WHERE
    EXISTS (
        SELECT
            *
        FROM
            lineitem l2
        WHERE
            l2.l_orderkey = l1.l_orderkey
    );
```

在这个查询中，子查询的存在仅仅是为了检查是否存在某个条件。这种结构在代数表达式中被翻译为下面这种格式, 注意上角标 d 表示依赖的意思, 或者左边是实心的, 不过我绘制不出来, 就放弃了
$$
l_1  ⋉^d (\sigma_{l_1.okey=l_2.okey}(l_2))
$$

这里有一个字符画 (参考自知乎 id 为[不会游泳的鱼](https://zhuanlan.zhihu.com/p/613010202)), 演示了谓词上移之后, 我们期待的结果是什么样子的
```
l1 ⋉ (σ[l1.okey=l2.okey](l2))   ====>   l1 ⋉[l1.okey=l2.okey] l2

            ⋉ᵈ                          σ[l1.okey=l2.okey]              σ[l1.okey=l2.okey]
           / \                                 |                              |
         l1   σ[l1.okey=l2.okey]    ====>      ⋉ᵈ         ====>               ⋉
                     |                        / \                            / \
                    l2                      l1   l2                        l1   l2
```

也就是通过将依赖谓词上移，可以将依赖连接转换为普通连接：
$$
l_1  ⋉_{l_1.okey=l_2.okey}(l_2)
$$
将嵌套子查询中的谓词（如 Q1 中的 `s.id = e2.sid` ）提升到更高层次的操作中，直到不再依赖外查询变量，从而将依赖连接转换为普通的连接，抽出来这部分谓词, 通过额外的分组和连接操作来替代.

# 简单解嵌套(Simple Unnesting)

简单解嵌套, 按照论文的意思, 大体上指代的就是那种人眼也能看出来应该怎么进行谓词上移的情况

## 先做一遍 Q1 : 简单解嵌套

以 Query Q1 为例, 这个查询的目的是为每个学生找到最好的考试成绩的对应科目 (根据德国评分系统，分数越低越好), 这个例子还是可以用过人工的改写, 写出等效的 `uncorelated subquery` 的, 但是我们还是"装模作样"地走一下流程, 感受一下谓词上移的效果

```sql
SELECT
    s.name,
    e.course
FROM
    students s,
    exams e
WHERE
    s.id = e.sid
    AND e.grade = (
        SELECT
            MIN(e2.grade)
        FROM
            exams e2
        WHERE
            s.id = e2.sid
    );
```

### 初始代数表达式

用 tex 的语法写, 就是这个样子的, 不过我不会在显示的时候控制缩进, 所以是一整行. 读着很困难
$$
\sigma_{e.grade = m} (
    students~s 
        \bowtie_{s.id = e.sid} (
            \Gamma_{\emptyset ; m: min(e2.grade)} (
                \sigma_{s.id = e2.sid} (
                    e2
                )
            )
        )
)
$$
挨个解释一下
- 最内层 $σ_{s.id = e2.sid}(e2)$ ：
	- 选择操作，从表 e2 中选择满足 s.id = e2. sid 的行
- $Γ_{\varnothing ; m: min(e2.grade)}$
	- 分组聚合操作
	- $\varnothing$ 表示不按任何列分组（全局聚合）
	- $m: min (e2. grade)$ 表示计算 $e2.grade$ 的最小值，结果命名为 $m$
- $students~s \bowtie_{s.id = e.sid}$
	- 连接操作，将 students 表（别名 s）与前面的结果进行连接
	- 连接条件是 $s.id = e.sid$
- 最外层 $\sigma_{e.grade = m}$
	- 选择操作，选择 grade 等于最小值 m 的行

为了控制缩进, 后面统一用代码段来进行表述, 这样可视化的效果会更好, 转换后的原始的关系代数表达式大体上长下面这样 (后面会有原作者的展示)
```
Π[name, course] (
    σ[
        (students s.id = exams e.sid) 
        ∧ 
        (e.grade = (
            Γ[sid] (
                σ[s.id = exams e2.sid] (exams)
            )[MIN(grade)]
        ))
    ] (students ⨯ exams)
)
```

我们看到 Q1 中包含一个子查询 `(select min(e2.grade) from exams e2 where s.id=e2.sid)` , 子查询被表示为
```
(e.grade = (
    Γ[sid] (
        σ[s.id = exams e2.sid] (exams)
    )[MIN(grade)]
))
```
即对 `exams` 表先进行选择操作 `σ[s.id = e2.sid]` ，筛选出满足 `s.id = e2.sid` 的行，然后进行分组操作 `Γ` ，对筛选后的结果按 `sid` 分组并计算每组中 `grade` 的最小值，将结果命名为 `min_grade`

需要注意, 对于 `students` 表中的每一行，都要做上面👆这个操作, 这就是依赖的地方

### 解嵌套

把子查询的关系代数操作从嵌套位置提升到与主查询的连接操作同一层次
```
Π[name, course] (
    (students s ⨝ (
        Γ[s.id, MIN(grade) AS min_grade] (exams)
    )) ⨝ (
        σ[s.id = exams e.sid ∧ e.grade = min_grade] (exams)
    )
)
```


1. `Γ[sid, MIN(grade) AS min_grade] (exams)`
	- 这部分计算了 `exams` 表中每个学生（根据 `sid` 分组）的最小成绩，并将结果命名为 `min_grade`
2. `(students ⨝ (Γ[sid, MIN(grade) AS min_grade] (exams))`
	- 将 `students` 表和计算了最小成绩的 `exams` 表结果进行自然连接，将每个学生与其最小成绩关联起来
3. `σ[s.id = e.sid ∧ e.grade = min_grade] (exams)`
   - 对 `exams` 表进行选择操作, 拿到最好的成绩所对应的 `course`

- 在原始表达式中，有一个嵌套的子查询结构，部分谓词是在子查询内部的
	- 具体来说，在 `σ[s.id = e2.sid] (exams)` 中的 `s.id = e2.sid` 是作为子查询的一部分。
- 在解嵌套后的表达式中，这个条件被 “上移” 并被重新组织
	- 我们将 `Γ[sid, MIN(grade) AS min_grade] (exams)` 操作单独拿出来，对 `exams` 表根据 `sid` 进行分组并计算 `MIN(grade)` 作为 `min_grade`
	- 此时，原本在子查询中的 `s.id = e2.sid` 条件，实际上是在分组操作 `Γ[sid, MIN(grade) AS min_grade] (exams)` 中隐式地被考虑了
	- 因为分组是基于 `sid` 的，这里将其与 `students` 表关联起来，将原本子查询中的部分条件上移到了更高层次的操作中
### 小结

再总结一下, 解嵌套的过程是
1. **识别依赖关系**
	- 原本的子查询 `(select min(e2.grade) from exams e2 where s.id = e2.sid)` 依赖于外部查询中的 `s.id`
2. **消除依赖**
	- 将子查询转换为一个临时表或派生表，该表包含每个学生的最低成绩。
3. **转换为常规连接**
	- 将子查询转换为一个派生表 `min_grades` ，其结构为 `{sid, min_grade}` 。
	- 然后将原查询转换为连接查询

最终得到结果是类似这样的
```sql
-- 优化后的SQL
SELECT
    s.name,
    e.course
FROM
    students s,
    exams e,
    (
        SELECT
            e2.sid AS id,
            MIN(e2.grade) AS min_grade
        FROM
            exams e2
        GROUP BY
            e2.sid
    ) AS min_grades
WHERE
    s.id = e.sid
    AND e.grade = min_grades.min_grade
    AND min_grades.id = s.id;
```

通过这些步骤，Query Q1 被成功解嵌套，消除了依赖连接, 后续可以进一步再优化, 不过这不是解开子嵌套的内容

在 Q1 中，将其上移到分组操作 `Γ[sid, MIN(grade) AS min_grade] (exams)` 中，通过分组和连接操作来替代子查询。

# 通用解嵌套(General Unnesting)

对于一些更复杂的查询，可能存在多个依赖连接和多层嵌套, 我们就没有办法一眼看出来应该怎样进行相关子查询的展平了, 形式化地对其进行解决, 就是这篇论文的创新点

比如说 Q2, 查询专业为“计算机科学”或“游戏工程”的学生, 这些学生的考试成绩必须低于以下两部分的平均成绩 1 分 (德国越低越好, again)
- 他们自己参加的所有考试的平均成绩。
- 与他们专业相同但入学年份更早的学生的考试成绩
```sql
SELECT
    s.name,
    e.course
FROM
    students s,
    exams e
WHERE
    s.id = e.sid
    AND (
        s.major = 'CS'
        OR s.major = 'Games Eng'
    )
    AND e.grade >= (
        SELECT
            AVG(e2.grade) + 1 -- one grade worse
        FROM
            exams e2          -- than the average grade
        WHERE
            s.id = e2.sid     -- of exams taken by
            OR (
                e2.curriculum = s.major  -- him/her or taken
                AND s.year > e2.date     -- by elder peers
            )
    );
```

Q2的复杂性在于子查询中存在多个条件（ `s.id = e2.sid` 和 `e2.curriculum = s.major ∧ s.year > e2.date` ），这些条件无法直接通过简单的谓词上移消除依赖关系。

```sql
SELECT
    AVG(e2.grade) + 1 -- one grade worse
FROM
    exams e2          -- than the average grade
WHERE
    s.id = e2.sid     -- of exams taken by
    OR (
        e2.curriculum = s.major  -- him/her or taken
        AND s.year > e2.date     -- by elder peers
    )
```

论文提出的通用解嵌套框架可以解决这个问题, 它总体上分两步

## 步骤1：转换依赖连接

论文首先将依赖连接转换为一种更易于操作的形式。具体来说，依赖连接被转换为以下形式：
用论文的公式, 表现出来是这样的, 其中 $D := \Pi_{\mathcal{F}(T_2) \cap \mathcal{A}(T_1)}(T_1)$

$$
T_1 ⧑_p T_2 \quad \equiv \quad T_1 \Join_{p \land T_1 =_{\mathcal{A}(D)} D} (D ⧑ T_2)
$$
这里开始对笔者有些不太友好, 所以笔者会详细且重复地介绍这里的符号的含义...

- $\mathcal{A}$ 用于表示属性集合 `the attributes produced by an expression T` 
	- $\mathcal{A}(students)$ 就是 `[id, name, major, year]`
- $\mathcal{F}$ 用于表示自由变量集合, `free variables occurring in an expression T`
	- 在关系代数的查询语境中，自由变量是指那些其取值依赖于其他关系的变量
	- 所以在 `Q1` 中, $T_1$ 是 `student` , $T_2$ 是 `exams` , `exams.sid` 依赖 `student.id` , 所以 $\mathcal{F}(T_2)$ 包含 `sid`
 - $\mathcal{F}(T_2) \subseteq \mathcal{A}(T_1)$ , 表示 `the attributes required by T2 must be produced by T1` , 通过这个来体现关联性
	 - 因为 $\mathcal{F}(T_2)$ 包含 `sid` (与 student 中的 idduiying ) , $\mathcal{A}(students)$ 就是 `[id, name, major, year]` , 所以 $\mathcal{F}(T_2) \cap \mathcal{A}(T_1)$ 的结果就是 `s.id` 
 - $D := \Pi_{\mathcal{F}(T_2) \cap \mathcal{A}(T_1)}(T_1)$ 
	 - $D$ 的作用是携带外部查询的上下文信息到内部查询
	 - 表示 $D$ 是通过对 $T_1$ 进行投影操作得到的，投影的属性是 $T_2$ 中的自由变量 $\mathcal{F}(T_2)$ 与 $T_1$ 的属性集 $\mathcal{A}(T_1)$ 的交集
	 - 而 $\mathcal{F}(T_2) \cap \mathcal{A}(T_1)$ 的结果就是 `s.id` , 所以 $D$ 是对 $T_1$ (students) 做投影, 投射出 `s.id` 列, 去除了其他无关的属性, 只保留 `exams` 需要的 `s.id` 

在后面的详细例子中, 能培养对于 $D$ 的更多感觉, 看不太懂也不影响往后看, 看到后面就懂了, 这里可以就记住==D的作用是携带外部查询的上下文信息到内部查询==, 看完 [[理论/数据库系统/Bottom Up/1-Plan/Optimizer/Unnesting Arbitrary Queries#Step 1 连接下推|Step 1]] 还看不懂就再休息休息, 笔者也是组织到那么后面的地方的时候才豁然开朗的, 懂了一次 D, 然后又懂了一次.

***

每一个符号都讨论清晰之后, 再整体地把握一下这个代换公式 (别怕，论文后面有例子, 本文后面也有)

1. 整体等价规则含义：
   - 原公式 $T_1 ⧑_p T_2 \equiv T_1 \Join_{p \land T_1 =_{\mathcal{A}(D)} D} (D ⧑ T_2)$ 是在描述一种将依赖连接（ $T_1 ⧑_p T_2$ ）转换为另一种形式的等价关系。其中 $T_1$ 和 $T_2$ 是两个关系， $p$ 是原来的连接条件。
2. 详细解释 $p \land T_1 =_{\mathcal{A}(D)} D$ ：
   - $p$ ： $p$ 是原来依赖连接 $T_1 ⧑_p T_2$ 中的连接条件, 在 Q1 中, $p$ 就是 `s.id = e.sid`
   - $T_1 =_{\mathcal{A}(D)} D$ ：这部分是一个新增加的条件, $\mathcal{A}(D)$ 表示 $D$ 的属性集合, 
		- $T_1 =_{\mathcal{A}(D)} D$ 意味着在 $D$ 的属性集 $\mathcal{A}(D)$ (也就是 `s.id` ) 上, $T_1$ 和 $D$ 的取值要相等
	- $p \land T_1 =_{\mathcal{A}(D)} D$ 的整体作用：它是转换后的连接操作 $T_1 \Join_{p \land T_1 =_{\mathcal{A}(D)} D} (D ⧑ T_2)$ 中的连接条件。它要求在进行 $T_1$ 和 $(D ⧑ T_2)$ 的连接时，既要满足原来的连接条件 $p$ ，又要满足在 $D$ 的属性集上 $T_1$ 和 $D$ 相等的条件。


```
   ⧑_p                        ⨝_{p ∧ T1 =_{A(D)} D}
  /    \                      /          \
T1      T2        ===>      T1           ⧑_p
                                        /    \
                                      D      T2
                                      |
                              Π_{F(T2) ∩ A(T1)}(T1)
```

这种转换咋一看有脱裤子放屁的嫌疑, 但实际上, 在引入了 `D` 之后 —— 虽然依然是一个关联子查询, 但实际上 $T_2$ 的调用次数已经从 $T_1$ 的每一行都要扫一遍, 如果 $T_1$ 有很多变量是重复的话, 那会很糟糕, 不过关系代数 (这个基于集合的理论) 在做投影的时候是会自动进行去重的. 所以现在它只会为每个不同的值执行一次关联子查询了.

***

下面是摘抄自论文的部分, 作者借用上面的 Q1 的 SQL 的关系代数表达式来说明引入 `D` 后的不同, 虽然它缺少了最终的投影, 但不影响示意
```
-- before
σ[e.grade = m] (
	(students s ⋈ s.id = e.sid exams e) ⧑
	(Γ∅; m: min(e2.grade) (σ[s.id = e2.sid] exams e2))
)
```

在原始表达式中
- `T1` 是 `(students s ⋈ s.id = e.sid exams e)`。 
- `T2` 是 `(Γ∅; m: min(e2.grade) (σ[s.id = e2.sid] exams e2))`。
- `T2` 的计算依赖于 $T_1$ 的 `s.id` , 因为 $T_2$ 需要 `s.id` 来筛选 `exams e2` 中的记录。因此：
	- $\mathcal{F}(T_2)=\{s.id\}$ ( $T_2$ 的计算依赖于 `s.id` )
	- $\mathcal{A}(T_1) = \{s.id,e.sid,e.grade,...\}$ ( $T_1$ 的属性集合)
	- 它们的交集是： $\mathcal{F}(T_2) \cap \mathcal{A}(T_1)$ 的结果就是 $\{s.id\}$
	- `D` 的定义是 `Π[d.id: s.id] (students s)
		- `s.id` 是 `students` 表中的原始属性。
		- 通过重命名 `s.id` 为 `d.id` ，我们实际上创建了一个新的命名空间
			- 原来依赖于 `s.id` 的地方都被替换成了 `d.id`

所以在带入转换之后, 大体上长这个样子
```
-- after
σ[e.grade = m](
    (
        (students s ⋈ s.id = e.sid exams e) ⋈
        (
            Π[d.id: s.id] (students s) ⧑
            (
                Γ[d.id]; m: min(e2.grade) (
                    σ[d.id = e2.sid] exams e2
                )
            )
        )
    )
)
```

论文中的第一幅图可视化了这个转换关系, 这里用字符画来意思一下, 感兴趣的看原图 `Figure.1`

```
                σ[e.grade=m]                                               σ[e.grade=m]
                     |                                                          |
                     ⧑                                                     ⋈[s.id=d.id]
                   /   \                                                   /          \
         ⋈[s.id=e.sid]   Γ[∅;m:min(e2.grade)]   ======>                  /             ⧑
        /            \           |                                      /            /    \
students s       exams e    σ[s.id=e2.sid]                            /      Π[d.id:s.id]  Γ[∅;m:min(e2.grade)]
                                 |                                  /        /                     |
                              exams e2                             |       /                 σ[d.id=e2.sid]
                                                             ⋈[s.id=e.sid]                         |
                                                             /          \                       exams e2
                                                      students s       exams e 
```

- `Π[d.id:s.id]` 到 `⋈[s.id=e.sid]` 之间有一根咋一看上去有点怪怪的线 ^4c262f
	- 单纯从字面意义上理解
		- 这根线的意思就是从 `students ⋈ exams` 的结果中获取 `s.id` 并重命名为 `d.id`
	- 实际上, 这根线代表了依赖关系
		- "我需要从这个连接结果中获取学生 ID，用于后续的操作"
			- 具体来说，就是需要把 `s.id` (通过 `d.id` ) 传递给子查询


### 小结

这里的总结非常精炼, 相关 [link](https://zhuanlan.zhihu.com/p/613010202)

- 例如 $T_1 ⧑_p T_2$ 中，从 $T_1$ 中按照一定规则投影出一个关系 $D$
- 先执行 $D ⧑ T_2$ ，结果用 $D'$ 表示， $D'$ 与 $D$ 的属性集相同
- 再执行 $T_1 ⧑_p ⋀ T_1 = A(D') D’$ ，这里 join 谓词 $p$ 同 $T_1 ⧑p T_2$ ，新增了一个 selection 条件， $T_1 =A(D'), D':= ∀t∈T_1 : t.a = d.a.$

### 意义

除了减少了一些可能有的重复值的计算 ( $D$ 肯定是不包含重复的, 论文作者强调了包含在基本关系中的所有重复项以及由查询生成的重复项，在优化后的计划中都会保留, 或者说等价)

除此之外, 更重要的事情是, 引入 $D$ 之后, **==我们将依赖连接下推了!==**, 你注意 D 的作用是携带外部查询的上下文信息到内部查询, 而能推一次, 就能推很多次可以说, 我们的终极目标就是把谓词查询给彻底剔除, 当 $\mathcal{F}(T) \cap \mathcal{A}(D) = \emptyset$ 的时候就可以做到把依赖连接给完全剔除

$$
D ⧑ T \equiv D \bowtie T \text{ if } \mathcal{F}(T) \cap \mathcal{A}(D) = \emptyset
$$

后面的论述中能看到, 总是能够将依赖连接给优化掉, 并且更好的是, 我们有可能用一些现有的属性取代一些 join

## 步骤2：将新的依赖连接下推到查询中

接下来，论文将新的依赖连接下推到查询中，直到可以将其转换为普通连接, 不同的算子的下推方式可能会不一样, 下面分类讨论一下, 不过轻舟已过万重山了

### 下推到选择

$$
D ⧑ \sigma_p(T_2) \equiv \sigma_p(D ⧑ T_2)
$$
虽然这种转换看起来可能不符合常规的选择下推思路 (通常希望先下推选择操作), 但在依赖连接解嵌套的过程中，我们首先要尽可能地下推依赖连接，直到其被完全消除。
一旦依赖连接消除后，就可以使用像选择下推这样的常规技术来对转换后的查询进行重新优化, 后面的 `group by` 也是类似的

### 下推到 join

将依赖连接向下推到另一个连接更复杂，因为连接操作的双方都可能对依赖连接的下推产生影响, 不过在这之前, 要先补充一点说明

#### 先补充关于 natural join 的说明

> Note that in this paper we sometimes explicitly mention natural join in the join predicate to simplify the notation. We assume that all relations occurring in a query will have unique attribute names, even if they reference the same physical table, thus $A \bowtie B \equiv A \times B$. However, if we explicitly reference the same relation name twice, and call for the natural - join, then the attribute columns with the same name are compared, and the duplicate columns are projected out.

1. 自然连接的定义：在关系代数中，自然连接是两个关系（表）之间的一种特殊类型的连接，它基于两个关系中具有相同名称的属性（列）进行匹配。如果两个关系中存在同名的属性，自然连接会自动匹配这些属性，并在结果中只保留一份。
2. 简化表示法：在论文中，为了简化表示法，作者有时会在连接谓词中明确提到自然连接。这意味着如果两个关系中有同名的属性，它们会自动进行匹配和连接。
3. 唯一属性名的假设：论文假设查询中涉及的所有关系（表）都有唯一的属性名，即使这些关系引用的是同一个物理表。这是因为在关系代数中，我们通常用关系（表）的模式（即属性集合）来表示它们，而不是物理表本身。
4. 处理重复列：如果我们在查询中明确地两次引用了同一个关系名，并要求进行自然连接，那么具有相同名称的属性列会被比较，并且结果中会消除重复的列。这意味着在结果关系中，每个属性名只会出现一次。

考虑这样一个例子 $(A \bowtie C) \bowtie_{p \land \text{natural join } C} (B \bowtie C)$ 
- 首先, 关系 A 和关系 C 进行连接。这里的连接可能基于 A 和 C 之间的某些公共属性或条件。
- 然后, 关系 B 和关系 C 进行连接。同样，这个连接可能基于 B 和 C 之间的某些公共属性或条件。
- 最后，将上述两个结果进行连接。这里的连接条件除了 p 以外, 还会进行自然连接，即比较 C 中相同名称的列，并将重复的列投影出去。

#### 等值 join

回来回来, 我们看这个分类讨论

$$
D ⧑ (T_1 \bowtie_p T_2) \equiv 
\begin{cases}
(D ⧑ T_1) \bowtie_p T_2 & : \mathcal{F}(T_2) \cap \mathcal{A}(D) = \emptyset \\
T_1 \bowtie_p (D ⧑ T_2) & : \mathcal{F}(T_1) \cap \mathcal{A}(D) = \emptyset \\
(D ⧑ T_1) \bowtie_{p \wedge \text{natural join } D} (D ⧑ T_2) & : \text{otherwise}
\end{cases}
$$

分类讨论
1. $\mathcal{F}(T_2) \cap \mathcal{A}(D) = \emptyset$
	- 当 $T_2$ 的自由变量集合与 $D$ 的属性集合的交集为空时，意味着 $T_2$ 的计算不依赖于 $D$ 的任何属性。
		- 此时，依赖连接 $D ⧑ (T_1 \bowtie_p T_2)$ 可以等价转换为 $(D ⧑ T_1) \bowtie_p T_2$
			- 因为 $T_2$ 已经没什么好解的了
1. $\mathcal{F}(T_1) \cap \mathcal{A}(D) = \emptyset$
	- 与第一种情况对偶
2. otherwise
	- 即 $T_1$ 和 $T_2$ 的计算都依赖于 $D$ 的属性, 那就一起往下推

#### 下推到外连接操作

外连接分为左外连接、右外连接和全外连接。以左外连接为例

> For outer joins we always have to replicate the dependent join if the inner side depends on it, as otherwise we cannot keep track of unmatched tuples from the outer side

$$
D ⧑ (T_1 ⟕_p T_2) \equiv 
\begin{cases}
(D ⧑ T_1) ⟕_p T_2 & : \mathcal{F}(T_2) \cap \mathcal{A}(D) = \emptyset \\
(D ⧑ T_1) ⟕_{p \wedge \text{natural join } D} (D ⧑ T_2) & : \text{otherwise}
\end{cases}
$$

对于全外连接, 需要直接进行两边的扩展
$$
D ⧑ (T_1 ⟗_p T_2) \equiv (D ⧑ T_1) ⟗_{p \wedge \text{natural join } D} (D ⧑ T_2)
$$

#### 下推到半连接和反连接：

对于半连接是类似的
$$
D ⧑ (T_1 \ltimes_p T_2) \equiv 
\begin{cases}
(D ⧑ T_1) \ltimes_p T_2 & : \mathcal{F}(T_2) \cap \mathcal{A}(D) = \emptyset \\
(D ⧑ T_1) \ltimes_{p \wedge \text{natural join } D} (D ⧑ T_2) & : \text{otherwise}
\end{cases}
$$

反半连接也差不多, 是类似的
$$
D ⧑ (T_1 \triangleright_p T_2) \equiv 
\begin{cases}
(D ⧑ T_1) \triangleright_p T_2 & : \mathcal{F}(T_2) \cap \mathcal{A}(D) = \emptyset \\
(D ⧑ T_1) \triangleright_{p \wedge \text{natural join } D} (D ⧑ T_2) & : \text{otherwise}
\end{cases}
$$

join 就这么多内容

### 下推到分组操作

> When pushing the dependent join down a _group-by_ operator, the group-operator must preserve all attributes produces by the dependent join

$$
D ⧑ (\Gamma_{A;a:f}(T)) \equiv \Gamma_{A \cup \mathcal{A}(D);a:f}(D ⧑  T)
$$
### 下推到投影

> The projection behaves similar to the group by operator

$$
D ⧑ (\Pi_A(T)) \equiv \Pi_{A \cup \mathcal{A}(D)}(D ⧑  T)
$$

### 集合操作
$$
\begin{align*}
D ⧑ (T_1 \cup T_2) & \equiv (D ⧑ T_1) \cup (D ⧑ T_2) \\
D ⧑ (T_1 \cap T_2) & \equiv (D ⧑ T_1) \cap (D ⧑ T_2) \\
D ⧑ (T_1 \setminus T_2) & \equiv (D ⧑ T_1) \setminus (D ⧑ T_2)
\end{align*}
$$

### 小结

通过这些规则, 别忘了还有 ( $D ⧑ T \equiv D \bowtie T \text{ if } \mathcal{F}(T) \cap \mathcal{A}(D) = \emptyset$ ), 依赖连接逐步被下推到查询的底层，直到可以将其转换为普通连接

对于依赖连接下推这种方法，一个潜在的担忧是集合 $D$ 可能会变得非常大。这是因为 $D$ 是嵌套子查询的所有变量绑定的集合。在复杂的嵌套查询场景中，子查询可能需要处理大量的变量和数据，这可能会让人担心 $D$ 的规模会失控，进而影响查询的性能和资源消耗

然而，幸运的是实际情况并非如此。如果原始的嵌套连接是 $T_1 \bowtie T_2$ ，那么有 $|D| \leq |T_1|$ ，即 $D$ 的大小不会超过 $T_1$ 的大小。这是一个重要的性质，它从理论上限制了 $D$ 的规模，使得我们不必过度担心 $D$ 会无限增大
例如，在 `Q1` 中，假设 $T_1$ 是学生表， $T_2$ 是成绩表， $D$ 是基于学生表生成的用于优化连接的集合，那么 $D$ 的记录数不会超过学生表的记录数

如果在对子查询进行去相关处理后，最顶层的连接是哈希连接，并且将 $T_1$ 存储在哈希表中，那么最糟糕的情况下, 计算 $D$ 最多只会使该连接的内存消耗翻倍
如果我们知道 $T_1$ 中的值是无重复的，比如 $T_1$ 中包含一个键（主键或唯一键），那么我们甚至可以避免物化 $D$ ，而是直接读取连接哈希表，这样就消除了计算 $D$ 带来的任何额外开销。

从积极的方面来看，依赖连接下推方法通过引入 $D$ ，将原本时间复杂度为 $O(n^2)$ 的操作（例如在嵌套循环连接中，随着数据量 $n$ 的增加，操作次数呈平方增长）理想情况下转化为 $O(n)$ 的操作（例如在一些优化后的线性扫描或单次哈希查找操作中）。这种时间复杂度的优化是非常有价值的，即使在最糟糕的情况下会有一定的内存开销，但相较于性能的大幅提升，这种内存开销是值得的。


具体步骤如下：
1. **识别依赖谓词**：找到依赖连接中涉及的谓词。
2. **上移谓词**：将这些谓词向上移动到查询树的更高层次，直到它们不再依赖于外部查询的变量。
3. **转换连接类型**：当谓词被上移后，依赖连接可以被转换为普通连接。

# 回到 Q1

作为说明性示例，让我们重新考虑查询Q1的代数转换。它使用依赖连接来计算嵌套子查询，然后使用生成的属性m来检查过滤条件, 论文中有很详细的图, 我这里为都用字符画代替

## Original Query

```
                σ[e.grade=m]
                     |
                     ⧑
                   /   \
         ⋈[s.id=e.sid]   Γ[∅;m:min(e2.grade)]
        /            \           |
students s       exams e    σ[s.id=e2.sid]
                                |
                            exams e2
```

再回到一开始的 Q1, 我们重新按照通用解嵌套的算法走一遍.

## Step 1 : 连接下推

应用依赖连接下推规则：

$$

D ⧑ (T_1 \bowtie_p T_2) \equiv (D ⧑ T_1) \bowtie_p T_2 \text{ if } \mathcal{F}(T_2) \cap \mathcal{A}(D) = \emptyset

$$
其中：
- $D := \Pi_{d.id:s.id}(students \bowtie_{s.id=e.sid} exams)$
	- 记得$D$的作用是携带外部查询的上下文信息到内部查询
		- 通过重命名 `s.id` 为 `d.id`，我们实际上创建了一个新的命名空间
		- 原来依赖于 s.id 的地方都被替换成了 d.id, 保留外层连接条件 `s.id=d.id`
- 依赖连接被下推到右分支的 $\Gamma$ 操作

转换后得到：
```
                 σ[e.grade=m]
                      |
                 ⋈[s.id=d.id]
               /             \
             /                ⧑
           /               /      \
         |        Π[d.id:s.id]  Γ[∅;m:min(e2.grade)]
         |           /                     |
         |         /                 σ[d.id=e2.sid]
         |       /                         |
   ⋈[s.id=e.sid]                     exams e2
   /          \
students s  exams e
```

`Π[d.id:s.id]` 到 `⋈[s.id=e.sid]` 的线, 在优化过程中会一直存在，直到我们完全消除了依赖关系, 这根线就是依赖连接（⧑）存在的一个视觉表现
## Step 2 : 分组下推

应用了分组操作的下推规则：

$$

D ⧑ (\Gamma_{A;a:f}(T)) \equiv \Gamma_{A \cup \mathcal{A}(D);a:f}(D ⧑ T)

$$

这一步将依赖连接下推到了分组操作下面，同时在分组属性中保留了 d.id

因为在第三步之后, 就已经具备了解开依赖 join 的基础, 所以在这里也对比一下, 为什么在这一步做不了, 记得规则是 $D ⧑ T \equiv D \bowtie T \text{ if } \mathcal{F}(T) \cap \mathcal{A}(D) = \emptyset$
- `D := Πd.id:s.id`
- `T := σs.id=e2.sid`
- 检查 $T$ 中的自由变量集合 $\mathcal{F}(T)$ 与 $D$ 中的属性集合 $\mathcal{A}(D)$ 的交集
	- $\mathcal{F}(T)$ 包含`s.id`(在选择条件中)
	- $\mathcal{A}(D)$ 包含`d.id`(重命名自 `s.id` )
	- 此时选择条件中还在使用原始的 `s.id`，说明右子树仍然依赖于外部查询的变量

所以不能直接解开依赖连接, 需要继续下推, 转换后得到
```
                σ[e.grade=m]
                     |
                ⋈[s.id=d.id]
               /            \
             /         Γ[d.id;m:min(e2.grade)]
           /                    |
         |                      ⧑
         |                    /   \
         |        — Π[d.id:s.id]    σ[d.id=e2.sid]
         |       /                        |
   ⋈[s.id=e.sid]                      exams e2
   /          \     
students s  exams e
```

## Step 3 : 谓词上移

继续应用选择操作
$$

D ⧑ \sigma_p(T_2) \equiv \sigma_p(D ⧑ T_2)

$$
在应用之后, 选择操作被上移到了依赖连接之上, 这样，现在的 D 中只包含了 d.id (重命名自s.id)，而右侧子树中所有涉及 `s.id` 的地方都被替换为了 `d.id` 。这个转换为下一步消除依赖连接创造了条件，因为现在右子树中不再有任何依赖于外部查询的属性了, 转换后得到
```
                σ[e.grade=m]
                     |
                ⋈[s.id=d.id]
               /            \
             /              Γ[d.id;m:min(e2.grade)]
           /                      |
         /                  σ[d.id=e2.sid]
        |                         |
        |                         ⧑
        |                       /    \
        |          — Π[d.id:s.id]   exams e2
        |        /   
  ⋈[s.id=e.sid]
  /            \ 
students s    exams e    
```

## Step 4 : 解除

继续运用规则
$$
D ⧑ T \equiv D \bowtie T \text{ if } \mathcal{F}(T) \cap \mathcal{A}(D) = \emptyset
$$
分析一下
- 此时的 `D := Πd.id:s.id`
- $T$ 是右子树 `exams e2`
- 检查 $T$ 中的自由变量集合 $\mathcal{F}(T)$ 与 $D$ 中的属性集合 $\mathcal{A}(D)$ 的交集
	- $\mathcal{F}(T)$ 只包含 `e2.sid`
	- $\mathcal{A}(D)$ 只包含 `d.id`
	- 交集为空
		- 不是在说值的层面(确实,  `d.id` 和 `e2.sid` 的值是一样的)
		- 而是在说命名空间的层面
			- 右子树中已经没有任何引用外部查询的变量了 (没有 `s.id` ，全都变成了 `d.id` )
	- 这就像是
		- 原来是："对每个学生s，找到他的考试记录e2"(依赖)
		- 现在是："对这组ID(`d.id`)，找到对应的考试记录e2" (独立)
	- 虽然值是一样的，但通过重命名解除了语法层面的依赖关系

因此可以将依赖连接转换为普通连接, 转换后得到
```
               σ[e.grade=m]
                    |
               ⋈[s.id=d.id]
              /            \
            /          Γ[d.id;m:min(e2.grade)]
          /                      |
         |                 σ[d.id=e2.sid]
         |                       |
         |                       ⨝
         |                     /    \
         |         — Π[d.id:s.id]  exams e2
         |       /   
  ⋈[s.id=e.sid]
  /            \ 
students s    exams e
```

## Step 5 : pushing selections back down

后续的过程就和依赖连接没什么关系了, 这一步是选择下推, 再把它给推下去, 尽早过滤数据，减少中间结果集的大小
```
                  σ[e.grade=m]
                       |
                  ⋈[s.id=d.id]
                 /            \
               /              Γ[d.id;m:min(e2.grade)]
             /                        |
           |                    ⋈[d.id=e2.sid]
           |                    /          \
           |       —— Π[d.id:s.id]         exams e2
           |      /          
     ⋈[s.id=e.sid]      
    /          \ 
students s   exams e
```

## Step 6 : finish!

![[理论/数据库系统/Bottom Up/1-Plan/Optimizer/Unnesting Arbitrary Queries#^4c262f|Unnesting Arbitrary Queries]]

现在要准备把这根线给移除掉了
```
               σ[e.grade=m]
                    |
               ⋈[s.id=d.id]
              /            \
            /              Γ[d.id;m:min(e2.grade)]
          /                       |
         |                  σ[d.id=e2.sid]
         |                        |
         |                  χ[d.id:e2.sid]
         |                        |
   ⋈[s.id=e.sid]              exams e2
   /          \
students s  exams e
```

这里的关键变化是：
1. 那根表示依赖关系的斜线消失了
2. 引入了 χ 操作(重命名)
3. 左右两边完全解耦
4. 所有的 `s.id` 引用都被替换为了 `d.id`

把这个关系代数树写回 SQL, 可以写成类似
```sql
SELECT s.name, e.course
FROM students s 
JOIN exams e ON s.id = e.sid
JOIN (
    SELECT sid, MIN(grade) as min_grade
    FROM exams
    GROUP BY sid
) min_grades ON s.id = min_grades.sid
WHERE e.grade = min_grades.min_grade;
```

或者用 CTE(Common Table Expression) 的方式
```sql
WITH min_grades AS (
    SELECT sid, MIN(grade) as min_grade
    FROM exams
    GROUP BY sid
)
SELECT s.name, e.course
FROM students s 
JOIN exams e ON s.id = e.sid
JOIN min_grades ON s.id = min_grades.sid
WHERE e.grade = min_grades.min_grade;
```

## 小结

在许多情况下，在这个例子中也是如此，完全可以消除与 $D$ 的join

如果域的所有属性都与现有属性进行 (等值) 连接, 那其实都没有必要生成 `join` , 在本例中, join后有 `d.id=e2.sid` ，那么就可以用一个映射操作符来替代，将 `d.id` 替换为 `e2.sid` 。这种替代的好处是查询的各个部分变得完全独立，不再有原始嵌套的痕迹

虽然这种替代在逻辑上使查询结构更简单，但它也会产生一定的副作用。替代操作会创建原始元组的超集，一般情况下，去除与域的连接并进行替代可能会导致中间结果集更大。这是因为连接原本具有的筛选效果被移除了，因此，是否移除与域的连接需要由查询优化器根据具体情况来决定，它需要权衡简化查询结构带来的好处和中间结果集增大导致的性能开销。

最后, 在尝试在替代操作后移除 `σ[d.id=e2.sid]` 时需要格外小心。由于 `d.id` 是从 `e2.sid` 推导而来，可能会想直接丢弃这个选择操作，但这仅在 `e2.sid` 不可为空的情况下才是安全的。如果可能取空值，那么这个选择操作必须保留，实际上它就相当于一个非空检查操作。这强调了在进行查询优化操作时，需要充分考虑数据的完整性约束 (如是否可为空), 以确保查询结果的正确性。

# Q2 : 更复杂的例子

对于更复杂的 Query Q2
```sql
SELECT
    s.name,
    e.course
FROM
    students s,
    exams e
WHERE
    s.id = e.sid
    AND (
        s.major = 'CS'
        OR s.major = 'Games Eng'
    )
    AND e.grade >= (
        SELECT
            AVG(e2.grade) + 1
        FROM
            exams e2
        WHERE
            s.id = e2.sid
            OR (
                e2.curriculum = s.major
                AND s.year > e2.date
            )
    );
```

这个查询的目的是找出计算机科学（CS）或游戏工程（Games Eng）专业的学生，他们在考试中的成绩至少比他们自己或比他们年长的学生的平均成绩低一个等级。

使用论文的解耦策略
- 就是要用投影 `Π[d.id:s.id, d.year:s.year, d.major:s.major]` 捕获所有需要的外部属性
- 这些属性通过那根斜线传递给右子树
- 在右子树中用 `d.` 替代所有 `s.` 的引用

然后一通操作吧啦吧啦...
```
-- origin
σ[s.major='CS' OR s.major='Games Eng'](students s) ⋈[s.id=e.sid] exams e ⧑ 
    Γ∅; m: avg(e2.grade)+1 (σ[s.id=e2.sid OR (e2.curriculum=s.major AND s.year>e2.date)] exams e2)

-- step 1: 应用依赖连接下推规则
σ[s.major='CS' OR s.major='Games Eng'](students s) ⋈[s.id=e.sid] exams e ⋈[s.id=d.id]
    (Π[d.id:s.id, d.year:s.year, d.major:s.major] ⧑ 
        Γ[d.id,d.year,d.major]; m: avg(e2.grade)+1 
            (σ[d.id=e2.sid OR (e2.curriculum=d.major AND d.year>e2.date)] exams e2))

-- step 2: 选择和投影下推
σ[e.grade>=m] (
    (σ[s.major='CS' OR s.major='Games Eng'](students s) ⋈[s.id=e.sid] exams e) ⋈[s.id=d.id]
        (Π[d.id,d.year,d.major] ⧑ 
            Γ[d.id,d.year,d.major]; m: avg(e2.grade)+1 
                (σ[d.id=e2.sid OR (e2.curriculum=d.major AND d.year>e2.date)] exams e2))
)
```

化简到后面, 你会得到在论文中用 `Figure 9` 描述的图
```
                                          Π[s.name,e.course]
                                          |
                     ⋈[e.grade>m+1 ∧ (d.id=s.id ∨ (d.year>e.date ∧ e.curriculum=d.major))]
                       /                                    \
                     /                    Γ[d.id,d.year,d.major;m:avg(e2.grade)]
                     |                                      |
                     |                 ⋈[d.id=e2.sid ∨ (d.year>e2.date ∧ e2.curriculum=d.major)]
                     |                                /                \
                     |              Π[d.id:s.id,    /                exams e2
                     |                d.year:s.year,
                     |                d.major:s.major]
                     |             /
              ⋈[s.id=e.sid] _ _ _/
              /            \ 
σ[s.major='CS' ∨          exams e
s.major='Games Eng']
       |
   students s
```

关系代数树的解释
- 左分支：students 和 exams 的连接，带有专业过滤条件
- 右分支：计算每个学生的平均分（考虑同专业且更早的考试）
- 顶层连接：将学生成绩与计算出的阈值比较

对比 `Q1` 这种只有 `s.id = e2.sid` 这样的简单等值连接, 可以通过重命名 s.id 为 d.id 完全消除依赖. 但是在 `Q2` , 连接谓词包含了 `d.id = e2.sid OR (d.year > e2.date AND e2.curriculum = d.major)`
这个谓词有两个特点
- 包含了OR条件
- 包含了不等值比较（>）

即使通过 `Π[d.id:s.id, d.year:s.year, d.major:s.major]` 捕获所有的需要的属性, 但
- OR条件使得我们无法将其分解为多个简单连接
- 不等值比较阻止了进一步的优化

所以需要将外部查询的域信息以侧向传递 (sideways transfer) 的方式, 传递给嵌套查询的评估过程。

尽管存在上述限制, 但 Q2 的优化仍取得了一定成果。所有的依赖连接 (dependent joins) 都已消失, 并且被高效的常规代数运算符所取代。这意味着虽然无法完全消除 $D$ 以及解耦嵌套子查询评估, 但通过优化, 将原本复杂的依赖连接转换为数据库系统能够更高效处理的常规代数运算形式, 从而在一定程度上提升了查询执行的效率。

