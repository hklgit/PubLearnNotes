
### 1. 概述

Common Table Expression 为公用表表达式，简称 CTE，是一个临时命名结果集。它是为了简化SQL查询，而被标准SQL引入的。

CTE 可以被认为是在单个 SELECT，INSERT，UPDATE，DELETE 或 CREATE VIEW 语句的执行范围内定义的临时结果集。CTE 类似于派生表，因为它不作为对象存储，并且仅在查询期间存在。与派生表不同，CTE可以是自引用的，并且可以在同一查询中多次引用。

Common Table Expression 是 WITH 子句中的简单查询生成的临时结果集，该子句紧接在 SELECT 或 INSERT 关键字之前。CTE 仅在单个语句的执行范围内定义。 可以在 Hive SELECT，INSERT，CREATE TABLE AS SELECT或CREATE VIEW AS SELECT 语句中使用一个或多个 CTE。

在 Hive 0.13.0 版本中添加了 Common Table Expressions，具体细看使[HIVE-1180](https://issues.apache.org/jira/browse/HIVE-1180)。

### 2. 语法

CTE由表示CTE的表达式名称，AS关键字和SELECT语句组成。定义CTE后，可以在SELECT，INSERT，UPDATE或DELETE语句中像表或视图一样引用它。CTE也可以在CREATE VIEW语句中用作其定义SELECT语句的一部分。

CTE的基本语法结构是：
```sql
WITH cte_name AS ( cte_query )
  [,cte_name2  AS ( cte_query2),……]
```
说明:
- cte_name：CTE的名称，不能与当前 WITH 子句中的其他CTE的名称相同。查询中任何使用到 cte_name 标识符的地方，均指CTE。
- cte_query：一个 SELECT 语句。它产生的结果集用于填充CTE。

### 3. Example

```sql
-- App用户学习总时长
SELECT
    SUM(duration) AS duration
FROM
(
    SELECT user_id, duration
    FROM dws_app_study_user_td
    WHERE dt = '${date}'
) AS study
LEFT SEMI JOIN
(
    SELECT user_id
    FROM dim_app_user_td
    WHERE dt = '${date}'
) AS user
ON study.user_id = user.user_id
UNION ALL
-- App用户学习总积分
SELECT
    SUM(score) AS score
FROM
(
    SELECT user_id, score
    FROM dws_app_score_user_td
    WHERE dt = '${date}'
) AS study
LEFT SEMI JOIN
(
    SELECT user_id
    FROM dim_app_user_td
    WHERE dt = '${date}'
) AS user
ON study.user_id = user.user_id;
```
顶层的 UNION 两侧各为一个 JOIN，JOIN 的右表是相同的查询。通过写子查询的方式，只能重复这段代码。可以使用CTE的方式重写以上语句：
```sql
WITH
    user AS (
        SELECT user_id
        FROM dim_app_user_td
        WHERE dt = '${date}'
    ),
    study_duration AS (
        SELECT user_id, duration
        FROM dws_app_study_user_td
        WHERE dt = '${date}'
    ),
    study_score AS (
        SELECT user_id, score
        FROM dws_app_score_user_td
        WHERE dt = '${date}'
    ),
    user_study_duration AS (
        SELECT SUM(a.duration) AS duration
        FROM study_duration AS a
        LEFT SEMI JOIN user AS b
        ON a.user_id = b.user_id
    ),
    user_study_score AS (
        SELECT SUM(a.score) AS score
        FROM study_score AS a
        LEFT SEMI JOIN user AS b
        ON a.user_id = b.user_id
    )
SELECT * FROM user_study_duration
UNION ALL
SELECT * FROM user_study_score;
```
重写后，user 对应的子查询只需写一次，便可在后面进行重用。你可以在 CTE 的 WITH 子句中指定多个子查询，像使用变量一样在整个语句中反复重用。除重用外，不必反复嵌套。

### 4. 使用场景


```sql
with q1 as ( select key from src where key = '5')
select *
from q1;

-- chaining CTEs
with q1 as ( select key from q2 where key = '5'),
q2 as ( select key from src where key = '5')
select * from (select key from q1) a;

-- union example
with q1 as (select * from src where key= '5'),
q2 as (select * from src s2 where key = '4')
select * from q1 union all select * from q2;
```

在视图/创建/插入语句的CTE：
```sql
-- insert example
create table s1 like src;
with q1 as ( select key, value from src where key = '5')
from q1
insert overwrite table s1
select *;

-- ctas example
create table s2 as
with q1 as ( select key from src where key = '4')
select * from q1;

-- view example
create view v1 as
with q1 as ( select key from src where key = '5')
select * from q1;
select * from v1;

-- view example, name collision
create view v1 as
with q1 as ( select key from src where key = '5')
select * from q1;
with q1 as ( select key from src where key = '4')
select * from v1;
```

### 5. 使用CTE的好处

- 提高可读性：使用 CTE 可以提高可读性并简化复杂查询的维护。不是将所有查询逻辑都集中到一个大型查询中，而是创建几个简单逻辑构建块的 CTE，然后可以使用它们构建更复杂的 CTE，直到生成最终结果集。
- 替代视图或表：定义CTE后，可以用作表或视图，并可以 SELECT，INSERT，UPDATE 或 DELETE 数据。如果你没有创建视图或表的权限，或者你根本不想创建一个视图或表，因为它仅在这一个查询中使用，这种情形使用 CTE 很方便。
- 排名：每当你想使用排名函数，如ROW_NUMBER()，RANK()，NTILE()等。

当SQL的逻辑很复杂，子查询嵌套比较多时，SQL的可读性会很差，后期理解和维护困难。这个时候可以使用Common Table Expression（CTE）来简化SQL，提高可读性和执行效率。




参考:[Common Table Expression](https://cwiki.apache.org/confluence/display/Hive/Common+Table+Expression)
[Introduction to common table expressions](http://dcx.sybase.com/1100/en/dbusage_en11/commontblexpr-s-5414852.html)
[WITH common_table_expression (Transact-SQL)](https://docs.microsoft.com/zh-cn/sql/t-sql/queries/with-common-table-expression-transact-sql?view=sql-server-2017#syntax)
[What are the Advantages of using common table expression (CTE)?](http://www.codesolution.org/what-are-the-advantages-of-using-common-table-expression-cte/)
