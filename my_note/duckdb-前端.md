---
tags:
  - db/example/duck
---
这篇笔记只关注[[理论/数据库系统/Bottom Up/0-Frontend/数据库前端职责概述|前端的职责]], 侧重关注实践, 快速地对代码进行走读, 其中对于 Parser 和 Binder 相关的逻辑会相对简略, 在这之前我们先回忆一下 Andy 课上关于数据库架构的常规组织
![[理论/数据库系统/Bottom Up/1-Plan/img/Architecture Overview.png]]
简单来说就是下面的流程
```
SQL Query 
-> Parser (语法分析)
-> Binder (语义分析) 
-> Planner (生成逻辑计划)
-> Optimizer (优化逻辑计划)
-> Physical Planner (生成物理计划)
```

# before we start

但是在进入上述这两个模块之前, 我们先整体地串一下, 看一下上面的流程在 duck 中是如何对应的

其中一个入口 [ClientContext::Query](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/main/client_context.cpp#L948) 很清晰的表明了这几个阶段的内容
```c++ title:src/main/client_context.cpp
unique_ptr<QueryResult> ClientContext::Query(const string &query, bool allow_stream_result) {
    auto lock = LockContext();
    
    // 1. 解析SQL语句
    ErrorData error;
    vector<unique_ptr<SQLStatement>> statements;
    if (!ParseStatements(*lock, query, statements, error)) {
        return ErrorResult<MaterializedQueryResult>(std::move(error), query);
    }
    
    // 2. 遍历每个语句进行处理
    for (idx_t i = 0; i < statements.size(); i++) {
        auto &statement = statements[i];
        bool is_last_statement = i + 1 == statements.size();
        
        // 3. 创建PendingQuery
        PendingQueryParameters parameters;
        parameters.allow_stream_result = allow_stream_result && is_last_statement;
        auto pending_query = PendingQueryInternal(*lock, std::move(statement), parameters);
        
        // 4. 执行查询
        unique_ptr<QueryResult> current_result;
        if (pending_query->HasError()) {
            current_result = ErrorResult<MaterializedQueryResult>(pending_query->GetErrorObject());
        } else {
            current_result = ExecutePendingQueryInternal(*lock, *pending_query);
        }
        //...
    }
}
```

- [ParseStatements](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/main/client_context.cpp#L1007) 负责解析
	- 调用ParseStatementsInternal, 其会初始化一个 [Parser](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/parser/parser.hpp#L30) 并开始具体的转换流程
- [PendingQueryInternal](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/main/client_context.cpp#L1074) 准备执行
	- 调用的 [PendingStatementOrPreparedStatement](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/main/client_context.cpp#L867) 会
		- 检查一些事务的内容, 比如说是否要进行 autoCommit 之类的
		- 检查是否要启动 `profiler`
		- 根据 SQL 是否是 Prepare 语句做不同的情况处理
			- 对于普通语句, 进入[PendingStatementInternal](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/main/client_context.cpp#L764)
				- 最终会调用到 [CreatePreparedStatementInternal](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/main/client_context.cpp#L350), 进行逻辑计划, 优化器, 物理计划的生成
	```c++ title:src/main/client_context.cpp
shared_ptr<PreparedStatementData>
 ClientContext::CreatePreparedStatementInternal(ClientContextLock &lock, const string &query,
                                                unique_ptr<SQLStatement> statement,
                                                optional_ptr<case_insensitive_map_t<BoundParameterData>> values) {
 	StatementType statement_type = statement->type;
 	auto result = make_shared_ptr<PreparedStatementData>(statement_type);
	...
 	Planner planner(*this); 
 	planner.CreatePlan(std::move(statement));
 
 	auto plan = std::move(planner.plan);
 	// extract the result column names from the plan
	...
 	// now convert logical query plan into a physical query plan
 	PhysicalPlanGenerator physical_planner(*this);
 	auto physical_plan = physical_planner.CreatePlan(std::move(plan));
 	profiler.EndPhase();
 
 #ifdef DEBUG
 	D_ASSERT(!physical_plan->ToString().empty());
 #endif
 	result->plan = std::move(physical_plan);
 	return result;
 }
	```
- [ExecutePendingQueryInternal](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/main/client_context.cpp#L1087) 执行查询
	- 这个部分后面再讨论

## 调用链路青春版

所以一个简练版本的调用链路是这样的, 我们通过这个初步感知一下理论和实践之间的不同

```
ClientContext::Query
    -> ParseStatements  // Parser阶段
        -> Parser::ParseQuery
    -> PendingQueryInternal
        -> PendingStatementOrPreparedStatementInternal
            -> CreatePreparedStatement
                -> Planner::CreatePlan
                    -> Binder::Bind  // Binder阶段
                        -> BindStatement
                            -> BindSelect/BindInsert等
                    -> Planner根据BoundStatement生成逻辑计划
    					-> CreatePlan(BoundStatement)
        					-> 根据语句类型生成不同的LogicalOperator
                -> Optimizer::Optimize  // 优化阶段
                -> PhysicalPlanGenerator::CreatePlan  // 物理计划生成
```

^70af6f

# Parser 阶段

逻辑上要做的事情就是[[理论/数据库系统/Bottom Up/0-Frontend/数据库前端职责概述#1. 查询解析与验证（Parse & Validate）|把SQL变成抽象语法树]], 具体代码的起点是 [Parser::ParseQuery](https://github.com/duckdb/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder.cpp#L146), DuckDB选择复用PostgreSQL的解析器, 然后用 [Transformer](https://github.com/jensenojs/duckdb/blob/170607dbce9c0bcddd2c08094ce772730f8c7a45/src/include/duckdb/parser/transformer.hpp#L46) 将 PostgreSQL 的解析树转换为 DuckDB 的语法树, 大体上的完整流程是这样的

## 主要流程

- Unicode 等字符的处理
```c++ title:"src/parser/transformer.cpp"
// 检查并处理Unicode空格
string new_query;
if (StripUnicodeSpaces(query, new_query)) {
    // 如果有Unicode空格，用处理后的查询重新解析
    ParseQuery(new_query);
    return;
}
```

- 主要解析流程, 其中核心是 [TransformParseTree](https://github.com/jensenojs/duckdb/blob/ec3870cd974aca1ba9657b00b64e38f9db63ea9d/src/parser/transformer.cpp#L28), 之后会展开论述
```c++ title:"src/parser/transformer.cpp"
PostgresParser::SetPreserveIdentifierCase(options.preserve_identifier_case);
bool parsing_succeed = false;
{
    PostgresParser parser;
    parser.Parse(query);
    if (parser.success) {
        if (!parser.parse_tree) {
            // 空语句，直接返回
            return;
        }
        // 成功解析：转换PostgreSQL语法树到DuckDB语法树
        transformer.TransformParseTree(parser.parse_tree, statements);
        parsing_succeed = true;
    } else {
        // 记录错误信息
        parser_error = parser.error_message;
        if (parser.error_location > 0) {
            parser_error_location = NumericCast<idx_t>(parser.error_location - 1);
        }
    }
}
```

- 如果解析失败, 尝试按照分号分割语句, 并且使用扩展解析器尝试解析
```c++ title:"src/parser/transformer.cpp"
// 如果DuckDB解析失败，尝试按分号分割语句
if (!parsing_succeed) {
    if (!options.extensions || options.extensions->empty()) {
        // 没有扩展解析器，直接抛出错误
        throw ParserException::SyntaxError(query, parser_error, parser_error_location);
    } else {
        // 分割SQL语句并尝试用扩展解析器解析
        auto query_statements = SplitQueryStringIntoStatements(query);
        for (auto const &query_statement : query_statements) {
            // 1. 先尝试DuckDB解析器
            // 2. 如果失败，尝试每个扩展解析器
            // 3. 如果所有解析器都失败，抛出错误
        }
    }
}
```

- 最终处理
```c++ title:"src/parser/transformer.cpp"
if (!statements.empty()) {
    // 设置最后一个语句的长度
    auto &last_statement = statements.back();
    last_statement->stmt_length = query.size() - last_statement->stmt_location;
    
    // 为每个语句设置原始查询文本
    for (auto &statement : statements) {
        statement->query = query;
        // 特殊处理CREATE语句
        if (statement->type == StatementType::CREATE_STATEMENT) {
            auto &create = statement->Cast<CreateStatement>();
            create.info->sql = query.substr(statement->stmt_location, statement->stmt_length);
        }
    }
}
```

至此, Parse 的主要流程就已经分析完了
## Transformer

[TransformParseTree](https://github.com/duckdb/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transformer.cpp#L28) 是从PostgreSQL语法树到DuckDB语法树的主要转换入口,
```c++ title:"src/parser/transformer.cpp"
bool Transformer::TransformParseTree(duckdb_libpgquery::PGList *tree, 
                                   vector<unique_ptr<SQLStatement>> &statements) {
    // 初始化堆栈检查
    InitializeStackCheck();
    
    // 遍历PostgreSQL语法树的每个节点
    for (auto entry = tree->head; entry != nullptr; entry = entry->next) {
        // 清理上一次转换的状态
        Clear();
        
        // 转换单个语句
        auto n = PGPointerCast<duckdb_libpgquery::PGNode>(entry->data.ptr_value);
        auto stmt = TransformStatement(*n);
        
        // 处理PIVOT（如果有）
        if (HasPivotEntries()) {
            stmt = CreatePivotStatement(std::move(stmt));
        }
        
        statements.push_back(std::move(stmt));
    }
    return true;
}
```

### 表达式的派分

从代码中你也能看到 [TransformStatement](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transformer.cpp#L58), 是主要的逻辑, 在做语句的转换分发. 它内部会进入到一个叫 `TransformStatementInternal` 的函数, 它的代码示例大体如下
```c++ title:"src/parser/transformer.cpp"
unique_ptr<SQLStatement> Transformer::TransformStatementInternal(duckdb_libpgquery::PGNode &stmt) {
    switch (stmt.type) {
    case T_PGSelectStmt:
        return TransformSelect(stmt);
    case T_PGCreateStmt:
        return TransformCreateTable(stmt);
    case T_PGCreateTableAsStmt:
        return TransformCreateTableAs(stmt);
    case T_PGPrepareStmt:
        return TransformPrepare(stmt);
    // ... 其他语句类型
    }
}
```

下面的结构大体上展示了DuckDB的转换系统
```shell title:"src/parser/transform/statement"
src/parser/transform/
├── constraint/          
│   ├── transform_constraint.cpp        # 转换表约束(PK, FK等)
│   └── transform_check_constraint.cpp  # 转换CHECK约束
│
├── expression/          
│   ├── transform_function.cpp          # 函数调用转换
│   ├── transform_operator.cpp          # 运算符转换
│   ├── transform_constant.cpp          # 常量转换
│   ├── transform_columnref.cpp         # 列引用转换
│   └── transform_subquery.cpp          # 子查询转换
│
├── helpers/            
│   ├── transform_typename.cpp          # 类型名转换
│   ├── transform_orderby.cpp           # ORDER BY转换
│   └── transform_groupby.cpp           # GROUP BY转换
│
├── statement/          
│   ├── transform_select.cpp            # SELECT语句转换
│   ├── transform_insert.cpp            # INSERT语句转换
│   ├── transform_create_table.cpp      # CREATE TABLE转换
│   ├── transform_create_view.cpp       # CREATE VIEW转换
│   └── transform_delete.cpp            # DELETE语句转换
│
└── tableref/           
    ├── transform_base_tableref.cpp     # 基础表引用转换
    ├── transform_join.cpp              # JOIN转换
    └── transform_subquery.cpp          # 子查询表引用转换
```

### 转换的终点🏁

PG AST 是怎么样的, 这里暂时不去细扣, 这里以一个后面会具体讨论的 SQL 的其中一部份的内容填充为例, 可以按图索骥地找到对应的 yacc 文件
```c++ title:"third_party/libpg_query/src_backend_parser_gram.cpp"
  case 990:
#line 2576 "third_party/libpg_query/grammar/statements/select.y"
    {
					PGSubLink *n = makeNode(PGSubLink);
					n->subLinkType = (yyvsp[(3) - (4)].subquerytype);
					n->subLinkId = 0;
					n->testexpr = (yyvsp[(1) - (4)].node);
					n->operName = (yyvsp[(2) - (4)].list);
					n->subselect = (yyvsp[(4) - (4)].node);
					n->location = (yylsp[(2) - (4)]);
					(yyval.node) = (PGNode *)n;
				;}
    break;
```

不过转换的终点我们需要大体地了解一下.
```c++ title:"src/include/duckdb/parser/sql_statement.hpp"
class SQLStatement {
public:
    StatementType type;  // 语句类型
    idx_t stmt_location; // 语句位置
    idx_t stmt_length;   // 语句长度
    string query;        // 原始查询文本
};
```

所有 duckdb 支持的 SQL 语句类型都要继承这个 [SQLStatement](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/parser/sql_statement.hpp#L20), 以 [SelectStatement](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/parser/statement/select_statement.hpp#L24) 为例：
```c++ : title:src/include/duckdb/parser/statement/select_statement.hpp
class SelectStatement : public SQLStatement {
public:
	static constexpr const StatementType TYPE = StatementType::SELECT_STATEMENT;

public:
	SelectStatement() : SQLStatement(StatementType::SELECT_STATEMENT) {
	}

	// 核心查询节点
`   unique_ptr<QueryNode> node;

protected:
	SelectStatement(const SelectStatement &other);

public:
	//! Convert the SELECT statement to a string

	DUCKDB_API string ToString() const override;
	//! Create a copy of this SelectStatement
	DUCKDB_API unique_ptr<SQLStatement> Copy() const override;
	//! Whether or not the statements are equivalent
	bool Equals(const SQLStatement &other) const;

	// 序列化/反序列化支持
	void Serialize(Serializer &serializer) const;
	static unique_ptr<SelectStatement> Deserialize(Deserializer &deserializer);
};
```

[QueryNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/parser/query_node.hpp#L48) 就是 AST 的节点, 这里不展开

## One example : with 子查询

下面我们以一个具体的例子, 来梳理一下转换的整个流程, 这个例子包含了基础列引用, 相关子查询, 聚合函数等内容. 自顶向下地感受一下整个状态的流转.

```sql
SELECT a, 
       (SELECT MAX(x) FROM t2 WHERE t2.id = t1.id) as sub_max,
       COUNT(*) as cnt
FROM t1
WHERE a > 10
GROUP BY a
HAVING COUNT(*) > 1
ORDER BY a DESC;
```
我们通过它来具体地学习 duckdb 如何处理子查询, 表达式树的构建过程的一些关键转换点, 以及 `StackChecker` 如何防止递归过深

### parse 层

```c++ title:"src/parser/parser.cpp"
bool Parser::ParseQuery(const string &query) {
    PostgresParser parser;
    parser.Parse(query);  // 得到PG的AST
    
    Transformer transformer(options);
    transformer.TransformParseTree(parser.parse_tree, statements);
}
```
### Transform 层

`TransformParseTree` 就是做一个简单的分发, 因此从前面已经讲过的 [TransformStatementInternal](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transformer.cpp#L135) 开始, 第一层是 `selectStatement` 的构造
```c++ title:"src/parser/transformer.cpp"
unique_ptr<SQLStatement> Transformer::TransformStatementInternal(duckdb_libpgquery::PGNode &stmt) {
	switch (stmt.type) {
		...
	    // 识别出SELECT语句
	    case duckdb_libpgquery::T_PGSelectStmt:
	        return TransformSelect(PGCast<duckdb_libpgquery::PGSelectStmt>(stmt));
	    ...
	}
}
```

随后我们进入 [TransformSelectStmt](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/statement/transform_select.cpp#L38), 但它其实也没有干太多具体的实现, 除了[[实践/examples/Database/duckdb/duckdb-前端#移动语义|移动语义]]以外, 就是进一步使唤 [TransformSelectNodeInternal](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/statement/transform_select.cpp#L18) 帮它做事情, 这里的 `is_last` 是为了区分独立的SELECT语句，还是其他语句（如CREATE TABLE AS）中的SELECT子句。
```c++ title:"src/parser/transform/statement/transform_select.cpp"
unique_ptr<QueryNode> Transformer::TransformSelectNodeInternal(duckdb_libpgquery::PGSelectStmt &select,
                                                               bool is_select) {
	// Both Insert/Create Table As uses this.
	if (is_select) {
		if (select.intoClause) {
			throw ParserException("SELECT INTO not supported!");
		}
		if (select.lockingClause) {
			throw ParserException("SELECT locking clause is not supported!");
		}
	}
	unique_ptr<QueryNode> stmt = nullptr;
	if (select.pivot) {
		stmt = TransformPivotStatement(select);
	} else {
`	   stmt = TransformSelectInternal(select);
	}
	return TransformMaterializedCTE(std::move(stmt));
}
```

### QUeryNode 层

只有在进入了关键的 [TransformSelectInternal](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/statement/transform_select_node.cpp#L45) 之后, 我们才能看到 `select` 的转换到底是在哪里发生的, 这里保留了这个函数的主要枝干, 删去了部分无关的代码. 可以看到, 每个分支的末端无非是用不同的成员字段作为参数传入表达式的计算. 

```c++ title:"src/parser/transform/statement/transform_select_node.cpp"
unique_ptr<QueryNode> Transformer::TransformSelectInternal(duckdb_libpgquery::PGSelectStmt &stmt) {
	D_ASSERT(stmt.type == duckdb_libpgquery::T_PGSelectStmt);
	auto stack_checker = StackCheck();

	unique_ptr<QueryNode> node;

	switch (stmt.op) {
	case duckdb_libpgquery::PG_SETOP_NONE: {
		node = make_uniq<SelectNode>();
		auto &result = node->Cast<SelectNode>();
		// 转换SELECT列表
		
		...
		
		// 先处理DISTINCT子句
		if (stmt.distinctClause != nullptr) {
			...
		}
		
		// do this early so the value lists also have a `FROM`
		if (stmt.valuesLists) {
			// VALUES list, create an ExpressionList
			D_ASSERT(!stmt.fromClause);
			result.from_table = TransformValuesList(stmt.valuesLists);
			result.select_list.push_back(make_uniq<StarExpression>());
		} else {
			// 这里会处理：
			// - 简单列引用 'a'
			// - 子查询 '(SELECT MAX(x) FROM t2...)'
			// - 聚合函数 'COUNT(*)'
			if (!stmt.targetList) {
				throw ParserException("SELECT clause without selection list");
			}
			// transform in the specified order to ensure positional parameters are correctly set
			if (stmt.from_first) {
				result.from_table = TransformFrom(stmt.fromClause);
				TransformExpressionList(*stmt.targetList, result.select_list);
			} else {
				TransformExpressionList(*stmt.targetList, result.select_list);
` 			   result.from_table = TransformFrom(stmt.fromClause);
			}
		}

		// where
`  	   result.where_clause = TransformExpression(stmt.whereClause);
		// group by
`       TransformGroupBy(stmt.groupClause, result);
		// having
`       result.having = TransformExpression(stmt.havingClause);
		// qualify
		result.qualify = TransformExpression(stmt.qualifyClause);
		// sample
		result.sample = TransformSampleOptions(stmt.sampleOptions);
		break;
	}
	case duckdb_libpgquery::PG_SETOP_UNION:
	...
	case duckdb_libpgquery::PG_SETOP_UNION_BY_NAME: {
		...
	}
	default:
		throw NotImplementedException("Statement type %d not implemented!", stmt.op);
	}

	TransformModifiers(stmt, *node);

	return node;
}
```

可以看到很多地方都调用了[TransformExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/expression/transform_expression.cpp#L86)和[TransformExpressionList](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/expression/transform_expression.cpp#L93), 不过在进入表达式转换的入口之前, 再仔细看一看这一部分的代码, 也没什么别的原因, 就是强调这一部分和后面的子查询是相关的.
```c++ title:""
		// do this early so the value lists also have a `FROM`
		if (stmt.valuesLists) {
			...
		} else {
			// 这里会处理：
			// - 简单列引用 'a'
			// - 子查询 '(SELECT MAX(x) FROM t2...)'
			// - 聚合函数 'COUNT(*)'
			if (!stmt.targetList) {
				throw ParserException("SELECT clause without selection list");
			}
			// transform in the specified order to ensure positional parameters are correctly set
			if (stmt.from_first) {
				result.from_table = TransformFrom(stmt.fromClause);
				TransformExpressionList(*stmt.targetList, result.select_list);
			} else {
				TransformExpressionList(*stmt.targetList, result.select_list);
` 			   result.from_table = TransformFrom(stmt.fromClause);
			}
		}
```

首先是from_first 是在 PostgreSQL 的语法解析阶段设置的，用于标识 SQL 语句中 FROM 和 SELECT 的顺序。PostgreSQL 支持类似 `FROM table SELECT column (from_first = true)` 的写法, 这里是一些不太要紧的小细节, DuckDB通过from_first标志来支持这两种语法, 最终生成的执行计划是一样的，只是语法解析的顺序不同
``` title:"third_party/libpg_query/grammar/grammar_out.output"
simple_select: SELECT opt_all_clause opt_target_list_opt_comma into_clause from_clause where_clause group_clause having_clause window_clause qualify_clause sample_clause
             | SELECT distinct_clause target_list_opt_comma into_clause from_clause where_clause group_clause having_clause window_clause qualify_clause sample_clause
             | FROM from_list opt_select into_clause where_clause group_clause having_clause window_clause qualify_clause sample_clause
             | FROM from_list SELECT distinct_clause target_list_opt_comma into_clause where_clause group_clause having_clause window_clause qualify_clause sample_clause
```
> 这里更快的验证方式其实是看 [PGSelectStmt](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/third_party/libpg_query/include/nodes/parsenodes.hpp#L1291) 行的 `from_list` 字段的注释, 我这里保险起见多翻了一步

### Expression 层

[TransformExpressionList](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/expression/transform_expression.cpp#L93) 只是一个循环调用 [TransformExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/expression/transform_expression.cpp#L86) 的封装, 而后者会处理表达式转换与子查询, 针对本例的情况, 它差不多会走到下面几种 case, 其中关于列引用和运算符表达式这里就不展开了.
```c++ title:"src/parser/transform/expression/transform_expression.cpp"
unique_ptr<ParsedExpression> Transformer::TransformExpression(duckdb_libpgquery::PGNode &node) {
    auto stack_checker = StackCheck();  // 检查递归深度
    
    switch (node.type) {
    case duckdb_libpgquery::T_PGColumnRef:  // 列引用，如 "a"
        return TransformColumnRef(PGCast<duckdb_libpgquery::PGColumnRef>(node));
        
    case duckdb_libpgquery::T_PGSubLink:    // 子查询
        return TransformSubquery(PGCast<duckdb_libpgquery::PGSubLink>(node));
        
    case duckdb_libpgquery::T_PGFuncCall:   // 函数调用，如 "COUNT(*)"
        return TransformFuncCall(PGCast<duckdb_libpgquery::PGFuncCall>(node));
        
    case duckdb_libpgquery::T_PGAExpr:      // 运算符表达式，如 "a > 10"
        return TransformAExpr(PGCast<duckdb_libpgquery::PGAExpr>(node));
    }
}
```

对于 SQL 中的 `COUNT(*)` , 最终会进入到 [TransformFuncCall](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/expression/transform_function.cpp#L134) 中, 其核心流程大体上如下所示, 实际情况会复杂一些, 毕竟对于函数的参数可能也会有没完没了的 [TransformExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/expression/transform_expression.cpp#L86), 也就是又要回到上面去. 
```c++ title:"src/parser/transform/expression/transform_function.cpp"
unique_ptr<ParsedExpression> Transformer::TransformFuncCall(duckdb_libpgquery::PGFuncCall &node) {
    auto function = make_uniq<FunctionExpression>();
    
    // 设置函数名
    function->function_name = node.funcname->head->data.ptr_value;
    
    // 转换函数参数
    for (auto arg = node.args->head; arg; arg = arg->next) {
        function->children.push_back(TransformExpression(arg->data.ptr_value));
    }
    
    // 处理特殊函数，如聚合函数
    if (node.agg_star) {
        // COUNT(*)的情况
        function->children.push_back(make_uniq<StarExpression>());
    }
    
    return function;
}
```

其中, [TransformSubquery](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/transform/expression/transform_subquery.cpp#L24) 我们多关注一点, 因为它本身值得说道的东西多, 还涉及父子 `Transformer` 的状态管理

结合 [SubqueryType](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/common/enums/subquery_type.hpp#L18) 的信息可以看到, DuckDB对子查询的分类有这么几种
```c++ title:src/include/duckdb/common/enums/subquery_type.hpp
enum class SubqueryType : uint8_t {
	INVALID = 0,
	SCALAR = 1,     // Regular scalar subquery
	EXISTS = 2,     // EXISTS (SELECT...)
	NOT_EXISTS = 3, // NOT EXISTS(SELECT...)
	ANY = 4,        // x = ANY(SELECT...) OR x IN (SELECT...)
};
```

理论情况比较复杂, 有哪些分类, 以及为什么要这么进行分类, 可以参阅[[理论/数据库系统/Bottom Up/1-Plan/Optimizer/子查询|子查询的理论部分]] 反正有一些具体的好处. 实际情况是, DuckDB 将 IN、ANY、ALL 都统一为 ANY 类型处理, 实现得比较简单

这里的子查询 `(SELECT MAX (x) FROM t2 WHERE t2. id = t1. id)` 是一个标量子查询, 所以它会走 `PG_SCALAR_SUBLINK` 分支，设置 `SubqueryType::SCALAR`
```c++ title:"src/parser/transform/expression/transform_subquery.cpp"
unique_ptr<ParsedExpression> Transformer::TransformSubquery(duckdb_libpgquery::PGSubLink &node) {
    auto subquery = make_uniq<SubqueryExpression>();
    
    // 创建子Transformer处理子查询
    Transformer child_transformer(*this);
    
    ...
    
	switch (node.subLinkType) {
	case ...
	case duckdb_libpgquery::PG_EXISTS_SUBLINK:
	    // EXISTS (SELECT * FROM t2)
	    subquery->subquery_type = SubqueryType::EXISTS;
	    break;
	    
	case duckdb_libpgquery::PG_SCALAR_SUBLINK:
	    // 标量子查询：返回单个值
	    // SELECT (SELECT max(x) FROM t2) ...
	    subquery->subquery_type = SubqueryType::SCALAR;
	    break;
	    
	case duckdb_libpgquery::PG_ANY_SUBLINK:
	    // 比较子查询：= ANY, > ANY, < ANY等
	    // WHERE x = ANY (SELECT id FROM t2)
	    subquery->subquery_type = SubqueryType::ANY;
	    subquery->child = TransformExpression(node.testexpr);
	    break;
	 ...
	
	}
    
    // 递归转换子查询
    Transformer child_transformer(*this);
	subquery->subquery = child_transformer.TransformSelect(node.subselect);
    
    return subquery;
}
```
### Transformer 状态 : 父子
我们并没有一开始就从[[实践/1/系统编程#阴：实现|数据结构]]的角度去解构代码, 而是从需求出发, 了解了 `Transformer` 的一些成员存在的必要性.
- 每个子查询创建新的Transformer实例
- 子Transformer继承父Transformer的状态和选项
- 相关子查询通过父子Transformer的关系维护外部引用
```c++ title:""
class Transformer {
    Transformer *parent;  // 指向父Transformer
    ParserOptions &options;  // 共享解析选项
    idx_t stack_depth;   // 递归深度控制
    
    // 构造函数：创建子Transformer
    Transformer(Transformer &parent) 
        : parent(&parent), options(parent.options), 
          stack_depth(DConstants::INVALID_INDEX) {
    }
};
```

## 小结

所以上面的转换链路, 大体上如下所示
```
Parser::ParseQuery
    → Transformer::TransformParseTree
        → TransformSelectStmt
            → TransformSelectNodeInternal
                → TransformExpressionList (SELECT列表)
                    → TransformColumnRef ('a')
                    → TransformSubquery (标量子查询)
                    → TransformFuncCall ('COUNT(*)')
                → TransformFrom ('t1')
                → TransformExpression (WHERE子句)
                → TransformGroupBy
                → TransformExpression (HAVING子句)
                → TransformModifiers (ORDER BY)
```

这里再简单地总结一下其他值得我学习的地方
### StackChecker

递归控制：使用 StackChecker 防止过深递归

- 堆栈深度控制：
	- 使用RAII模式自动管理堆栈深度
	- 所有Transformer共享根Transformer的stack_depth
	- 防止恶意SQL导致无限递归

```c++ title:"src/include/duckdb/parser/transformer.hpp"
class Transformer {
private:
    idx_t stack_depth;
    static constexpr const idx_t MAX_EXPRESSION_DEPTH = 1000;

    // 获取根Transformer
    Transformer &RootTransformer() {
        if (parent) {
            return parent->RootTransformer();
        }
        return *this;
    }

public:
    StackChecker<Transformer> StackCheck(idx_t extra_stack = 1) {
        auto &root = RootTransformer();
        // 检查是否超过最大深度
        if (root.stack_depth + extra_stack >= MAX_EXPRESSION_DEPTH) {
            throw ParserException("Max expression depth exceeded");
        }
        return StackChecker<Transformer>(root, extra_stack);
    }
};

// RAII方式管理堆栈深度
template <class T>
class StackChecker {
public:
    StackChecker(T &root, idx_t extra_stack) : root(root) {
        root.stack_depth += extra_stack;
    }
    
    ~StackChecker() {
        root.stack_depth--;
    }
private:
    T &root;
};
```


### 移动语义
那些看上去“什么都没干”的函数实际上是在利用 `unique_ptr` 实现移动语义, 避免深拷贝大型表达式树, 明确所有权的转移
```c++ title:"src/parser/transform/statement/transform_select.cpp"
unique_ptr<SelectStatement> Transformer::TransformSelectStmt(duckdb_libpgquery::PGSelectStmt &select) {
    // 创建新的SelectStatement
    auto result = make_uniq<SelectStatement>();
    
    // TransformSelectNodeInternal返回unique_ptr<QueryNode>
    // 这里发生了所有权转移
    result->node = TransformSelectNodeInternal(select, true);
    
    // 返回时，result的所有权被移动给调用者
    return result;  // 编译器自动使用移动语义
}
```

再明确一下调用链中的所有权转移
```
Parser::ParseQuery
    → auto statements = make_uniq<SQLStatement>();     // 创建
    → statements.push_back(TransformSelectStmt())      // 移动到vector
        → auto result = make_uniq<SelectStatement>()   // 创建
        → result->node = TransformSelectNodeInternal() // 移动赋值
        → return result                                // 移动返回
```

### 引用和指针的显式区分

这是C++中一种常见的防御性编程实践。
```c++
// 引用参数：直接操作原对象
void TransformExpressionList(duckdb_libpgquery::PGList &list,
                           vector<unique_ptr<ParsedExpression>> &result);

// 指针参数：可能为空
unique_ptr<ParsedExpression> TransformExpression(duckdb_libpgquery::PGNode *node);
```

```c++
// 引用参数：直接操作原对象
void TransformExpressionList(duckdb_libpgquery::PGList &list,
                           vector<unique_ptr<ParsedExpression>> &result);

// 指针参数：可能为空
unique_ptr<ParsedExpression> TransformExpression(duckdb_libpgquery::PGNode *node);
```
- 引用(&)：必须有值，不能为null
	- list是输入参数，我们确定它一定有值
	- result 是输出参数，用来收集转换结果
- 指针(\*)：可以为null，需要检查
	- 比如WHERE子句可能不存在：`SELECT * FROM t1`
- `optional_ptr`明确表示可能为空的语义

### 挂一漏万

其他值得关注的细节

1. **错误处理与位置追踪**：
Parser 不仅要解析 SQL，还要提供准确的错误信息和位置。DuckDB 通过以下机制实现：
- 每个表达式都记录 query_location
- 异常包含详细的错误位置信息
- 支持多语言错误消息
```c++
// 错误位置追踪示例

unique_ptr<ParsedExpression> Transformer::TransformExpression(duckdb_libpgquery::PGNode &node) {
	try {
		auto expr = TransformExpressionInternal(node);
		expr->query_location = node.location; // 记录位置信息
		return expr;
	} catch (ParserException &e) {
		e.SetQueryLocation(node.location); // 异常中包含位置
		throw;
	}
}
```
2. **表达式继承体系**：
DuckDB 的表达式系统通过继承实现了强大的类型安全和扩展性：
- BaseExpression 作为所有表达式的基类
- [ParsedExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/parser/parsed_expression.hpp#L30) 提供解析阶段的通用接口
- 具体表达式类型（ColumnRef、Subquery 等）继承自 ParsedExpression
	- 这一部分其实与后面的 [BoundQueryNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/bound_query_node.hpp#L18) 是对应的, 但表达式本身是另外一个大的话题
```c++
class ParsedExpression : public BaseExpression {
	public:
	bool IsAggregate() const override;
	bool IsWindow() const override;
	bool HasSubquery() const override;
	bool IsScalar() const override;
// 提供统一的表达式接口
};
```

3. **扩展解析器机制**：
DuckDB 支持通过扩展机制添加新的语法特性：
- 主解析器失败时会尝试扩展解析器
- 允许添加自定义函数和操作符
- 支持语法规则的动态扩展

这些机制确保了 Parser 的健壮性、可扩展性和用户友好性，但对于理解主流程也许不是必需的, 限于笔者精力有限, 关于 Parser 部分的学习就先暂时到这.

# Binder 阶段

逻辑上要做的事情就是[[理论/数据库系统/Bottom Up/0-Frontend/数据库前端职责概述#2. 查询绑定与参数化 (Bind & Parametrize）|查询的绑定和参数化]], 就是名称解析、类型检查、语义验证之类的东西. 从目录结构来看，Binder的实现分为几个主要部分：
```
src/planner/binder/
	|	   ├── expression/          # 表达式绑定
	|	   ├── query_node/          # 查询节点绑定
	|	   ├── statement/           # SQL语句绑定
	|	   └── tableref/            # 表引用绑定
	├── binder                       # 绑定入口
	├── binding_alias                # .
	└── bind_context                 # 绑定上下文
	
```

DuckDB的 [Binder](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/binder.hpp#L88) 实现架构其实和前面的 `transformer` 有点像的
```c++
class Binder {
    // 上下文
    ClientContext &context;                                  // 客户端上下文
    shared_ptr<Binder> parent;                               // 父Binder(处理子查询)
    
    // 绑定状态
    BindContext bind_context;                                // 维护表和列的映射关系
    vector<CorrelatedColumnInfo> correlated_columns;         // 相关子查询的列
    
    // 绑定模式
    BindingMode mode = BindingMode::STANDARD_BINDING;
    
    // 主要方法
    BoundStatement Bind(SQLStatement &stmt);                 // 入口
    unique_ptr<BoundQueryNode> BindNode(QueryNode &node);    // 绑定查询节点
    unique_ptr<BoundTableRef> Bind(TableRef &ref);           // 绑定表引用
};
```
- 支持嵌套绑定(parent)
	- 表达式绑定器可以切换(active_binders)
	- 每层都有独立的绑定上下文

在 Binder 之后, 就正式地进入了生成逻辑计划的部分, 那就让我们开始吧!

## 主要流程

具体代码的起点是以 `SQLStatement` 为参数的 [Binder::Bind](https://github.com/duckdb/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder.cpp#L146), 是一个大型的 `switch-case` 的递归调用, 根据不同的节点类型做 `Bind` 的分
```c++ title:"src/planner/binder.cpp"
BoundStatement Binder::Bind(SQLStatement &statement) {
	root_statement = &statement;
	switch (statement.type) {
	case StatementType::SELECT_STATEMENT:
		return Bind(statement.Cast<SelectStatement>());
	case StatementType::INSERT_STATEMENT:
		return BindWithCTE(statement.Cast<InsertStatement>());
	case StatementType::COPY_STATEMENT:
		return Bind(statement.Cast<CopyStatement>(), CopyToType::COPY_TO_FILE);
	case StatementType::DELETE_STATEMENT:
		return BindWithCTE(statement.Cast<DeleteStatement>());
	case StatementType::UPDATE_STATEMENT:
		return BindWithCTE(statement.Cast<UpdateStatement>());
	...
}
```

其实没有什么太多好分析的, 所以直接进入例子吧, 我们还是用 Parser 阶段的 SQL 示例，自顶向下地走一遍整体的流程, 看一下会涉及哪些检查. 
```sql
SELECT a,                                     -- 
       (SELECT MAX(x) FROM t2                 -- 子查询作用域
        WHERE t2.id = t1.id) as sub_max,      -- 相关子查询的列引用
       COUNT(*) as cnt                        -- 聚合函数类型检查
FROM t1                                       -- 需要解析a来自哪个表
WHERE a > 10                                  -- 类型兼容性检查
GROUP BY a                                    -- GROUP BY语义验证
HAVING COUNT(*) > 1                           -- 聚合函数语义检查
ORDER BY a DESC;                              -- 排序表达式类型检查
```
## still the example

你会看到就是一系列不同类型的 `Bind` 来不断地进行绑定, 标题做出了一些区分

### 以 SelectStatement 为参数

针对我们的例子而言, 入口的 Binder 分发到以 `SelectStatement &stmt` 为参数的 [Bind](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/statement/bind_select.cpp#L7) , 这里有一些 parser 没有涉及到的细节, 比如说 [StatementProperties](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/binder.hpp#L254), 会包含这个 SQL 的属性, 比如说 `allow_stream_result` 是否能允许结果流式地返还给客户端, 持此之外, 还有类似 `read_databases` 和 `modified_databases` 之类的其他字段
```c++ title:"src/planner/binder/statement/bind_select.cpp"
BoundStatement Binder::Bind(SelectStatement &stmt) {
	auto &properties = GetStatementProperties();
	properties.allow_stream_result = true;
	properties.return_type = StatementReturnType::QUERY_RESULT;
	return Bind(*stmt.node);
}
```

### 以 QueryNode为参数

然后再进入以 `QueryNode` 为参数的 [Bind](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder.cpp#L344), 我们可以直接进入 `SelectNode` 的绑定, 由于这个 SQL 没有 `With` , 因此走的是 [BindNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder.cpp#L321) 的路径.
```c++ title:"src/planner/binder.cpp"
BoundStatement Binder::Bind(QueryNode &node) {
	BoundStatement result;
	if (node.type != QueryNodeType::CTE_NODE && ...) {
		switch (node.type) {
			...
			result = BindWithCTE(node.Cast<SetOperationNode>());
			...
		}
	} else {
`	   auto bound_node = BindNode(node);

		result.names = bound_node->names;
		result.types = bound_node->types;

		// and plan it
		result.plan = CreatePlan(*bound_node);
	}
	return result;
}
```
### 以 SelectNode 为参数

而 [BindNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder.cpp#L321), 也是一个有移动语义的 `switch-case` , 它会进一步调用 `bind_select_node` 的 [BindNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/query_node/bind_select_node.cpp#L366), 不过正如代码里面看到的
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
unique_ptr<BoundQueryNode> Binder::BindNode(SelectNode &statement) {
	D_ASSERT(statement.from_table);

	// first bind the FROM table statement
	auto from = std::move(statement.from_table);
	auto from_table = Bind(*from); // 这里进入表引用的绑定
	return BindSelectNode(statement, std::move(from_table));
}
```

它会先去绑定 `From` , 所以在进入 `BindSelectNode` 之前, 
### From : 以 TableRef 为参数

以 `TableRef` 为参数的 [Bind](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder.cpp#L391) 会根据不同的表类型继续分发. 
```c++ title:"src/planner/binder.cpp"
unique_ptr<BoundTableRef> Binder::Bind(TableRef &ref) {
	unique_ptr<BoundTableRef> result;
	switch (ref.type) {
	case TableReferenceType::BASE_TABLE:
		result = Bind(ref.Cast<BaseTableRef>());
		break;
	case TableReferenceType::JOIN:
		result = Bind(ref.Cast<JoinRef>());
		break;
	,,,
}
```

我们就在这个小节继续看表的绑定是如何做的, 继续往下跟看以 `BaseTableRef` 为参数的 [Bind](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/tableref/bind_basetableref.cpp#L82), 这里省略一些不相关的代码, 留下一些关键的流程, 大体上, 这个过程至少涉及
- 名称解析
	- CTE 名称查找
	- Catalog 中表/视图查找
	- 列名解析
- 类型检查
	- 视图的类型验证
	- 列类型收集
- 语义验证：
	- CTE 循环引用检查
	- 视图内容验证
	- 权限检查

在梳理完整体的流程之后, 会对如 [BindSchemaOrCatalog](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/tableref/bind_basetableref.cpp#L171), [bind_context.AddBaseTable](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/tableref/bind_basetableref.cpp#L256) 等核心函数再做展开, 大体上就是先去 `catalog` 中找, 找到了再根据是表或者视图进行区分.
```c++ title:"src/planner/binder/tableref/bind_basetableref.cpp"
unique_ptr<BoundTableRef> Binder::Bind(BaseTableRef &ref) {
	// 0. 检查是否是CTE引用, 略过
	...
	
	// 1. 表或者视图的catalog的查找
	BindSchemaOrCatalog(ref.catalog_name, ref.schema_name);
	auto table_or_view = entry_retriever.GetEntry(
	    CatalogType::TABLE_ENTRY,
	    ref.catalog_name, 
	    ref.schema_name,
	    ref.table_name,
	    OnEntryNotFound::RETURN_NULL, 
	    error_context
	);

	// 2. 如果找不到表，尝试替换扫描或者自动加载扩展, 失败则报错, 略过
	if (!table_or_view) {
		...
	}
	
	// 3. 绑定检查
	switch (table_or_view->type) {
	case CatalogType::TABLE_ENTRY: {
	    // 3a. 基础表处理
	    // 获取扫描函数
	    // 收集列信息
	    // 创建LogicalGet	    
	    // 更新绑定上下文
	    bind_context.AddBaseTable(table_index, ref.alias, table_names, 
	                            table_types, col_ids, ref.table_name);
	                            
	    return make_uniq_base<BoundTableRef, BoundBaseTableRef>(table, std::move(logical_get));
	}
	
	case CatalogType::VIEW_ENTRY: {
	    // 3b. 视图处理
	    // 创建新的视图绑定器
	    // 将视图转换为子查询
	    SubqueryRef subquery(...);
	    // 绑定子查询
	    auto bound_child = view_binder->Bind(subquery);
	    // 验证视图的类型和名称
	    // ... 类型检查和名称检查
	    return bound_child;
	}
	}

}
```

下面我们对这里相关的代码做稍微详细一点的展开.
#### catalog

让我们先看这两行代码
```c++ title:"src/planner/binder/tableref/bind_basetableref.cpp"
// 1. 表或者视图的catalog的查找
BindSchemaOrCatalog(ref.catalog_name, ref.schema_name);
auto table_or_view = entry_retriever.GetEntry(
    CatalogType::TABLE_ENTRY,      // 查找类型：表
    ref.catalog_name,              // catalog名
    ref.schema_name,               // schema名
    ref.table_name,                // 表名
    OnEntryNotFound::RETURN_NULL,  // 找不到时返回NULL而不是抛异常
    error_context                  // 错误上下文
);
```

 [BindSchemaOrCatalog](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/statement/bind_create.cpp#L47) 会通过 [ClientContext](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/main/client_context.hpp#L65) 中已经封装好的信息, 解析和验证catalog和schema的路径。在DuckDB中，表的完整路径是：`catalog_name.schema_name.table_name`。
```sql
-- 完整形式
SELECT * FROM my_catalog.my_schema.my_table;
-- 简写形式
SELECT * FROM my_table;  -- 使用默认catalog和schema
```
- 如果用户没有指定catalog和schema，使用默认值
- 验证指定的catalog和schema是否存在

```c++ title:"src/planner/binder/statement/bind_create.cpp"
void Binder::BindSchemaOrCatalog(ClientContext &context, string &catalog, string &schema) {
	if (catalog.empty() && !schema.empty()) {
		// schema is specified - but catalog is not
		// try searching for the catalog instead
		auto &db_manager = DatabaseManager::Get(context);
		auto database = db_manager.GetDatabase(context, schema);
		if (database) {
			// we have a database with this name
			// check if there is a schema
			auto &search_path = *context.client_data->catalog_search_path;
			auto catalog_names = search_path.GetCatalogsForSchema(schema);
			if (catalog_names.empty()) {
				catalog_names.push_back(DatabaseManager::GetDefaultDatabase(context));
			}
			for (auto &catalog_name : catalog_names) {
				auto &catalog = Catalog::GetCatalog(context, catalog_name); // 这里会做权限验证
				if (catalog.CheckAmbiguousCatalogOrSchema(context, schema)) {
					throw BinderException(
					    "Ambiguous reference to catalog or schema \"%s\" - use a fully qualified path like \"%s.%s\"",
					    schema, catalog_name, schema);
				}
			}
			catalog = schema;
			schema = string();
		}
	}
}
```

***

[GetEntry](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/catalog/catalog_entry_retriever.cpp#L36) 则是在确定了catalog和schema路径后，尝试从catalog系统中获取表或视图的元数据, 这里就不往下跟踪了, Catalog 是 shard-compoents 的内容, 以后有机会再深入.

#### TABLE_ENREY
以 `BaseTableRef` 为参数的 [Bind](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/tableref/bind_basetableref.cpp#L82) 在得到 `CatalogEntry` 之后, 会跟上一个 `switch-case` , 我们这里就看我们这个例子中的处理逻辑的部分
```c++ title:"src/planner/binder/tableref/bind_basetableref.cpp"
case CatalogType::TABLE_ENTRY: {
    // 1. 生成表索引
    auto table_index = GenerateTableIndex(); // idx_t Binder::GenerateTableIndex
    auto &table = table_or_view->Cast<TableCatalogEntry>();

    // 2. 注册数据库读操作
    auto &properties = GetStatementProperties();
    properties.RegisterDBRead(table.ParentCatalog(), context);

    // 3. 获取表的扫描函数
    unique_ptr<FunctionData> bind_data;
    auto scan_function = table.GetScanFunction(context, bind_data);

    // 4. 收集表的列信息
    vector<LogicalType> table_types;
    vector<string> table_names;
    for (auto &col : table.GetColumns().Logical()) {
        table_types.push_back(col.Type());
        table_names.push_back(col.Name());
        return_types.push_back(col.Type());
        return_names.push_back(col.Name());
    }
    
    // 5. 处理列名别名
    table_names = BindContext::AliasColumnNames(ref.table_name, table_names, ref.column_name_alias);

    // 6. 创建LogicalGet算子
    auto logical_get = make_uniq<LogicalGet>(
        table_index,
        scan_function,
        std::move(bind_data),
        std::move(return_types),
        std::move(return_names),
        table.GetRowIdType()
    );

    // 7. 更新绑定上下文
    auto &col_ids = logical_get->GetMutableColumnIds();
    bind_context.AddBaseTable(table_index, ref.alias, table_names, table_types, col_ids, ref.table_name);

    // 8. 返回绑定结果
    return make_uniq_base<BoundTableRef, BoundBaseTableRef>(table, std::move(logical_get));
}
```
##### table.GetScanFunction

其中第三步的扫描函数, 会通过
```c++ title:"src/catalog/catalog_entry/duck_table_entry.cpp"
TableFunction DuckTableEntry::GetScanFunction(ClientContext &context, unique_ptr<FunctionData> &bind_data) {
	bind_data = make_uniq<TableScanBindData>(*this); // 创建 TableScanBindData 并赋值给 bind_data
	return TableScanFunction::GetFunction();
}
```
- [TableScanBindData](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/function/table/table_scan.hpp#L19) 继承自 FunctionData，用于存储表扫描时需要的信息, 下面是其一些字段的注释
 ```c++ title:"src/include/duckdb/function/table_function.hpp"
//! The table to scan.
TableCatalogEntry &table;
//! The old purpose of this field has been deprecated.
//! We now use it to express an index scan in the ANALYZE call.
//! I.e., we const-cast the bind data and set this to true, if we opt for an index scan.
bool is_index_scan;
//! Whether or not the table scan is for index creation.
bool is_create_index;
```
这样设计的目的是为了将表的定义信息 (DuckTableEntry) 和表的扫描操作 (TableScanFunction) 解耦，通过这个作为中间层来传递必要的信息, 后面在表扫描的上下文中都会用到, 我们再往下看.

[GetScanFunction](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/catalog/catalog_entry/duck_table_entry.cpp#L899) 返回的类型是 [TableFunction](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/function/table_function.hpp#L298), 它一些相关的成员大体上如下所示
```c++ title:"src/include/duckdb/function/table_function.hpp"
class TableFunction : public SimpleNamedParameterFunction {
public:
    // 核心函数指针
    table_function_t function;                      // 实际的扫描函数
    table_function_init_local_t init_local;         // 初始化本地状态
    table_function_init_global_t init_global;       // 初始化全局状态
    
    // 优化相关
    bool projection_pushdown = false;               // 是否支持投影下推
    bool filter_pushdown = false;                   // 是否支持过滤下推
    bool filter_prune = false;                      // 是否支持过滤剪枝
    
    // 统计信息和依赖
    table_statistics_t statistics;                  // 统计信息函数
    table_dependency_t dependency;                  // 依赖信息函数
    table_cardinality_t cardinality;               // 基数估算函数
    
public:
	DUCKDB_API
	TableFunction(string name, vector<LogicalType> arguments, table_function_t function,
	              table_function_bind_t bind = nullptr, table_function_init_global_t init_global = nullptr,
	              table_function_init_local_t init_local = nullptr);
};
```

构造函数中可以与 [TableScanFunction:GetFunction](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/function/table/table_scan.cpp#L622) 进行对应, 可以看到这里其注册了一个函数指针, 然后做一些其他的初始化, 以便后续的状态管理和优化策略的选择.
```c++ title:"src/function/table/table_scan.cpp"
TableFunction TableScanFunction::GetFunction() {
	// 创建基础表扫描函数
    TableFunction scan_function(
    	"seq_scan",     // 函数名
    	{},             // 参数列表(空)
    	TableScanFunc   // 实际的扫描函数指针
    );
    scan_function.init_local = TableScanInitLocal;       // 初始化本地状态
    scan_function.init_global = TableScanInitGlobal;     // 初始化全局状态
    scan_function.statistics = TableScanStatistics;      // 统计信息
    scan_function.pushdown_complex_filter = nullptr;      // 复杂过滤下推
    scan_function.projection_pushdown = true;            // 投影下推
    scan_function.filter_pushdown = true;                 // 过滤下推
    scan_function.filter_prune = true;                    // 过滤剪枝
    // ...其他配置
    return scan_function;
}
```

上面的 `TableScanFunc` 会把逻辑转发到具体的扫描状态实现, 这里又有一个新的结构体 [TableScanGlobalState](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/function/table/table_scan.cpp#L68), 它维护一些扫描状态的信息, 如最大线程数, 需要投影的列 ID, 扫描列的类型之类的, 然后具体的 `TableScanFunc` 是一个纯虚函数, 需要子类具体的实现
```c++ title:"src/function/table/table_scan.cpp"
static void TableScanFunc(ClientContext &context, TableFunctionInput &data_p, DataChunk &output) {
    // 转发到具体的扫描状态实现
    auto &g_state = data_p.global_state->Cast<TableScanGlobalState>();
    g_state.TableScanFunc(context, data_p, output);
}
```

[DuckTableScanState](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/function/table/table_scan.cpp#L199) 具体地实现了 [TableScanFunc](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/function/table/table_scan.cpp#L231), 这个函数应该会在执行的阶段, LogicalGet算子会使用这里存储的扫描函数, 我们的对 `GetFunction` 的探索差不多就到这里
```c++ title:"src/function/table/table_scan.cpp"
class DuckTableScanState : public TableScanGlobalState {
public:
    ParallelTableScanState state;  // 并行扫描状态

    void TableScanFunc(ClientContext &context, 
                      TableFunctionInput &data_p, 
                      DataChunk &output) override {
        // 1. 获取必要的上下文
        auto &bind_data = data_p.bind_data->Cast<TableScanBindData>();
        auto &duck_table = bind_data.table.Cast<DuckTableEntry>();
        auto &storage = duck_table.GetStorage();
        auto &l_state = data_p.local_state->Cast<TableScanLocalState>();

        do {
            if (CanRemoveFilterColumns()) {
                // 2a. 有投影的情况：先扫描所有列
                l_state.all_columns.Reset();
                storage.Scan(tx, l_state.all_columns, l_state.scan_state);
                // 然后只保留需要的列
                output.ReferenceColumns(l_state.all_columns, projection_ids);
            } else {
                // 2b. 无投影的情况：直接扫描
                storage.Scan(tx, output, l_state.scan_state);
            }

            // 3. 检查是否有数据
            if (output.size() > 0) {
                return;
            }

            // 4. 获取下一批数据
            auto next = storage.NextParallelScan(context, state, l_state.scan_state);
            if (!next) {
                return;  // 没有更多数据了
            }
        } while (true);
    }
};
```
针对这个函数本身, 它的一些总结
- 并行扫描：
	- ParallelTableScanState管理多线程扫描
	- NextParallelScan分配下一个扫描范围
- 投影优化：
	- CanRemoveFilterColumns()判断是否需要投影
	- 如果需要，先扫描所有列然后投影
	- 如果不需要，直接扫描需要的列
- 批处理：
	- 使用DataChunk进行批量数据处理
	- 每次扫描一个chunk的数据
- 状态管理：
	- TableScanState维护扫描位置
	- LocalTableFunctionState维护本地状态
	- GlobalTableFunctionState维护全局状态

***

如果读者还想往下深入, 也许可以看一下这些部分
- ParallelTableScanState的实现
- DataChunk的结构
- 或者storage.Scan的具体实现？

#### LogicalGet算子的创建

在活了扫描函数, 收集了列的信息之后, 就会创建一个 `LogicalGet` 的算子, 把相关信息都塞进去, 这一步似乎没有太多好说的, 主要是在这个流程中 `LogicalGet` 并没有被使用, 等待后续的流程与这里相呼应
```c++ title:"src/planner/binder/tableref/bind_basetableref.cpp"
case CatalogType::TABLE_ENTRY: {
    auto table_index = GenerateTableIndex();
    auto &table = table_or_view->Cast<TableCatalogEntry>();

    // 1. 获取扫描函数和绑定数据
    unique_ptr<FunctionData> bind_data;
    auto scan_function = table.GetScanFunction(context, bind_data);

    // 2. 收集表的列信息
    vector<LogicalType> table_types;
    vector<string> table_names;
    for (auto &col : table.GetColumns().Logical()) {
        table_types.push_back(col.Type());
        table_names.push_back(col.Name());
        return_types.push_back(col.Type());
        return_names.push_back(col.Name());
    }

    // 3. 创建LogicalGet算子
    auto logical_get = make_uniq<LogicalGet>(
        table_index,              // 表的唯一标识
        scan_function,            // 扫描函数
        std::move(bind_data),     // 绑定数据
        std::move(return_types),  // 返回类型
        std::move(return_names),  // 返回列名
        table.GetRowIdType()      // 行ID类型
    );
```

#### bind_context中表信息的注册

最后就是绑定上下文的更新了, 
```c++ title:""
    // 4. 更新绑定上下文
    auto &col_ids = logical_get->GetMutableColumnIds();
    bind_context.AddBaseTable(
        table_index,     // 表索引
        ref.alias,       // 表别名
        table_names,     // 列名
        table_types,     // 列类型
        col_ids,         // 列ID
        ref.table_name   // 原始表名
    );

    // 5. 返回绑定结果
    return make_uniq_base<BoundTableRef, BoundBaseTableRef>(table, std::move(logical_get));
}
```

其中 [AddBaseTable](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/bind_context.cpp#L613) 这个函数, 会调用到 `AddBinding` , 就是将相关的绑定关系按照顺序塞入 [BindContext](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/bind_context.hpp#L42) 中
```c++ title:""
void BindContext::AddBinding(unique_ptr<Binding> binding) {
	bindings_list.push_back(std::move(binding));
}
```

[BindContext](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/bind_context.hpp#L42) 是绑定关系的核心数据结构, 它维护了很多映射关系, 不过对于当前的FROM t1绑定来说，我们只是在建立基础的表名和列的映射, 后面会有更多的展开, 马上在 `where a > 10` 子句的处理中, 我们就能体会到, 非限定列名（如a）的解析就是需要在`bindings_list`中查找的

#### 小结

在 FROM 子句中，通过 bind_context. AddBaseTable 注册了 t1 表的信息

```sql
SELECT a, 
       (SELECT MAX(x) FROM t2 WHERE t2.id = t1.id) as sub_max,
       COUNT(*) as cnt
FROM t1           -- 这部分的绑定基本完成
WHERE a > 10
GROUP BY a
HAVING COUNT(*) > 1
ORDER BY a DESC;
```

### 进入 BindSelectNode

可以看到, [BindSelectNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/query_node/bind_select_node.cpp#L406) 需要利用到上面已经处理好的 `From` 的逻辑, 在这里会绑定
1. 状态初始化：为查询处理的各个阶段设置状态
2. 数据源设置：处理输入表
3. 表达式绑定器设置：准备表达式处理环境
4. 过滤和计算绑定：处理WHERE和SELECT表达式
5. 分组和聚合绑定：处理GROUP BY和HAVING
6. 结果修饰符绑定：处理ORDER BY、LIMIT/OFFSET和DISTINCT
7. 状态收尾：设置最终的输出格式
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
unique_ptr<BoundQueryNode> Binder::BindSelectNode(SelectNode &statement, unique_ptr<BoundTableRef> from_table) {
    // 1. 初始化查询处理阶段的状态索引
    auto result = make_uniq<BoundSelectNode>();
    result->projection_index = GenerateTableIndex();  // 投影
    result->group_index = GenerateTableIndex();       // 分组
    result->aggregate_index = GenerateTableIndex();   // 聚合
    result->window_index = GenerateTableIndex();      // 窗口
    result->prune_index = GenerateTableIndex();       // 剪枝

    // 2. 设置数据源
    result->from_table = std::move(from_table);

    // 3. 设置表达式绑定器和绑定状态
    SelectBinder select_binder(*this, context);
    SelectBindState select_state;

    // 4. 绑定过滤和计算表达式
    // 4.1 基础过滤
    if (statement.where_clause) {
        result->where_clause = select_binder.Bind(statement.where_clause);
    }
    // 4.2 投影表达式
    BindSelectList(statement, select_binder, *result, select_state);
    
    // 5. 绑定分组和聚合
    // 5.1 GROUP BY
    if (!statement.groups.empty()) {
        BindGroups(statement, select_binder, *result);
    }
    // 5.2 分组后过滤
    if (statement.having) {
        result->having = select_binder.Bind(statement.having);
    }

    // 6. 绑定结果修饰符
    // 6.1 排序
    if (!statement.orders.empty()) {
        BindModifiers(*result, statement.orders, result->orders);
    }
    // 6.2 分页
    if (statement.limit || statement.offset) {
        BindLimitModifier(statement, order_binder, *result);
    }
    // 6.3 去重
    if (statement.HasDistinct()) {
        BindDistinct(statement, *result, select_state);
    }

    // 7. 最终状态设置
    result->names = select_state.names;
    result->types = select_state.types;
    result->need_prune = select_state.need_prune;
    
    return result;
}
```

因为代码比较长, 这里适当地做了一些提炼和抽象, 并且删去了和这个 SQL 并没有什么关联的复杂度, 比如说 `*` 会在这里被展开, 也算是[[理论/数据库系统/Bottom Up/0-Frontend/数据库前端职责概述#4. 查询重写|查询重写]] (吗?), 还有像 `QUALIFY` , 以及 `UNNEST` 的处理, 都在这个函数中有体现.

代码一开头抽出了一个 [BoundSelectNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/query_node/bound_select_node.hpp#L37), 它的注释是 `Bound equivalent of SelectNode` , 封装了 `select` 在绑定过程中可能会用到的所有的成员
```c++ title:"src/include/duckdb/planner/query_node/bound_select_node.hpp"
class BoundSelectNode : public BoundQueryNode {
	...
	
	//! Bind information
	SelectBindState bind_state;
	//! The projection list
	vector<unique_ptr<Expression>> select_list;
	//! The FROM clause
	unique_ptr<BoundTableRef> from_table;
	//! The WHERE clause
	unique_ptr<Expression> where_clause;
	//! list of groups
	BoundGroupByNode groups;
	//! HAVING clause
	unique_ptr<Expression> having;
	//! QUALIFY clause
	unique_ptr<Expression> qualify;
	//! SAMPLE clause
	unique_ptr<SampleOptions> sample_options;

	//! The amount of columns in the final result
	idx_t column_count;
	//! The amount of bound columns in the select list
	idx_t bound_column_count = 0;
	...
}
```

下面的内容就是继续针对那个 SQL 的剩余部分, 按顺序观察接下来的绑定关系是如何进行处理的, 特别地, 如何利用上 `From` 字句的绑定结果
#### where a > 10

回忆一下我们要分析的具体SQL
```sql
SELECT a, 
       (SELECT MAX(x) FROM t2 WHERE t2.id = t1.id) as sub_max,
       COUNT(*) as cnt
FROM t1
WHERE a > 10        -- 我们先分析这个WHERE子
GROUP BY a
HAVING COUNT(*) > 1
ORDER BY a DESC;
```

在 [BoundSelectNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/query_node/bound_select_node.hpp#L37) 中处理WHERE子句: 
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
if (statement.where_clause) {
    // 1. 首先处理WHERE子句中的星号表达式
    BindWhereStarExpression(statement.where_clause);

    // 2. 创建列别名绑定器和WHERE绑定器
    ColumnAliasBinder alias_binder(bind_state);
    WhereBinder where_binder(*this, context, &alias_binder);

    // 3. 移动WHERE条件并进行绑定
    unique_ptr<ParsedExpression> condition = std::move(statement.where_clause);
    result->where_clause = where_binder.Bind(condition);
}
```
对于我们的WHERE a > 10，这里有几个关键步骤：
- BindWhereStarExpression - 处理WHERE中的星号表达式（虽然我们的例子中没有）
	- ColumnAliasBinder - 处理列别名引用
	- 允许在 WHERE 子句中引用 SELECT 列表中定义的别名
- [WhereBinder](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/expression_binder/where_binder.hpp#L18) - 实际的WHERE条件绑定器
	- 继承自 [ExpressionBinder](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/expression_binder.hpp#L70)
	- 专门处理 WHERE 子句的表达式绑定, 但其实用的 `Bind` 还是一样的, 不过有自己的 [BindColumnRef](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder/where_binder.cpp#L12)

这里的关键就是 [where_binder.Bind](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/query_node/bind_select_node.cpp#L455), 它会进入 `BindExpression::Bind` , 调用下面的方法, 进入 `BindExpression` 方法.
```c++ title:"src/planner/expression_binder.cpp"
ErrorData ExpressionBinder::Bind(unique_ptr<ParsedExpression> &expr, idx_t depth, bool root_expression) {
    // 如果表达式已经绑定，直接返回
    if (expr->GetExpressionClass() == ExpressionClass::BOUND_EXPRESSION) {
        return ErrorData();
    }
    // 绑定表达式
    BindResult result = BindExpression(expr, depth, root_expression);
    if (result.HasError()) {
        return std::move(result.error);
    }
    // 成功绑定：用BoundExpression替换节点
    expr = make_uniq<BoundExpression>(std::move(result.expression));
    // ...
}
```

然后通过虚函数调用进入 `WhereBinder::BindExpression` , 对于我们的表达式是 `a > 10` , 在 `Parse` 阶段就已经被构建成了一个表达式树：
```
ComparisonExpression (>)
├── ColumnRefExpression (a)
└── ConstantExpression (10)
```

可以看到也是一个 switch, 做一些绑定的检查, 然后我们的 `a > 10` 是一个 ComparisonExpression，因此会走到 default 分支
```c++ title:"src/planner/expression_binder/where_binder.cpp"
BindResult WhereBinder::BindExpression(unique_ptr<ParsedExpression> &expr_ptr, 
                                     idx_t depth, 
                                     bool root_expression) {
    auto &expr = *expr_ptr;
    switch (expr.GetExpressionClass()) {
    case ExpressionClass::DEFAULT:
        return BindUnsupportedExpression(expr, depth, 
               "WHERE clause cannot contain DEFAULT clause");
    case ExpressionClass::WINDOW:
        return BindUnsupportedExpression(expr, depth, 
               "WHERE clause cannot contain window functions!");
    case ExpressionClass::COLUMN_REF:
        return BindColumnRef(expr_ptr, depth, root_expression);
    default:
        // 其他类型的表达式(如比较表达式)委托给父类ExpressionBinder处理
        return ExpressionBinder::BindExpression(expr_ptr, depth);
    }
}
```

也就是
- 由父类的 [ExpressionBinder::BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder.cpp#L59) 去派分如何比较表达式, 是一个负责类型分发的 switch
	- 比较表达式的左右两边会分别被绑定：
		- 左边 a 是 ColumnRef，会调用 BindColumnRef
		- 右边 10 是 Constant，由父类处理

委托给父类 ExpressionBinder 的 [BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder.cpp#L59) 的情况代码大体如下所示, 实际上, 比较表达式的两个子节点的处理也会重新回到这个地方来进行再次分派.
```c++ title:"src/planner/expression_binder.cpp"
BindResult ExpressionBinder::BindExpression(unique_ptr<ParsedExpression> &expr, idx_t depth, bool root_expression) {
	auto stack_checker = StackCheck(*expr);

	auto &expr_ref = *expr;
	switch (expr_ref.GetExpressionClass()) {
	case ExpressionClass::BETWEEN:
		return BindExpression(expr_ref.Cast<BetweenExpression>(), depth);
	case ExpressionClass::CASE:
		return BindExpression(expr_ref.Cast<CaseExpression>(), depth);
	...
	case ExpressionClass::COMPARISON:
		return BindExpression(expr_ref.Cast<ComparisonExpression>(), depth);
	case ExpressionClass::FUNCTION: {
		auto &function = expr_ref.Cast<FunctionExpression>();
		if (IsUnnestFunction(function.function_name)) {
			// special case, not in catalog
			return BindUnnest(function, depth, root_expression);
		}
		// binding a function expression requires an extra parameter for macros
		return BindExpression(function, depth, expr);
	}
	...
	case ExpressionClass::STAR:
		return BindResult(BinderException::Unsupported(expr_ref, "STAR expression is not supported here"));
	default:
		throw NotImplementedException("Unimplemented expression class");
	}
}
```
##### 比较表达式的绑定

而以 `ComparisonExpression` 为参数的 [BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/expression/bind_comparison_expression.cpp#L157) 就会先尝试递归地绑定左右孩子, 然后在进行比较表达式的绑定, 并在必要的时候做类型转换
```c++ title:"src/planner/binder/expression/bind_comparison_expression.cpp"
BindResult ExpressionBinder::BindExpression(ComparisonExpression &expr, idx_t depth) {
	// first try to bind the children of the case expression
	ErrorData error;
	BindChild(expr.left, depth, error);
	BindChild(expr.right, depth, error);
	
	...
	// add casts (if necessary)
	left = BoundCastExpression::AddCastToType(context, std::move(left), input_type,
	                                          input_type.id() == LogicalTypeId::ENUM);
	right = BoundCastExpression::AddCastToType(context, std::move(right), input_type,
	                                           input_type.id() == LogicalTypeId::ENUM);

	PushCollation(context, left, input_type);
	PushCollation(context, right, input_type);

	// now create the bound comparison expression
	return BindResult(
	    make_uniq<BoundComparisonExpression>(expr.GetExpressionType(), std::move(left), std::move(right))
	);
}
```

这里的调用链路虽然不深, 但是跳来跳去挺烦的, 特别是最后到绑定 `ColumnRef` 的时候, 笔者绕了好一会, 本来想偷懒的, 但是没偷成功, 还是得把这里的关系给捋顺了.

##### col a 的绑定关系

这里有一个小知识点需要先补充, 那就是 [ColumnRefExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/parser/expression/columnref_expression.hpp#L19) 的构造方式, 从这里能看出来 `IsQualified` 的含义
```c++
// 1. 单列名构造
ColumnRefExpression::ColumnRefExpression(string column_name)
    : ColumnRefExpression(vector<string> {std::move(column_name)}) { }

// 2. 表名+列名构造
ColumnRefExpression::ColumnRefExpression(string column_name, string table_name)
    : ColumnRefExpression(table_name.empty() 
        ? vector<string> {std::move(column_name)}
        : vector<string> {std::move(table_name), std::move(column_name)}) { }
```
所以IsQualified表示列引用是否包含多个部分，比如：
- `a` -> 不限定 `(column_names = ["a"])`
- `t1.a` -> 限定 `(column_names = ["t1", "a"])`
- `schema.table.column` -> 限定 `(column_names = ["schema", "table", "column"])`

让我们回到 `where_binder` 的列名的绑定, 以 `ComparisonExpression` 为参数的 [BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/expression/bind_comparison_expression.cpp#L157) 会调用 `BindChild` , 它其实会绕一圈, 重新回到 [ExpressionBinder::BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder.cpp#L59) 进行派分, 随后才进入以 `ColumnRefExpression` 为参数的 [BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/expression/bind_columnref_expression.cpp#L423)
```
BindExpression(expr_ref.Cast<ColumnRefExpression>()
^
|   ...
|   
|   BindChild(expr.left, depth, error);
|	   -> ExpressionBinder::Bind(unique_ptr<ParsedExpression>
|		   -> BindResult ExpressionBinder::BindExpression(unique_ptr<ParsedExpression>
|			   ->  
|			    |
└──—————————————— // switch to 
			 
```

而以 `ColumnRefExpression` 为参数的 [BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/expression/bind_columnref_expression.cpp#L423), 它的逻辑主要分成这么几个部分
```c++ title:"src/planner/binder/expression/bind_columnref_expression.cpp"
BindResult ExpressionBinder::BindExpression(ColumnRefExpression &col_ref_p, idx_t depth, bool root_expression) {
    // [1] 名称提取模式检查 - 通常不会走这个分支
    if (binder.GetBindingMode() == BindingMode::EXTRACT_NAMES) {
        return BindResult(make_uniq<BoundConstantExpression>(Value(LogicalType::SQLNULL)));
    }

    // [2] 核心步骤：尝试将列名转换为限定形式（如：将'a'转换为't1.a'）
    ErrorData error;
    auto expr = QualifyColumnName(col_ref_p, error);
    if (!expr) {
        // [2.1] 如果限定失败且是非限定列名（如我们的'a'），尝试其他方式
        if (!col_ref_p.IsQualified()) {
            // 尝试作为别名绑定（如SELECT a as b中的b）
            BindResult alias_result;
            auto found_alias = TryBindAlias(col_ref_p, root_expression, alias_result);
            if (found_alias) {
                return alias_result;
            }
            // 尝试作为SQL值函数（如CURRENT_TIME）
            auto value_function = GetSQLValueFunction(col_ref_p.GetColumnName());
            if (value_function) {
                return BindExpression(value_function, depth);
            }
        }
        error.AddQueryLocation(col_ref_p);
        return BindResult(std::move(error));
    }

    // [3] 保存查询位置信息用于错误报告
    expr->SetQueryLocation(col_ref_p.GetQueryLocation());

    // [4] 处理特殊表达式（如生成列、结构体提取等）
    if (expr->GetExpressionType() != ExpressionType::COLUMN_REF) {
        auto alias = expr->GetAlias();
        auto result = BindExpression(expr, depth);
        if (result.expression) {
            result.expression->SetAlias(std::move(alias));
        }
        return result;
    }

    // [5] 核心绑定逻辑：将限定后的列引用绑定到具体的表
    BindResult result;
    auto &col_ref = expr->Cast<ColumnRefExpression>();
    D_ASSERT(col_ref.IsQualified());  // 此时列名应该已经被限定
    auto &table_name = col_ref.GetTableName();

    // [5.1] 选择绑定方式：宏绑定或常规绑定
    if (binder.macro_binding && table_name == binder.macro_binding->GetAlias()) {
        result = binder.macro_binding->Bind(col_ref, depth);
    } else {
        // 对于WHERE a > 10，会走这个分支
        result = binder.bind_context.BindColumn(col_ref, depth);
    }

    // [6] 错误处理和结果记录
    if (result.HasError()) {
        result.error.AddQueryLocation(col_ref_p);
        return result;
    }

    // [7] 记录成功绑定的列信息
    BoundColumnReferenceInfo ref;
    ref.name = col_ref.column_names.back();
    ref.query_location = col_ref.GetQueryLocation();
    bound_columns.push_back(std::move(ref));
    return result;
}
```

对于我们的 SQL `where a > 10` , 这里的 `a` 是单独的列名, 所以本来 `IsQualified` 是 `false` , 但是会在 [QualifyColumnName](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/expression/bind_columnref_expression.cpp#L363) 的修正下变成 `true` , 也就是将非限定列名（如a）转换为限定形式（如t1. a）, 就是一个[[理论/数据库系统/Bottom Up/0-Frontend/数据库前端职责概述#4. 查询重写|查询重写]]

然后我们就进入了以 `ColumnRefExpression` 为参数的 [BindColumn](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/bind_context.cpp#L392), 如果错过了上面的第二步, 那可能会在第一个 if 的判断中就误以为这个函数不是我们应该进入的入口

```c++ title:src/planner/bind_context.cpp
BindResult BindContext::BindColumn(ColumnRefExpression &colref, idx_t depth) {
	if (!colref.IsQualified()) {
 	   throw InternalException("Could not bind alias \"%s\"!", colref.GetColumnName());
	}
	
	...
	...
	...
	
   
	if (binder.macro_binding && table_name == binder.macro_binding->GetAlias()) {
		result = binder.macro_binding->Bind(col_ref, depth);
	} else {
`	   result = binder.bind_context.BindColumn(col_ref, depth);
	}
	
	if (result.HasError()) {
		...
	}

	// we bound the column reference
	BoundColumnReferenceInfo ref;
	ref.name = col_ref.column_names.back();
	ref.query_location = col_ref.GetQueryLocation();
	bound_columns.push_back(std::move(ref));
	return result;
}
```

不难看出, 这里的关键是 [BindContext::BindColumn](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/bind_context.cpp#L392), 不过在这之前要先回想一下，在之前 `from` 子句的绑定时, 已经通过 `AddBaseTable` 完成了 [[实践/examples/Database/duckdb/duckdb-前端#bind_context中表信息的注册|bind_context中表信息的注册]]
```c++
class BindContext {
    //! The set of bound tables
    vector<unique_ptr<Binding>> bindings_list;
    //! The mapping of (table) name to Binding index
    case_insensitive_map_t<vector<reference<Binding>>> bindings;
};

bind_context.AddBaseTable(
    table_index,     // 表索引
    ref.alias,       // 表别名
    table_names,     // 列名
    table_types,     // 列类型
    col_ids,         // 列ID
    ref.table_name   // 原始表名
);
```

然后我们再来看 [BindContext::BindColumn](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/bind_context.cpp#L392), 所以当我们在 BindColumn 中查找列 a 时：
1. 首先会在 bind_context 中查找所有可能的表绑定
2. 对于每个表绑定，检查是否包含列 a
3. 如果找到匹配的列：
	- 创建 BoundColumnRefExpression
	- 设置正确的表索引和列索引

```c++ title:"src/planner/bind_context.cpp"
BindResult BindContext::BindColumn(ColumnRefExpression &colref, idx_t depth) {
    // 此时colref已经被QualifyColumnName处理过，应该是限定形式
    if (!colref.IsQualified()) {
        throw InternalException("Could not bind alias \"%s\"!", colref.GetColumnName());
    }

    // 获取绑定
    ErrorData error;
    BindingAlias alias;
    auto binding = GetBinding(GetBindingAlias(colref), colref.GetColumnName(), error);
    if (!binding) {
        return BindResult(std::move(error));
    }
    // 执行实际的绑定
    return binding->Bind(colref, depth);
}
```

最后是在以 `ColumnRefExpression` 的 [Bind](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/table_binding.cpp#L65) 中完成最后的绑定, 建立了列引用与FROM子句中表的联系
```c++ title:"src/planner/table_binding.cpp"
BindResult Binding::Bind(ColumnRefExpression &colref, idx_t depth) {
	column_t column_index;
	bool success = false;
	success = TryGetBindingIndex(colref.GetColumnName(), column_index);
	if (!success) {
		return BindResult(ColumnNotFoundError(colref.GetColumnName()));
	}
	ColumnBinding binding;
	binding.table_index = index;
	binding.column_index = column_index;
	LogicalType sql_type = types[column_index];
	if (colref.GetAlias().empty()) {
		colref.SetAlias(names[column_index]);
	}
	return BindResult(make_uniq<BoundColumnRefExpression>(colref.GetName(), sql_type, binding, depth));
}
```
##### 常量 10 的绑定
常量的绑定和列的绑定差不多, 只是一个是左节点一个是右节点.
```
BindExpression(expr_ref.Cast<ColumnRefExpression>()
^
|   ...
|   
|   BindChild(expr.right, depth, error);
|	   -> ExpressionBinder::Bind(unique_ptr<ParsedExpression>
|		   -> BindResult ExpressionBinder::BindExpression(unique_ptr<ParsedExpression>
|			   ->  
|			    |
└──—————————————— // switch to 
			 
```
以 `ConstanExpression` 为参数的 [BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/expression/bind_constant_expression.cpp#L7) 也只有短短几行.
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
// expression_binder.cpp
BindResult ExpressionBinder::BindExpression(ConstantExpression &expr, idx_t depth) {
    // 常量表达式的绑定相对简单
    // 创建BoundConstantExpression，保持原值和类型
    auto result = make_uniq<BoundConstantExpression>(expr.value);
    return BindResult(std::move(result));
}
```

然后我们再回到比较表达式绑定的开头, 做一些可能要类型转换的检查, where 子句的绑定关系就到这里结束了

##### 小结
在处理 `WHERE` 子句时:
- 解析为比较表达式(`ComparisonExpression`)
- 分别绑定左右子节点
	- 左节点 `a` 的绑定涉及：
	- 通过 `QualifyColumnName` 确定列属于哪个表
	- 通过 `bind_context.BindColumn` 与 `FROM` 子句中的表建立联系
- 右节点`10`作为常量直接绑定
- 最后合并为完整的比较表达式

#### group by
现在我们来看 `group by a` 这个子句的绑定关系应该是怎样的
```sql
SELECT a, 
       (SELECT MAX(x) FROM t2 WHERE t2.id = t1.id) as sub_max,
       COUNT(*) as cnt
FROM t1
WHERE a > 10
GROUP BY a   -- 现在轮到了这个子句
HAVING COUNT(*) > 1
ORDER BY a DESC;
```
[BindSelectNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/query_node/bind_select_node.cpp#L406) 其中的代码, 对应的大体上是下面的部分, 
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
unique_ptr<BoundQueryNode> Binder::BindSelectNode(SelectNode &statement, unique_ptr<BoundTableRef> from_table) {
    // ... 前面的部分我们已经讨论过
    
    if (!group_expressions.empty()) {
		// the statement has a GROUP BY clause, bind it
		unbound_groups.resize(group_expressions.size());
		// 1. 创建GROUP BY绑定器
		GroupBinder group_binder(*this, context, statement, result->group_index, bind_state, info.alias_map);
		for (idx_t i = 0; i < group_expressions.size(); i++) {
	
			// we keep a copy of the unbound expression;
			// we keep the unbound copy around to check for group references in the SELECT and HAVING clause
			// the reason we want the unbound copy is because we want to figure out whether an expression
			// is a group reference BEFORE binding in the SELECT/HAVING binder
			group_binder.unbound_expression = group_expressions[i]->Copy();
			group_binder.bind_index = i;
	
			// bind the groups
			// 2. 绑定每个GROUP BY表达式
			LogicalType group_type;
			auto bound_expr = group_binder.Bind(group_expressions[i], &group_type);
			D_ASSERT(bound_expr->return_type.id() != LogicalTypeId::INVALID);
	
			// find out whether the expression contains a subquery, it can't be copied if so
			auto &bound_expr_ref = *bound_expr;
			bool contains_subquery = bound_expr_ref.HasSubquery();
	
			// push a potential collation, if necessary
			bool requires_collation = ExpressionBinder::PushCollation(context, bound_expr, group_type);
			if (!contains_subquery && requires_collation) {
				// if there is a collation on a group x, we should group by the collated expr,
				// but also push a first(x) aggregate in case x is selected (uncollated)
				info.collated_groups[i] = result->aggregates.size();
	
				auto first_fun = FirstFunctionGetter::GetFunction(bound_expr_ref.return_type);
				vector<unique_ptr<Expression>> first_children;
				// FIXME: would be better to just refer to this expression, but for now we copy
				first_children.push_back(bound_expr_ref.Copy());
	
				FunctionBinder function_binder(*this);
				auto function = function_binder.BindAggregateFunction(first_fun, std::move(first_children));
				function->SetAlias("__collated_group");
				result->aggregates.push_back(std::move(function));
			}
			 // 3. 创建分组集合
			result->groups.group_expressions.push_back(std::move(bound_expr));
	
			// in the unbound expression we DO bind the table names of any ColumnRefs
			// we do this to make sure that "table.a" and "a" are treated the same
			// if we wouldn't do this then (SELECT test.a FROM test GROUP BY a) would not work because "test.a" <> "a"
			// hence we convert "a" -> "test.a" in the unbound expression
			unbound_groups[i] = std::move(group_binder.unbound_expression);
			ExpressionBinder::QualifyColumnNames(*this, unbound_groups[i]);
			info.map[*unbound_groups[i]] = i;
		}
	}
```

可以看到其关键是 [GroupBinder](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/expression_binder/group_binder.hpp#L20)
```c++
// planner/expression_binder/group_binder.hpp
class GroupBinder : public ExpressionBinder {
public:
    GroupBinder(Binder &binder, ClientContext &context, SelectBindState &bind_state, SelectNode &node)
        : ExpressionBinder(binder, context), bind_state(bind_state), node(node) {
        target_type = LogicalType::INVALID;  // GROUP BY可以是任意类型
    }

protected:
    BindResult BindExpression(unique_ptr<ParsedExpression> &expr_ptr, idx_t depth, bool root_expression) override {
        // 主要是复用ExpressionBinder的功能
        // 但会额外检查表达式是否可以用于GROUP BY
        return ExpressionBinder::BindExpression(expr_ptr, depth);
    }

private:
    SelectBindState &bind_state;
    SelectNode &node;
};

// src/planner/expression_binder/group_binder.cpp
BindResult GroupBinder::BindExpression(unique_ptr<ParsedExpression> &expr_ptr, idx_t depth, bool root_expression) {
	auto &expr = *expr_ptr;
	if (root_expression && depth == 0) {
		switch (expr.GetExpressionClass()) {
		case ExpressionClass::COLUMN_REF:
			return BindColumnRef(expr.Cast<ColumnRefExpression>());
		...
		case ExpressionClass::PARAMETER:
			throw ParameterNotAllowedException("Parameter not supported in GROUP BY clause");
		default:
			break;
		}
	}
	switch (expr.GetExpressionClass()) {
	case ExpressionClass::DEFAULT:
		return BindUnsupportedExpression(expr, depth, "GROUP BY clause cannot contain DEFAULT clause");
	case ExpressionClass::WINDOW:
		return BindUnsupportedExpression(expr, depth, "GROUP BY clause cannot contain window functions!");
	default:
		return ExpressionBinder::BindExpression(expr_ptr, depth);
	}
}
```
它也是类似 `WhereBinder` 的设计
- 使用基础的ExpressionBinder功能绑定列引用a
	- 检查这个列是否来自基表 `(t1)`
	- 这一点和 where 子句是类似的, 所以 [GroupBinder::BindColumnRef](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder/group_binder.cpp#L105) 就不过多展开了
- 记录这个分组表达式，供后续HAVING和SELECT使用, 我们重点来关注一下这一点, 它其实是在 `BindSelectNode` 通过 `result->groups.grouping_sets` 来完成的, 这些绑定后的分组表达式会影响
	- SELECT列表的处理：
		- 非分组列不能直接出现在`SELECT`列表中
		- 必须通过聚合函数（如我们SQL中的`COUNT(*)`）
		- 或者必须是`GROUP BY`中的表达式（如我们的`a`）
	- HAVING子句的处理：
		- HAVING 中只能使用：
			- GROUP BY中的表达式（如`a`）
			- 聚合函数（如`COUNT(*) > 1`）
			- 常量
	- 执行计划的生成：
		- 需要添加分组算子（`GroupingOperator`）
		- 确定分组键的类型和排序
		- 可能涉及到哈希表或排序的选择
##### 小结
- 绑定过程：
	- 与WHERE子句类似，主要通过`GroupBinder`处理
	- 对于`GROUP BY a`，复用了列引用绑定的机制
	- 将绑定后的表达式存储在`result->groups.group_expressions`中
- 关键影响：
	- 限制了`SELECT`列表中可以出现的表达式类型
	- 影响`HAVING`子句的处理
	- 为后续的执行计划提供分组信息

这里需要补充说明的是, 需要关注 [BoundGroupInformation](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/expression_binder/base_select_binder.hpp#L21) 的使用, 因为后面会发现, 在处理 `select list` 的时候, [SelectBinder](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/expression_binder/select_binder.hpp#L16) 会利用到这一部分的数据
```c++ title:src/include/duckdb/planner/expression_binder/base_select_binder.hpp
struct BoundGroupInformation {
	parsed_expression_map_t<idx_t> map;               // GROUP BY表达式映射
	case_insensitive_map_t<idx_t> alias_map;
	unordered_map<idx_t, idx_t> collated_groups;
};

class BaseSelectBinder : public ExpressionBinder {
public:
	BaseSelectBinder(Binder &binder, ClientContext &context, BoundSelectNode &node, BoundGroupInformation &info);
	...
}
```

##### group 的遗漏

其实 [BindSelectNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/query_node/bind_select_node.cpp#L406) 的代码后面还有一些内容, 是处理类似 `group by all` 或者是分组聚合的, 但是跟本例无关, 这里就不展开了了
- GROUP BY ALL的处理：
	- 会自动将 `SELECT` 列表中的所有表达式加入到`GROUP BY`中
- `grouping_sets` 的处理：
	- 支持更复杂的分组操作
	- 如 `GROUP BY GROUPING SETS ((a), (b), (a, b))`

#### having count (\*) > 1

```sql
SELECT a, 
       (SELECT MAX(x) FROM t2 WHERE t2.id = t1.id) as sub_max,
       COUNT(*) as cnt
FROM t1
WHERE a > 10
GROUP BY a
HAVING COUNT(*) > 1 -- 现在轮到了这个子句
ORDER BY a DESC;
```
现在让我们来看这个部分, 其实和 `group` 是很像的
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
// bind the HAVING clause, if any
if (statement.having) {
	HavingBinder having_binder(*this, context, *result, info, statement.aggregate_handling);
	ExpressionBinder::QualifyColumnNames(having_binder, statement.having);
	result->having = having_binder.Bind(statement.having);
}
```
aggregate_handling是一个枚举类型，定义了聚合的处理方式：
- `STANDARD_HANDLING`：标准聚合处理
	- 我们的 `HAVING COUNT(*) > 1` 就是这种情况
- `FORCE_AGGREGATES`：强制使用聚合
- `NO_AGGREGATES`：不允许聚合
	- `GROUP_BY_ALL`：我们刚才在 `GROUP BY`中看到的，自动将所有`SELECT`列加入`GROUP BY`

[HavingBinder](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/expression_binder/having_binder.hpp#L18) 的代码比较简单, 而且与前面讨论的 `where` 和 `group` 在逻辑上有各种交叠的部分, 这里就不展开了. 
```c++ title:"src/include/duckdb/planner/expression_binder/having_binder.hpp"
class HavingBinder : public BaseSelectBinder {
public:
	HavingBinder(Binder &binder, ClientContext &context, BoundSelectNode &node, BoundGroupInformation &info,
	             AggregateHandling aggregate_handling);

protected:
	BindResult BindLambdaReference(LambdaRefExpression &expr, idx_t depth);
	BindResult BindWindow(WindowExpression &expr, idx_t depth) override;
	BindResult BindColumnRef(unique_ptr<ParsedExpression> &expr_ptr, idx_t depth, bool root_expression) override;

	unique_ptr<ParsedExpression> QualifyColumnName(ColumnRefExpression &col_ref, ErrorData &error) override;

private:
	ColumnAliasBinder column_alias_binder;
	AggregateHandling aggregate_handling;
};
```

小结也略过了, 看 select list 吧
#### SELECT LIST

对于我们的 SQL 而言, 就是这几个部分
```sql
SELECT a,                                     -- 列引用，来自GROUP BY
       (SELECT MAX(x) FROM t2                 -- 相关子查询
        WHERE t2.id = t1.id) as sub_max,
       COUNT(*) as cnt                        -- 聚合函数
FROM t1
WHERE a > 10
GROUP BY a
HAVING COUNT(*) > 1
ORDER BY a DESC;
```

对于我们的三个 SELECT 项：
1. `a`
	- 这是一个列引用
	- 绑定过程类似 WHERE 子句中的列引用
	- 但因为有 `GROUP BY` ，需要额外检查这个列是否在 `GROUP BY` 列表中

2. `(SELECT MAX (x) FROM t2 WHERE t2. id = t1. id) as sub_max`
	- 这是一个相关子查询
	- 需要特殊的子查询绑定器
	- 需要处理相关性（ `t1.id` 的引用）

3. `COUNT(*) as cnt`
	- 这是一个聚合函数
	- 需要特殊的聚合绑定器
	- 需要与 `GROUP BY` 关联

再次回到 [BindSelectNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/query_node/bind_select_node.cpp#L406), 这回需要关注的是 `after that, we bind to the SELECT list` 之后的部分, 经过提炼之后的 `select list` 的处理框架大体上是这样的
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
// SELECT列表的绑定过程
// after that, we bind to the SELECT list
SelectBinder select_binder(*this, context, *result, info);

// 为可能的列表展开准备
vector<idx_t> group_by_all_indexes;      // GROUP BY ALL的列索引
vector<string> new_names;                // 新的列名（包括展开后的）
vector<LogicalType> internal_sql_types;  // 内部SQL类型

// 处理每个SELECT项
for (idx_t i = 0; i < statement.select_list.size(); i++) {
    // 1. 基本信息收集
    bool is_window = statement.select_list[i]->IsWindow();       // 是否是窗口函数
    idx_t unnest_count = result->unnests.size();                 // 当前UNNEST数量
    
    // 2. 绑定表达式
    LogicalType result_type;
    auto expr = select_binder.Bind(statement.select_list[i], &result_type, true);
    
    // 3. 列属性判断
    bool is_original_column = i < result->column_count;          // 是否是原始列
    bool can_group_by_all =                                      // 是否可以参与GROUP BY ALL
        statement.aggregate_handling == AggregateHandling::FORCE_AGGREGATES && 
        is_original_column;
    result->bound_column_count++;

    // 4. 特殊表达式处理
    if (expr->GetExpressionType() == ExpressionType::BOUND_EXPANDED) {
        // 处理UNNEST/UNLIST展开的结构体
        // ...
        continue;
    }

    // 5. 表达式属性标记
    if (expr->IsVolatile()) {                    // 易变表达式
        bind_state.SetExpressionIsVolatile(i);
    }
    if (expr->HasSubquery()) {                   // 包含子查询
        bind_state.SetExpressionHasSubquery(i);
    }
    bind_state.AddRegularColumn();

    // 6. GROUP BY ALL处理
    if (can_group_by_all && select_binder.HasBoundColumns()) {
        // 检查聚合和窗口函数的限制
        // ...
        group_by_all_indexes.push_back(i);
    }

    // 7. 结果收集
    result->select_list.push_back(std::move(expr));
    if (is_original_column) {
        new_names.push_back(std::move(result->names[i]));
        result->types.push_back(result_type);
    }
    internal_sql_types.push_back(result_type);

    // 8. 重置绑定（如果需要）
    if (can_group_by_all) {
        select_binder.ResetBindings();
    }
}
```

其实不难发现这里的关键函数还是 [ExpressionBinder::Bind](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder.cpp#L314), 因为 [SelectBinder](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/expression_binder/select_binder.hpp#L16) 也没有重写多少办法
```c++ title:“src/include/duckdb/planner/expression_binder/select_binder.hpp”
//! The SELECT binder is responsible for binding an expression within the SELECT clause of a SQL statement
class SelectBinder : public BaseSelectBinder {
public:
	SelectBinder(Binder &binder, ClientContext &context, BoundSelectNode &node, BoundGroupInformation &info);

protected:
	void ThrowIfUnnestInLambda(const ColumnBinding &column_binding) override;
	BindResult BindUnnest(FunctionExpression &function, idx_t depth, bool root_expression) override;
	BindResult BindColumnRef(unique_ptr<ParsedExpression> &expr_ptr, idx_t depth, bool root_expression) override;

	bool QualifyColumnAlias(const ColumnRefExpression &colref) override;
	unique_ptr<ParsedExpression> GetSQLValueFunction(const string &column_name) override;

protected:
	idx_t unnest_level = 0;
};
```

下面让我们分开讨论一下
##### a

因为前面WHERE子句已经讨论过列引用的基本绑定机制, 这里只需要重点关注 `GROUP BY` 相关的检查, 因为列引用的处理和前面的 `where` 字句的处理应该是类似的.
- 检查列`a`是否在`GROUP BY`列表中
- 或者是否是聚合函数的结果

```sql
SELECT a,                                     -- 这里的a
       (SELECT MAX(x) FROM t2 
        WHERE t2.id = t1.id) as sub_max,
       COUNT(*) as cnt
FROM t1
WHERE a > 10
GROUP BY a                                    -- 必须在GROUP BY中
HAVING COUNT(*) > 1
ORDER BY a DESC;
```

这一个部分的检查代码, 我找了好半天都没找到, 是利用 [duckdb shell](https://shell.duckdb.org) 反馈的报错信息才找到线索的
```shell
duckdb> CREATE TABLE sales (
   ...>     id INT,
   ...>     name VARCHAR,
   ...>     amount DECIMAL(10, 2)
   ...> );
   ...> 
   ...> INSERT INTO sales VALUES
   ...> (1, 'Alice', 200),
   ...> (1, 'Bob', 300),
   ...> (2, 'Charlie', 150),
   ...> (2, 'David', 250),
   ...> (3, 'Eve', 500),
   ...> (3, 'Frank', 600);
┌───────┐
│ Count │
╞═══════╡
│     6 │
└───────┘

duckdb> SELECT id, name, SUM(amount)
   ...> FROM sales
   ...> GROUP BY id;
Binder Error: column "name" must appear in the GROUP BY clause or must be part of an aggregate function.
Either add it to the GROUP BY list, or use "ANY_VALUE(name)" if the exact value of "name" is not important.
LINE 1: SELECT id, name, SUM(amount)
```
原来还是在 [BindSelectNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/query_node/bind_select_node.cpp#L406) 中, 藏在被我一开始打算[[实践/examples/Database/duckdb/duckdb-前端#group 的遗漏|放过]]的地方, 一开始的注释是代码中原来的位置保留的
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
	// in the normal select binder, we bind columns as if there is no aggregation
	// i.e. in the query [SELECT i, SUM(i) FROM integers;] the "i" will be bound as a normal column
	// since we have an aggregation, we need to either (1) throw an error, or (2) wrap the column in a FIRST() aggregate
	// we choose the former one [CONTROVERSIAL: this is the PostgreSQL behavior]
	if (!result->groups.group_expressions.empty() ||  // 有GROUP BY子句
	    !result->aggregates.empty() ||               // 有聚合函数
	    statement.having ||                          // 有HAVING子句
	    !result->groups.grouping_sets.empty())       // 有GROUPING SETS
	{
		// 然后检查是否允许聚合
		if (statement.aggregate_handling == AggregateHandling::NO_AGGREGATES_ALLOWED) {
		    throw BinderException("Aggregates cannot be present in a Project relation!");
		} else {
			// 关键的错误检查
			vector<BoundColumnReferenceInfo> bound_columns;
			if (select_binder.HasBoundColumns()) {
			    bound_columns = select_binder.GetBoundColumns();  // 获取SELECT中绑定的列
			}
			// ...
			if (!bound_columns.empty()) {
			    string error = "column \"%s\" must appear in the GROUP BY clause or must be part of an aggregate function.";
			    // ...添加额外的提示信息
			    throw BinderException(bound_columns[0].query_location, error, bound_columns[0].name);
			}
		}
	}
```
对于我们的例子,
- select_binder会记录所有绑定的非聚合列（这里是a）
	- 因为有 GROUP BY，会进行检查
		- 发现 a既不在GROUP BY中，也不是聚合函数
			- 抛出错误，建议要么加入GROUP BY，要么使用ANY_VALUE

***

我一开始以为这个检查会跟 `TryBindGroup` 和 `BindGroup` 有关系
```c++
struct BoundGroupInformation {
    parsed_expression_map_t<idx_t> map;          // GROUP BY表达式到索引的映射
    case_insensitive_map_t<idx_t> alias_map;     // 别名到索引的映射
    unordered_map<idx_t, idx_t> collated_groups; // 排序组
};

class BaseSelectBinder {
protected:
    idx_t TryBindGroup(ParsedExpression &expr);
    BindResult BindGroup(ParsedExpression &expr, idx_t depth, idx_t group_index);
}
```

前面在讨论 `group by` 的时候提过一嘴, [SelectBinder](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/expression_binder/select_binder.hpp#L16) 会接受 `BoundGroupInformation` 的信息
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
// 创建SELECT绑定器，传入了绑定结果和GROUP信息
SelectBinder select_binder(*this, context, *result, info);

// 绑定SELECT列表
for (idx_t i = 0; i < statement.select_list.size(); i++) {
    auto expr = select_binder.Bind(statement.select_list[i], &result_type, true);
    // ...
}
```

但其实关系不大, 我之前找错了. 这里就简单地记录一下
- [TryBindGroup](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder/base_select_binder.cpp#L42)：
	- 尝试将一个表达式匹配到已存在的GROUP BY表达式
	- 如果表达式在GROUP BY列表中，返回其索引
	- 如果不在，返回 INVALID_INDEX
	- 这是用于表达式重用，而不是用于检查 GROUP BY约束
- [BindGroup](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder/base_select_binder.cpp#L99)
	- 当找到匹配的GROUP BY表达式时，创建对应的绑定
	- 生成一个指向 GROUP BY 结果的列引用
	- 处理排序和 GROUPING SETS 的特殊情况

这两个函数主要用于优化和表达式重用，而不是用于强制GROUP BY的约束检查。GROUP BY的约束检查是在我们刚才看到的 `BindSelectNode` 中完成的。

##### 子查询
这是一个相关子查询
```sql
SELECT a,                                    
       (SELECT MAX(x) FROM t2                 -- 子查询
        WHERE t2.id = t1.id) as sub_max,
       COUNT(*) as cnt
FROM t1
WHERE a > 10
GROUP BY a
HAVING COUNT(*) > 1
ORDER BY a DESC;
```

需要详细讨论：
- 子查询的绑定机制
- 相关性的处理（t1.id如何在子查询中被识别和绑定）
- 子查询结果如何与外层查询集成
- 子查询的类型检查和结果类型推导

[ExpressionBinder::BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder.cpp#L59) 有一个分支是专门处理 `ExpressionClass::SUBQUERY` 的, 所以我们直接切到以 [SubqueryExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/parser/expression/subquery_expression.hpp#L18) 为参数的 [BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/expression/bind_subquery_expression.cpp#L84) 上分析, 并且, 这个部分也回应了在 Parser 阶段我们讨论过的[[实践/examples/Database/duckdb/duckdb-前端#Expression 层|对子查询的分类]], 这个函数可能有点长, 我们先纵览一下
```c++ title:"src/planner/binder/expression/bind_subquery_expression.cpp"
BindResult ExpressionBinder::BindExpression(SubqueryExpression &expr, idx_t depth) {
    // 1. 初始绑定阶段：绑定子查询节点
    if (expr.subquery->node->type != QueryNodeType::BOUND_SUBQUERY_NODE) {
        // 创建子查询绑定器并绑定节点
        auto subquery_binder = Binder::CreateBinder(context, &binder);
        subquery_binder->can_contain_nulls = true;
        auto bound_node = subquery_binder->BindNode(*expr.subquery->node);

        // 处理相关列（如我们例子中的t1.id）
        for (idx_t i = 0; i < subquery_binder->correlated_columns.size(); i++) {
            CorrelatedColumnInfo corr = subquery_binder->correlated_columns[i];
            if (corr.depth > 1) {
                corr.depth -= 1;
                binder.AddCorrelatedColumn(corr);
            }
        }

        // 创建绑定后的子查询节点
        expr.subquery = make_uniq<SelectStatement>();
        expr.subquery->node = make_uniq<BoundSubqueryNode>(...);
    }

    // 2. 子表达式绑定阶段
    if (expr.child) {
        auto error = Bind(expr.child, depth);
        if (error.HasError()) {
            return BindResult(std::move(error));
        }
    }

    // 3. 类型检查和结果处理阶段
    auto &bound_subquery = expr.subquery->node->Cast<BoundSubqueryNode>();
    vector<unique_ptr<Expression>> child_expressions;

    // 根据子查询类型进行不同处理
    if (expr.subquery_type == SubqueryType::EXISTS) {
        // EXISTS子查询：不需要检查列数，返回布尔值
        // 例如：WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)
    } else if (expr.subquery_type == SubqueryType::ANY) {
        // ANY/ALL子查询：需要处理比较操作
        // 例如：WHERE x > ANY (SELECT amount FROM t2)
        for (idx_t child_idx = 0; child_idx < child_expressions.size(); child_idx++) {
            // 处理类型兼容性和类型转换
            auto &child = child_expressions[child_idx];
            // ... 类型检查和转换逻辑
        }
    } else {
        // SCALAR子查询：必须返回单列
        // 例如我们的：(SELECT MAX(x) FROM t2 WHERE t2.id = t1.id)
        idx_t expected_columns = 1;
        if (bound_subquery.bound_node->types.size() != expected_columns) {
            throw BinderException(expr, "Subquery returns %zu columns - expected %d",
                                bound_subquery.bound_node->types.size(), expected_columns);
        }
    }

    // 4. 结果构建阶段
    // 根据子查询类型确定返回类型
    LogicalType return_type;
    if (expr.subquery_type == SubqueryType::SCALAR) {
        return_type = bound_node->types[0];
    } else {
        return_type = LogicalType(LogicalTypeId::BOOLEAN);
    }

    // 创建最终的绑定表达式
    auto result = make_uniq<BoundSubqueryExpression>(return_type);
    result->binder = std::move(subquery_binder);
    result->subquery = std::move(bound_node);
    result->subquery_type = expr.subquery_type;
    result->comparison_type = expr.comparison_type;

    return BindResult(std::move(result));
}
```
总结一下

- 标量子查询 [SubqueryType::SCALAR](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/common/enums/subquery_type.hpp#L20)：
	- 必须返回单列
	- 返回类型就是该列的类型
	- 这是我们例子中的情况：`(SELECT MAX(x) FROM t2 WHERE t2.id = t1.id)`
- EXISTS子查询 [SubqueryType::(NOT) EXISTS](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/common/enums/subquery_type.hpp#L21) ：
	- 不检查列数
	- 返回类型是BOOLEAN
	- 例如：`WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)`
- ANY/ALL子查询 [SubqueryType::ANY](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/common/enums/subquery_type.hpp#L23)
	- 需要处理比较操作
	- 需要类型兼容性检查
	- 例如：`WHERE x > ANY (SELECT amount FROM t2)`

然后再针对我们的子查询的处理流程, 大体上分这么几个部分
1. 初始绑定阶段：
```c++
// 创建子查询绑定器
auto subquery_binder = Binder::CreateBinder(context, &binder);
subquery_binder->can_contain_nulls = true;
// 绑定子查询节点，这里会：
// - 绑定FROM t2
// - 绑定WHERE t2.id = t1.id（发现t1.id是相关引用）
// - 绑定MAX(x)
auto bound_node = subquery_binder->BindNode(*expr.subquery->node);

// 处理相关列t1.id
for (idx_t i = 0; i < subquery_binder->correlated_columns.size(); i++) {
    CorrelatedColumnInfo corr = subquery_binder->correlated_columns[i];
    // ...
}
```
2. 类型检查阶段：
```c++
// 因为是标量子查询（SubqueryType::SCALAR），检查必须只返回一列
if (expr.subquery_type != SubqueryType::EXISTS) {
    idx_t expected_columns = 1;
    if (bound_subquery.bound_node->types.size() != expected_columns) {
        throw BinderException(expr, "Subquery returns %zu columns - expected %d",
                            bound_subquery.bound_node->types.size(), expected_columns);
    }
}
```
3. 结果构建阶段：
```c++
// 标量子查询，返回类型是MAX(x)的类型
LogicalType return_type = bound_node->types[0];
auto result = make_uniq<BoundSubqueryExpression>(return_type);
```
大体上是这样
##### 聚合函数
```sql
SELECT a,                                    
       (SELECT MAX(x) FROM t2
        WHERE t2.id = t1.id) as sub_max,
       COUNT(*) as cnt                      -- 聚合函数
FROM t1
WHERE a > 10
GROUP BY a
HAVING COUNT(*) > 1
ORDER BY a DESC;
```

对于`COUNT(*) as cnt`：
- 聚合函数的绑定
- 别名的处理
- 与`GROUP BY`的交互

经典 [BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder.cpp#L59), 之后, 会走进以 `FunctionExpression` 为参数的 [BindExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/expression/bind_function_expression.cpp#L37) 的分支
```c++ title:"src/planner/expression_binder.cpp"
BindResult ExpressionBinder::BindExpression(unique_ptr<ParsedExpression> &expr_ptr, idx_t depth) {
    switch (expr_ptr->GetExpressionClass()) {
    case ExpressionClass::FUNCTION: {
		auto &function = expr_ref.Cast<FunctionExpression>();
		if (IsUnnestFunction(function.function_name)) {
			// special case, not in catalog
			return BindUnnest(function, depth, root_expression);
		}
		// binding a function expression requires an extra parameter for macros
		return BindExpression(function, depth, expr);
    }
    // ...
    }
}
```

我们来看一下这个函数的大体的结构, 它会比子查询的逻辑要简单一些
```c++ title:"src/planner/binder/expression/bind_function_expression.cpp"
BindResult ExpressionBinder::BindExpression(FunctionExpression &function, idx_t depth,
                                          unique_ptr<ParsedExpression> &expr_ptr) {
    // 1. 函数查找阶段
    // 首先尝试在catalog中查找标量函数
    auto func = GetCatalogEntry(CatalogType::SCALAR_FUNCTION_ENTRY, 
                               function.catalog, function.schema,
                               function.function_name, 
                               OnEntryNotFound::RETURN_NULL, 
                               error_context);
    
    // 2. 函数不存在时的处理阶段
    if (!func) {
        // 检查是否是表函数
        auto table_func = GetCatalogEntry(/*...*/);
        if (table_func) {
            // 表函数错误处理
            throw BinderException(/*...*/);
        }

        // 尝试将schema转换为列引用
        // 例如：x.lower() 转换为 lower(x)
        if (!function.schema.empty()) {
            // ... 转换逻辑
        }

        // 重新尝试绑定函数
        func = GetCatalogEntry(/*...*/);
    }

    // 3. 函数修饰符检查阶段
    // 检查DISTINCT、FILTER、ORDER BY是否只用于聚合函数
    if (func->type != CatalogType::AGGREGATE_FUNCTION_ENTRY &&
        (function.distinct || function.filter || !function.order_bys->orders.empty())) {
        throw InvalidInputException(/*...*/);
    }

    // 4. 函数类型分发阶段
    switch (func->type) {
    case CatalogType::SCALAR_FUNCTION_ENTRY:
        // 标量函数处理
        if (function.IsLambdaFunction()) {
            return TryBindLambdaOrJson(/*...*/);
        }
        return BindFunction(/*...*/);
    
    case CatalogType::MACRO_ENTRY:
        // 宏函数处理
        return BindMacro(/*...*/);
    
    default:
        // 聚合函数处理
        return BindAggregate(function, func->Cast<AggregateFunctionCatalogEntry>(), depth);
    }
}
```
对于我们的`COUNT(*) as cnt`：
- 在函数查找阶段：
	- 查找"`COUNT`"函数
	- 识别为聚合函数
- 在函数修饰符检查阶段：
	- 没有`DISTINCT`修饰符
	- 没有`FILTER`子句
	- 没有`ORDER BY`
- 在函数类型分发阶段：
	- 走向聚合函数处理分支
	- 调用`BindAggregate`进行处理

[BindAggregate](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/expression/bind_function_expression.cpp#L264) 会走进 [BindUnsupportedExpression](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder.cpp#L380)
```c++
BindResult ExpressionBinder::BindUnsupportedExpression(ParsedExpression &expr, idx_t depth, const string &message) {
	// we always prefer to throw an error if it occurs in a child expression
	// since that error might be more descriptive
	// bind all children
	ErrorData result;
	ParsedExpressionIterator::EnumerateChildren(
	    expr, [&](unique_ptr<ParsedExpression> &child) { BindChild(child, depth, result); });
	if (result.HasError()) {
		return BindResult(std::move(result));
	}
	return BindResult(BinderException::Unsupported(expr, message));
}
```

而 [EnumerateChildren](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/parser/parsed_expression_iterator.cpp#L29) 是一个表达式树遍历器，它的作用是
- 给定一个表达式（ParsedExpression），遍历它的所有子表达式
- 对每个子表达式执行一个回调函数（callback）

```c++ title:"src/parser/parsed_expression_iterator.cpp"
void ParsedExpressionIterator::EnumerateChildren(
    ParsedExpression &expr, const std::function<void(unique_ptr<ParsedExpression> &child)> &callback) {
	switch (expr.GetExpressionClass()) {
	case ExpressionClass::BETWEEN: {
		auto &cast_expr = expr.Cast<BetweenExpression>();
		callback(cast_expr.input);
		callback(cast_expr.lower);
		callback(cast_expr.upper);
		break;
	}
	case ExpressionClass::CASE: {
		auto &case_expr = expr.Cast<CaseExpression>();
		for (auto &check : case_expr.case_checks) {
			callback(check.when_expr);
			callback(check.then_expr);
		}
		callback(case_expr.else_expr);
		break;
	}
	...
	case ExpressionClass::FUNCTION: {
	    auto &func_expr = expr.Cast<FunctionExpression>();
	    // 遍历函数参数
	    for (auto &child : func_expr.children) {
	        callback(child);
	    }
	    // 处理FILTER子句（如果有）
	    if (func_expr.filter) {
	        callback(func_expr.filter);
	    }
	    // 处理ORDER BY子句（如果有）
	    if (func_expr.order_bys) {
	        for (auto &order : func_expr.order_bys->orders) {
	            callback(order.expression);
	        }
	    }
	    break;
	}
}
```
比如对于不同类型的表达式：
- BETWEEN表达式：遍历input、lower、upper三个子表达式
- CASE表达式：遍历所有的when_expr、then_expr和else_expr
- 函数表达式：遍历所有参数、FILTER子句、ORDER BY子句

这个遍历器的主要用途是：
- 在需要处理表达式树的时候，不用每种表达式都写遍历代码
- 提供统一的遍历接口，让调用者只需要关注对子表达式的处理


##### 小结

总结一下SELECT列表的三个部分：

1. 列引用：
	- 因为有 `GROUP BY` ，需要特殊检查
	- 在`bind_select_node.cpp`中检查非聚合列必须在GROUP BY中
	- 如果列不在 `GROUP BY` 中且不是聚合函数，会抛出错误
2. 相关子查询:
	- 在bind_function_expression. cpp中处理
	- 是一个标量子查询（必须返回单个值）
	- 包含相关引用（t1.id）
	- 处理相关列的深度和类型转换
3. 聚合函数
	- 是一个函数表达式（FunctionExpression）
	- 在 parser 阶段识别为聚合函数
	- 可以包含参数、FILTER、ORDER BY 等子句
	- 与 GROUP BY 交互，影响其他列的检查

#### OREDE BY

这个部分是在 [BindSelectNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder/query_node/bind_select_node.cpp#L406) 的最后部分, 通过调用 `BindModifiers` 来实现的
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
	...
	// now that the SELECT list is bound, we set the types of DISTINCT/ORDER BY expressions
	BindModifiers(*result, result->projection_index, result->names, internal_sql_types, bind_state);
	return std::move(result);
}
```
这个函数的主要作用是：
- 在`SELECT`列表绑定完成后，处理查询的修饰符
- 包括`DISTINCT`、`ORDER BY`、`LIMIT`等
- 对于`ORDER BY`，会为每个排序表达式调用`BindOrderExpression`
```c++ title:"src/planner/binder/query_node/bind_select_node.cpp"
void Binder::BindModifiers(BoundQueryNode &result, idx_t table_index, const vector<string> &names,
                          const vector<LogicalType> &sql_types, const SelectBindState &bind_state) {
    // 1. 处理DISTINCT修饰符
    if (result.modifiers.size() > 0 && result.modifiers[0]->type == ResultModifierType::DISTINCT_MODIFIER) {
        // ... DISTINCT处理
    }

    // 2. 处理ORDER BY修饰符
    for (auto &mod : result.modifiers) {
        if (mod->type == ResultModifierType::ORDER_MODIFIER) {
            auto &order_modifier = mod->Cast<BoundOrderModifier>();
            for (auto &order : order_modifier.orders) {
                // 绑定ORDER BY表达式
                auto bound_expr = BindOrderExpression(order, names, sql_types, table_index, bind_state);
                if (bound_expr) {
                    order.expression = std::move(bound_expr);
                }
            }
        }
    }

    // 3. 处理LIMIT/OFFSET修饰符
    for (auto &mod : result.modifiers) {
        if (mod->type == ResultModifierType::LIMIT_MODIFIER) {
            // ... LIMIT处理
        }
    }
}
```

对于我们的SQL中的`ORDER BY a DESC`：
1. 会被识别为ORDER修饰符
2. 表达式a会通过BindOrderExpression绑定
3. DESC会作为排序方向保存在order中

最终会调用到 [OrderBinder::Bind](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/expression_binder/order_binder.cpp#L120), 对于我们的情况
```c++ title:src/planner/expression_binder/order_binder.cpp
// 会根据表达式类型进行处理：
switch (expr->GetExpressionClass()) {
case ExpressionClass::COLUMN_REF: {
    // 检查是否可以绑定到SELECT列表的别名
    auto index = TryGetProjectionReference(*expr);
    if (index.IsValid()) {
        return CreateProjectionReference(*expr, index.GetIndex());
    }
    break;
}
// ...

// 如果不是简单的引用，会进行更一般的处理：
// 限定列名
for (auto &binder : binders) {
    ExpressionBinder::QualifyColumnNames(binder.get(), expr);
}

// 检查是否已经在投影列表中
auto entry = bind_state.projection_map.find(*expr);
if (entry != bind_state.projection_map.end()) {
    // 找到匹配项，直接引用
    return CreateProjectionReference(*expr, entry->second);
}

// 如果找不到匹配
if (!extra_list) {
    throw BinderException("Could not ORDER BY column \"%s\": add the expression/function to every SELECT, or move "
                         "the UNION into a FROM clause.", expr->ToString());
}
// 将ORDER BY表达式添加到SELECT列表
return CreateExtraReference(std::move(expr));
```
这个绑定过程的关键点是：
- 优先尝试将ORDER BY表达式匹配到SELECT列表中的项
- 如果找不到匹配，且允许的话，会将表达式添加到SELECT列表
	- 不然投影之后就访问不到了捏
- 最终返回一个指向SELECT列表的引用

## 小结

通过这个例子, 逐渐地了解以下的流程
- 基础绑定流程
	- `Binder` 如何初始化
	- 绑定上下文的创建和管理
	- 简单 `SELECT` 的绑定过程
- 表达式绑定
	- 列引用的解析
	- 函数调用的绑定
	- 类型推导机制
- 子查询处理
	- 作用域管理
	- 相关子查询的处理
	- 类型检查和验证
- 其他
	- 聚合函数处理
	- `GROUP BY` 和 `HAVING`
	- `ORDER BY` 绑定

总结一下, 上面的整体流程会
- 为 t1 表创建一个唯一的表索引
- 收集 t1 表的所有列信息（包括 a 列和 id 列）
- 创建一个 LogicalGet 算子来扫描 t1 表
- 在 bind_context 中注册 t1 表的信息，这对后续绑定很重要：
	- WHERE 子句中的 a > 10 需要知道 a 列来自 t1
	- 子查询中的 t1. id 需要能找到 t1 表的 id 列
	- GROUP BY 中的 a 列也需要能找到 t1 表

这样，后续的绑定过程就能通过 bind_context 找到所有需要的列信息了, 这也是为什么在 `select` 之前要先处理 `From` 的绑定

```
TableFunction (抽象接口)
    │
    ├── 绑定阶段 (bind)
    │   └── 返回 FunctionData
    │
    ├── 初始化阶段
    │   ├── init_global -> GlobalTableFunctionState
    │   └── init_local  -> LocalTableFunctionState
    │
    └── 执行阶段 (function)
        └── 使用上述所有状态进行实际的数据扫描
```

Binder 的最终输出结果是一个与 Parse 差不多对应的 [BoundSelectNode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/query_node/bound_select_node.hpp#L37), 还保留着SQL语句的结构特征
```c++ title:src/include/duckdb/planner/query_node/bound_select_node.hpp
class BoundSelectNode : public BoundQueryNode {
    unique_ptr<BoundTableRef> from_table;        // FROM子句
    unique_ptr<Expression> where_clause;         // WHERE子句
    vector<unique_ptr<Expression>> select_list;  // SELECT列表
    vector<unique_ptr<Expression>> groups;       // GROUP BY
    unique_ptr<Expression> having;               // HAVING
    unique_ptr<Expression> qualify;              // QUALIFY
    vector<unique_ptr<BoundResultModifier>> modifiers; // ORDER BY, LIMIT等
    ...
}
```

需要逻辑计划的生成器将其转换为基于关系代数的操作树

### 挂一漏万


1. CTE (Common Table Expression) 处理：
```c++ title:"src/planner/bind_context.cpp"
void BindContext::AddCTEBinding(...) {
    cte_bindings[alias] = binding;
    cte_references[alias] = make_shared_ptr<idx_t>(0); // 引用计数
}
```
- WITH子句的优化
	- [OptimizeCTEs](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/binder.cpp#L253)
- CTE的物化策略
- CTE的递归处理


2. 扩展机制
```c++ title:"src/include/duckdb/planner/operator_extension.hpp"
class OperatorExtension;
class BindContext {
    vector<reference<OperatorExtension>> operator_extensions;
};
```
- [OperatorExtension](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/operator_extension.hpp#L29) 支持自定义操作符扩展
- 支持自定义函数绑定
- 支持宏扩展

3. 特殊绑定类型 : [BindingMode](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/include/duckdb/planner/binder.hpp#L60)
```c++ title:""
enum class BindingMode {
    STANDARD_BINDING,         // 标准绑定
    EXTRACT_NAMES,           // 仅提取名称
    BIND_ALIAS,             // 绑定别名
};
```

4. Lambda和宏处理：
```c++ title:""
optional_ptr<DummyBinding> macro_binding;
optional_ptr<vector<DummyBinding>> lambda_bindings;
```
- 支持Lambda表达式
- 支持宏展开和替换

5. 参数化查询：[BoundParameterMap](https://github.com/jensenojs/duckdb/blob/8458a56aa64d5a222ea5062ce6987c243e46bb7a/src/planner/bound_parameter_map.cpp#L7)
```c++ title:""
optional_ptr<BoundParameterMap> parameters;
```
- 这会设计 prepare 语句的支持

6. USING和自然连接处理
```c++ title:""
unordered_map<string, set<reference<UsingColumnSet>>> using_columns;
```
- 复杂的列名解析
- 多表连接的列歧义处理

但不会涉及：
- CTE优化
- Lambda表达式
- 宏处理
- 参数化查询
- USING/自然连接


# before we end

我们在一开始做了一个[[实践/examples/Database/duckdb/duckdb-前端#调用链路青春版|调用链路青春版]]的演示, 在梳理玩两大板块之后, 这里给出调用链路普通版本

```
ClientContext::Query(query)
    -> ParseStatements(query)  // Parser阶段
        -> Parser::ParseQuery
            -> PostgresParser::Parse  // 实际的解析过程
            -> 生成 SQLStatement

    -> PendingQueryInternal(statement)
        -> PendingStatementOrPreparedStatementInternal
            -> BeginQueryInternal  // 开始查询，设置活动查询上下文
            -> PendingStatementInternal
                -> CreatePreparedStatement
                    -> Planner::CreatePlan(statement)
                        -> Binder::Bind(statement)  // Binder阶段
                            -> BindStatement
                                -> 根据语句类型调用不同的Bind*
                                -> 例如: BindSelect
                                    -> BindSelectNode
                                        -> 绑定FROM子句
                                        -> 绑定WHERE子句
                                        -> 绑定GROUP BY
                                        -> 绑定HAVING
                                        -> 绑定SELECT列表
                                    -> 生成BoundSelectNode
                            -> 返回BoundStatement
                        
                        -> Planner根据BoundStatement生成逻辑计划
                            -> CreatePlan(BoundStatement)
                                -> 根据语句类型生成不同的LogicalOperator
                                -> 例如：CreatePlan(BoundSelectNode)
                                    -> 生成LogicalGet/LogicalProjection等算子
                                    -> 构建逻辑算子树
                    
                    -> Optimizer::Optimize  // 优化阶段
                        -> 应用各种优化规则
                        -> 例如：谓词下推、列裁剪等
                        -> 返回优化后的逻辑计划
                    
                    -> PhysicalPlanGenerator::CreatePlan  // 物理计划生成
                        -> 将逻辑算子转换为物理算子
                        -> 例如：LogicalGet -> PhysicalTableScan
                        -> 返回可执行的物理计划

                -> PendingPreparedStatementInternal
                    -> 创建Executor
                    -> 设置结果收集器
                    
    -> ExecutePendingQueryInternal  // 执行阶段
        -> Executor::ExecuteTask
            -> 执行物理计划
            -> 收集查询结果
```
1. Parser阶段：
	- 如何从SQL文本到SQLStatement的转换
	- 使用PostgresParser进行实际解析
2. Binder阶段：
	- 详细的绑定过程
	- 各种子句的绑定顺序
	- 最终生成BoundStatement
3.  Planner阶段：
	- 如何从BoundStatement生成逻辑计划
	- 逻辑算子树的构建过程
4. 优化器阶段：
	- 优化规则的应用
	- 逻辑计划的转换
5. 物理计划生成：
	- 逻辑算子到物理算子的转换
	- 可执行计划的生成
6. 执行阶段：
	- Executor的创建和初始化
	- 查询执行和结果收集

这样的调用链路更好地展示了DuckDB查询处理的完整流程，以及各个组件之间的交互关系, 同时也是一份能够查询其他遗漏内容的小地图

