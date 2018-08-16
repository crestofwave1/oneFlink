## Concept  & Common API
Table API和SQL集成在一个联合的API中。这个API核心概念是表作为查询的输入和输出。这篇文章展示了使用Table API和SQL查询的程序结构，如何去进行表的创建，如何去进行表的查询，并且展示如何去进行表的输出。



## 1. Structure of Table API and SQL Programs

​	所有使用批量和流式相关的Table API和SQL的程序都遵循以下模式。下面的代码实例展示了Table API和SQL程序的结构。

```
// 在批处理程序中使用ExecutionEnvironment代替StreamExecutionEnvironment
val env = StreamExecutionEnvironment.getExecutionEnvironment

// 创建TableEnvironment对象
val tableEnv = TableEnvironment.getTableEnvironment(env)

// 注册表
tableEnv.registerTable("table1", ...)           // or
tableEnv.registerTableSource("table2", ...)     // or
tableEnv.registerExternalCatalog("extCat", ...) 

// 基于Table API的查询创建表
val tapiResult = tableEnv.scan("table1").select(...)
// 从SQL查询创建表
val sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table2 ...")

// 将表操作API查询到的结果表输出到TableSink，SQL查询到的结果一样如此
tapiResult.writeToSink(...)

// 执行
env.execute()
```

注意：Table API和SQL查询很容易被整合到DataStream或者DataSet程序中。查看[将DataStream和DataSet API进行整合](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/common.html#integration-with-datastream-and-dataset-api)章节学习DataSteams和DataSets是如何转换成表或为什么不能转换成表。



## 2. Create a TableEnvironment
TableEnvironment是Table API与SQL整合的核心概念之一，它主要有如下功能：
 - 向internal catalog注册表
 - 注册external catalog
 - 执行SQL查询
 - 注册UDF函数（user-defined function)，例如 scalar, table或aggregation
 - 将DataStream或者DataSet转换为表
 - 保持ExecutionEnvironment或者StreamExecutionEnvironment的引用指向
 
一个表总是与一个特定的TableEnvironment绑定在一块，在同一个查询中(例如join或者是union)不可能将一个表跟多个TableEnvironment结合到一起。

创建TableEnvironment的方法通常是通过StreamExecutionEnvironment，ExecutionEnvironment对象调用其中的静态方法TableEnvironment.getTableEnvironment()，或者是TableConfig来创建。
TableConfig可以用作配置TableEnvironment或是对自定义查询或者是编译过程进行优化(详情查看[查询优化](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/common.html#query-optimization))

```
// ***************
// 流式查询
// ***************
val sEnv = StreamExecutionEnvironment.getExecutionEnvironment
// 为流式查询创建一个TableEnvironment对象
val sTableEnv = TableEnvironment.getTableEnvironment(sEnv)

// ***********
// 批量查询
// ***********
val bEnv = ExecutionEnvironment.getExecutionEnvironment
// 为批量查询创建一个TableEnvironment对象
val bTableEnv = TableEnvironment.getTableEnvironment(bEnv)
```
## Register Tables in the Catalog
TableEnvironment包含了通过名称注册表时的表的catalog信息。通常情况下有两种表，一种为输入表，
一种为输出表。输入表主要是在使用Table API和SQL查询时提供输入数据，输出表主要是将Table API和
SQL查询的结果作为输出结果对接到外部系统。

输入表有多种不同的输入源进行注册：
- 已经存在的Table对象，通常是是作为Table API和SQL查询的结果
- TableSource，主要是从外部输入数据，例如文件，数据库或者是消息系统
- 来自DataStream或是DataSet系统中的DataStream或DataSet，讨论DataStream或是DataSet
可以[整合DataStream和DataSet API](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/common.html#integration-with-datastream-and-dataset-api)了解到

输出表可使用TableSink进行注册

## Register a Table
Table是如何注册到TableEnvironment中如下所示：
```
// 获取(创建)TableEnvironment对象
val tableEnv = TableEnvironment.getTableEnvironment(env)

// 从简单的查询结果中作为表
val projTable: Table = tableEnv.scan("X").select(...)

// 将表projTable命名为projectedTable注册到TableEnvironment中
tableEnv.registerTable("projectedTable", projTable)
```
注意：一张注册表就跟关系型数据库中的视图性质相同，定义表的查询未进行优化，但在另一个查询引用已注册的表时将进行内联。
如果多表查询引用了相同的注册表，它就会将每一个引用进行内联并且查询多次，注册表的结果之间不会进行共享。

## Register a TableSource
TableSource提供对外部数据的访问，外部系统存储例如数据库（Mysql,HBase），特殊格式编码的文件(CSV, Apache [Parquet, Avro, ORC], …)
或者是消息系统 (Apache Kafka, RabbitMQ, …)

Flink旨在为常见的数据格式和存储系统提供TableSource。请查看[此处](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/sourceSinks.html)
了解支持的TableSource类型并且了解到如何去自定义TableSour的指导。

TableSource是如何注册到TableEnvironment中如下所示：
```
// 获取TableEnvironment对象
val tableEnv = TableEnvironment.getTableEnvironment(env)

// 创建TableSource对象
val csvSource: TableSource = new CsvTableSource("/path/to/file", ...)

// 将创建的TableSource作为表并命名为csvTable注册到TableEnvironment中
tableEnv.registerTableSource("CsvTable", csvSource)
```