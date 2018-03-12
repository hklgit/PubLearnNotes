---
layout: post
author: sjf0115
title: Flink1.4 Table与SQL的概念以及常见API
date: 2018-03-12 11:29:01
tags:
  - Flink
  - Flink SQL

categories: Flink
permalink: flink-sql-table-and-sql-concepts-common-api
---

Table API 和 SQL 集成在一个联合API中。这个API的核心概念是一个用作输入和输出查询的表。这篇文章展示了使用 Table API 和 SQL 查询程序的常见结构，如何注册表，如何查询表以及如何发出表。


### 1. Table API 和 SQL 查询程序的常见结构

批处理和流式处理的所有 Table API 和 SQL 程序遵循相同的模式。以下代码示例显示了 Table API 和 SQL 程序的常见结构。

Java版本:
```java
// for batch programs use ExecutionEnvironment instead of StreamExecutionEnvironment
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// create a TableEnvironment
// for batch programs use BatchTableEnvironment instead of StreamTableEnvironment
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register a Table
tableEnv.registerTable("table1", ...)            // or
tableEnv.registerTableSource("table2", ...);     // or
tableEnv.registerExternalCatalog("extCat", ...);

// create a Table from a Table API query
Table tapiResult = tableEnv.scan("table1").select(...);
// create a Table from a SQL query
Table sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table2 ... ");

// emit a Table API result Table to a TableSink, same for SQL result
tapiResult.writeToSink(...);

// execute
env.execute();
```
Scala版本:
```scala
// for batch programs use ExecutionEnvironment instead of StreamExecutionEnvironment
val env = StreamExecutionEnvironment.getExecutionEnvironment

// create a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// register a Table
tableEnv.registerTable("table1", ...)           // or
tableEnv.registerTableSource("table2", ...)     // or
tableEnv.registerExternalCatalog("extCat", ...)

// create a Table from a Table API query
val tapiResult = tableEnv.scan("table1").select(...)
// Create a Table from a SQL query
val sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table2 ...")

// emit a Table API result Table to a TableSink, same for SQL result
tapiResult.writeToSink(...)

// execute
env.execute()
```
> 备注

> Table API 和 SQL 查询可以轻松集成并嵌入到DataStream或DataSet程序中。查看[与DataStream和DataSet API的集成](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/table/common.html#integration-with-datastream-and-dataset-api)部分，了解如何将DataStream和DataSet转换为表格，反之亦然。

### 2. 创建 TableEnvironment

`TableEnvironment` 是 Table API 和 SQL 集成的核心概念。它负责：
- 在内部目录中注册一个 Table
- 注册外部目录
- 执行SQL查询
- 注册一个用户自定义（`scalar`, `table`, 或者 `aggregation`）函数
- 将 `DataStream` 或 `DataSet` 转换为 Table
- 保存对 `ExecutionEnvironment` 或 `StreamExecutionEnvironment` 的引用

Table 总是与特定的 `TableEnvironment` 绑定。无法在同一个查询中组合来自不同 `TableEnvironments` 的表，例如，`join` 或者 `union`。

`TableEnvironment` 通过调用静态方法 `TableEnvironment.getTableEnvironment()` 来创建，参数为 `StreamExecutionEnvironment` 或 `ExecutionEnvironment` 以及可选的 `TableConfig`。 `TableConfig` 可用于配置 `TableEnvironment` 或自定义查询优化和转换过程（请参阅查询优化部分）。

Java版本:
```java
// STREAMING QUERY
StreamExecutionEnvironment sEnv = StreamExecutionEnvironment.getExecutionEnvironment();
// create a TableEnvironment for streaming queries
StreamTableEnvironment sTableEnv = TableEnvironment.getTableEnvironment(sEnv);

// BATCH QUERY
ExecutionEnvironment bEnv = ExecutionEnvironment.getExecutionEnvironment();
// create a TableEnvironment for batch queries
BatchTableEnvironment bTableEnv = TableEnvironment.getTableEnvironment(bEnv);
```
Scala版本:
```scala
// STREAMING QUERY
val sEnv = StreamExecutionEnvironment.getExecutionEnvironment
// create a TableEnvironment for streaming queries
val sTableEnv = TableEnvironment.getTableEnvironment(sEnv)

// BATCH QUERY
val bEnv = ExecutionEnvironment.getExecutionEnvironment
// create a TableEnvironment for batch queries
val bTableEnv = TableEnvironment.getTableEnvironment(bEnv)
```

### 3. 在目录中注册 Table

`TableEnvironment` 维护一个按名称注册的表的目录。有两种类型的表，输入表和输出表。输入表可以在 Table API 和 SQL 查询中引用并提供输入数据。输出表可用于将 Table API 或 SQL 查询的结果发送到外部系统中。

输入表可以从各种数据源注册：
- 一个存在的 Table 对象，通常是 Table API 或 SQL 查询的结果。
- 一个 TableSource，用于访问外部数据，如文件，数据库或消息系统。
- 来自 DataStream 或 DataSet 程序的 DataStream 或 DataSet。

输出表可以使用 TableSink 注册。

#### 3.1 注册 Table

在 TableEnvironment 中注册 Table 如下所示:

Java版本:
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Table is the result of a simple projection query
Table projTable = tableEnv.scan("X").project(...);

// register the Table projTable as table "projectedX"
tableEnv.registerTable("projectedTable", projTable);
```
Scala版本:
```scala
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// Table is the result of a simple projection query
val projTable: Table = tableEnv.scan("X").project(...)

// register the Table projTable as table "projectedX"
tableEnv.registerTable("projectedTable", projTable)
```

> 备注

> 注册表的处理方式与关系数据库系统中我们熟知的 VIEW 类似，即，定义 Table 的查询没有经过优化，但是当另一个查询引用已注册的表时，将会进行内联。如果多个查询引用相同的注册表，那么它将针对每个引用查询进行内联，并执行多次，即不会共享注册表的结果。

#### 3.2 注册 TableSource

TableSource 提供访问存储在存储系统（例如数据库（MySQL，HBase，...）），具有特定编码的文件（CSV，Apache [Parquet，Avro，ORC]，...）或消息系统（Apache Kafka，RabbitMQ，...）中的外部数据的能力。

Flink 目的是为常见数据格式和存储系统提供 TableSources。请查看[Table Sources and Sinks](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/table/sourceSinks.html)以获取支持的 TableSources 列表以及如何构建自定义 TableSource 的说明。

在 TableEnvironment 中注册 TableSource 如下所示:

Java版本:
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create a TableSource
TableSource csvSource = new CsvTableSource("/path/to/file", ...);

// register the TableSource as table "CsvTable"
tableEnv.registerTableSource("CsvTable", csvSource);
```
Scala版本:
```scala
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// create a TableSource
val csvSource: TableSource = new CsvTableSource("/path/to/file", ...)

// register the TableSource as table "CsvTable"
tableEnv.registerTableSource("CsvTable", csvSource)
```

#### 3.3 注册 TableSink

已注册的 TableSink 可用于将 Table API 或 SQL 查询的结果输出到外部存储系统中，例如数据库，key-value存储系统，消息队列或文件系统（以不同的编码方式，例如CSV，Apache [ Parquet，Avro，ORC]，...）。

Flink 目的是为常见的数据格式和存储系统提供 TableSinks。有关可用接收器以及如何实现自定义TableSink的的详细信息，请参阅[Table Sources and Sinks](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/table/sourceSinks.html)。

Java版本:
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create a TableSink
TableSink csvSink = new CsvTableSink("/path/to/file", ...);

// define the field names and types
String[] fieldNames = {"a", "b", "c"};
TypeInformation[] fieldTypes = {Types.INT, Types.STRING, Types.LONG};

// register the TableSink as table "CsvSinkTable"
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, csvSink);
```
Scala版本:
```scala
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// create a TableSink
val csvSink: TableSink = new CsvTableSink("/path/to/file", ...)

// define the field names and types
val fieldNames: Array[String] = Array("a", "b", "c")
val fieldTypes: Array[TypeInformation[_]] = Array(Types.INT, Types.STRING, Types.LONG)

// register the TableSink as table "CsvSinkTable"
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, csvSink)
```

### 4. 注册外部目录

外部目录可以提供有关外部数据库和表的信息，例如它们的名称，schema，统计信息以及有关如何访问存储在外部数据库，表或文件中的数据的信息。

外部目录可以通过实现 ExternalCatalog 接口来创建，并在 TableEnvironment 中注册如下：

Java版本:
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create an external catalog
ExternalCatalog catalog = new InMemoryExternalCatalog();

// register the ExternalCatalog catalog
tableEnv.registerExternalCatalog("InMemCatalog", catalog);
```
Scala版本：
```scala
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// create an external catalog
val catalog: ExternalCatalog = new InMemoryExternalCatalog

// register the ExternalCatalog catalog
tableEnv.registerExternalCatalog("InMemCatalog", catalog)
```
一旦在 TableEnvironment 中注册，可以通过指定其完整路径（如catalog.database.table）从 Table API 或 SQL 查询中访问 ExternalCatalog 中定义的所有表。

目前，Flink 提供用于演示和测试目的的 InMemoryExternalCatalog。 但是，ExternalCatalog 接口也可用于将 HCatalog 或 Metastore 等目录连接到 Table API。

### 5. 查询 Table

#### 5.1 Table API

Table API 是用于 Scala 和 Java 的集成语言查询API。与SQL相比，查询不是以字符串形式指定的，而是以宿主语言逐步编写的。

该 API 基于表示表（流或批处理）的 `Table` 类，并提供了应用关系性操作的方法。这些方法返回一个新的 Table 对象，表示在输入表上应用关系性操作的结果。一些关系性操作由多个方法调用组成，如 table.groupBy（...）.select（），其中 groupBy（...） 指定表的分组， select（...） 指定在分组表上的投影。

[Table API](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/table/tableApi.html) 文档描述了在流和批处理表上支持的所有 Table API 操作。以下示例显示了一个简单的 Table API 聚合查询：

Java版本:
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register Orders table

// scan registered Orders table
Table orders = tableEnv.scan("Orders");
// compute revenue for all customers from France
Table revenue = orders
  .filter("cCountry === 'FRANCE'")
  .groupBy("cID, cName")
  .select("cID, cName, revenue.sum AS revSum");

// emit or convert Table
// execute query
```
Scala版本:
```scala
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// register Orders table

// scan registered Orders table
Table orders = tableEnv.scan("Orders")
// compute revenue for all customers from France
Table revenue = orders
  .filter('cCountry === "FRANCE")
  .groupBy('cID, 'cName)
  .select('cID, 'cName, 'revenue.sum AS 'revSum)

// emit or convert Table
// execute query
```

#### 5.2 SQL

Flink 的 SQL 集成基于 Apache Calcite，其实现了 SQL 标准。SQL查询被指定为常规字符串。[SQL](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/table/sql.html) 文档描述了 Flink 对流和批处理表的 SQL 支持。以下示例显示如何指定查询并将结果作为表返回。

Java版本:
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register Orders table

// compute revenue for all customers from France
Table revenue = tableEnv.sqlQuery(
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// emit or convert Table
// execute query
```

Scala版本：
```scala
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// register Orders table

// compute revenue for all customers from France
Table revenue = tableEnv.sqlQuery("""
  |SELECT cID, cName, SUM(revenue) AS revSum
  |FROM Orders
  |WHERE cCountry = 'FRANCE'
  |GROUP BY cID, cName
  """.stripMargin)

// emit or convert Table
// execute query
```
以下示例展示了如何指定将其结果插入注册表的更新查询：

Java版本:
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register "Orders" table
// register "RevenueFrance" output table

// compute revenue for all customers from France and emit to "RevenueFrance"
tableEnv.sqlUpdate(
    "INSERT INTO RevenueFrance " +
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// execute query
```
Scala版本:
```scala
// get a TableEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// register "Orders" table
// register "RevenueFrance" output table

// compute revenue for all customers from France and emit to "RevenueFrance"
tableEnv.sqlUpdate("""
  |INSERT INTO RevenueFrance
  |SELECT cID, cName, SUM(revenue) AS revSum
  |FROM Orders
  |WHERE cCountry = 'FRANCE'
  |GROUP BY cID, cName
  """.stripMargin)

// execute query
```

#### 5.3 Table API 与 SQL 混合使用‘

Table API 和 SQL 查询可以轻松混合使用，因为它们都返回 Table 对象：
- Table API 查询可以在 SQL 查询返回的 Table 对象上定义。
- SQL 查询可以通过在 TableEnvironment 中注册结果表并在 SQL 查询的 FROM 子句中引用它，在 Table API 查询的结果上定义。

### 6. 输出 Table

### 7. 转换与执行查询

### 8. 与 DataStream 和 DataSet API 集成

### 9. 查询优化









原文: https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/table/common.html#create-a-tableenvironment
