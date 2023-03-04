# PG内核


### 第一篇送给我从事的工作

# Postgresql Kernel

## Query Parse

根据用户的查询语句生成数据库中最优执行计划。

### Inroduction

| 命令类型                       | 模块                                     |      |
| ------------------------------ | ---------------------------------------- | ---- |
| 简单命令(建表，创建用户，备份) | 功能性命令处理                           |      |
| SELECT/INSERT/DELETE/UPDATE    | 查询树->查询重写->生成最优路径->执行计划 |      |

查询优化主要处理生成路径和生成计划。查询执行时，表连接开销最大。

| 阶段     | 模块         |          | 功能说明                                     | 源码路径                   |
| -------- | ------------ | -------- | -------------------------------------------- | -------------------------- |
| 查询编译 | 查询分析     |          | 由SQL查询语句生成查询树                      | src/backend/parser         |
|          | 查询重写     |          | 对查询书重写并生成新的查询树以支持规则和视图 | src/backend/rewrite        |
|          | 查询优化     | 生成路径 | 由查询树计算最优路径                         | src/backend/optimizer/path |
|          |              | 生成计划 | 通过最优路径生成计划                         | src/backend/optimizer/plan |
| 查询执行 | 执行计划     |          | 执行生成的计划                               | src/backend/executor       |
|          | 调度         |          | 将请求分配到合适的处理模块                   | src/backend/tcop           |
|          | 附件命令处理 |          | 处理建表,备份等命令                          | src/backend/commands       |

### Query Analyze

用户输入的SQL命令------>词法分析，语法分析---->分析树---->语义分析------->查询树

| 输入    | 功能模块 | 函数                         | 输出   | 备注                         |
| ------- | -------- | ---------------------------- | ------ | ---------------------------- |
|         |          | exec_simple_query            |        |                              |
| SQL命令 | 词法分析 | pg_parse_query/pg_raw_parser |        | 可在此处根据兼容性要求，     |
|         | 语法分析 | pg_parse_query/pg_raw_parser | 分析树 | 选择封装一层选择合适的解析器 |
|         | 语义分析 | pg_analyze_and_rewrite       |        |                              |
|         |          | parse_analyzer               | 查询树 |                              |
|         |          |                              |        |                              |

#### Lex and Yacc

| 工具名称 | 功能                                                         |
| -------- | ------------------------------------------------------------ |
| Lex      | 生成扫描器，其能识别模式如数字、字符串、特殊符号。           |
| Yacc     | 生成语法分析器，识别模式的组合，即语法。                     |
| 共同     | 生成用于词法分析和语法分析的源代码，使用符号表，通过yylval传递表示的值。 |

##### Lex

通过正则表示式进行模式识别。

**Lex代码文件.l的基本结构**

| 代码段 | 功能                                                         |
| ------ | ------------------------------------------------------------ |
| 定义   | 包含C语言头文件，符号说明，其代码会被拷贝到生成的扫描器代码中 |
| 规则   | 利用正则表达式来匹配模式，每当成功匹配一个模式，就对应气候的{}中的代码，这些匹配规则及动作都会反映到扫描器代码中 |
| 代码   | 可以是任意C代码，但是其中必须调用Lex提供的函数               |
|        |                                                              |

其中各段由%%分隔

**Lex函数**

| 函数   | 作用                                                         |
| ------ | ------------------------------------------------------------ |
| yylex  | 开始分析工作。由lex自动生成。                                |
| yywrap | 在文件末尾调用。方法是使用yyin文件指针指向不同的文件。yywrap返回1表示解析结束 |
| yyless | 送回除了前n个字符外的所有读出标记                            |
| yymore | 告诉Lexer将下一个标记附加到当前标记后                        |

**Lex变量**

| 变量     | 作用                              |
| -------- | --------------------------------- |
| yyin     | FILE *类型。lexer正在解析的文件。 |
| yyout    | FILE* 类型。指向lexer输出的位置。 |
| yytext   |                                   |
| yyleng   |                                   |
| yylineno |                                   |



##### Yacc

#### Lexical and Syntactic Analysis

```sql
SELECT classno,classname, AVG(score) AS avg_score
FROM sc, (SELECT * from class where class.gno = '2005') AS sub
where sc.sno IN (select sno FROM student FROM student WHRER student.classno = sub.classno) AND sc.cno IN (SELECT course.cno FROM course WHERE course.cname = '高等数学')
GROUP BY classno, classname
HAVING AVG(score) > 80.0
ORDER BY avg_score;
```

Example Statemet

##### DISTINCT Clause

##### TARGET ATTRIBUTE

##### FROM CLAUSE

##### WHERE CLAUSE

##### GROUP BY CLAUSE

##### HAVING CLAUSE AND ORDER BY CLAUSE

#### Semantic Analysis

| 函数                   | 功能                                   | 层级 | 输入          | 输出       |
| ---------------------- | -------------------------------------- | ---- | :------------ | ---------- |
| exec_simple_query      | 对每颗分析树调用pg_analyze_and_rewrite | 1    | parsetreelist |            |
| pg_analyze_and_rewrite | 语义分析和查询重写                     | 2    | 单棵分析数    | 查询树链表 |
| parse_analyze          | 语义分析                               | 3    | 单棵分析数    | 单棵查询树 |
| pg_rewrite_query       | 查询重写                               | 3    | 单棵查询树    | 查询树链表 |

**Query Struct**

| Type        | name           | comment                                                      |
| ----------- | -------------- | ------------------------------------------------------------ |
| NodeTag     | type           | 节点类型，T_Query                                            |
| CmdType     | commandType    | 命令类型                                                     |
| QuerySource | querySource    | 是原始查询还是来自于规则的查询                               |
| bool        | canSetTag      | 查询重写时用到，如果该Query是由原始查询转换为来则此字段为假，如果Query是由查询重写或查询规划时新增加的则此字段为真 |
| Node*       | utilityStmt    | 定义游标或者不可优化的查询语句                               |
| int         | resultRelation | 结果关系                                                     |
| IntoClause* | intoClause     | 处理SELECT INTO和CREATE TABLE AS的信息                       |
| bool        | hasAggs        |                                                              |
| bool        | hasWindowFuncs |                                                              |
| bool        | hasSubLinks    |                                                              |
| bool        | hasDistincton  |                                                              |
| bool        | hasRecursive   |                                                              |
| List*       | cteList        |                                                              |
| List*       | rtable         |                                                              |
| FromExpr*   | jointree       | 连接树，描述FROM和WHERE子句出现的连接                        |
| List*       | targetList     |                                                              |
| List*       | returningList  |                                                              |
| List*       | groupClause    |                                                              |
| Node*       | havingQual     |                                                              |
| List*       | windowClause   |                                                              |
| List*       | sortClause     |                                                              |
| List*       | distinctClause |                                                              |
| List*       | sortClause     |                                                              |
| List*       | limitOffset    |                                                              |
| List*       | limitCount     |                                                              |
| List*       | rowMarks       |                                                              |
| Node*       | setOperations  |                                                              |

## Query Execution



### Plan Node

#### Scan Node

##### SeqScanNode
