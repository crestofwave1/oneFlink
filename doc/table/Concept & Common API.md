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
 - 注册UDF函数（user-defined function)，例如 scalar, table, or aggregation
 - 将DataStream或者DataSet转换为表
 - 保持ExecutionEnvironment或者StreamExecutionEnvironment的引用指向
 
一个表总是与一个特定的TableEnvironment绑定在一块，在同一个查询中(例如join或者是union)不可能将表跟多个TableEnvironment结合到一起。




 

