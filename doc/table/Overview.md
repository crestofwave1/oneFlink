
# Table API和SQL

Apache Flink 有两个关系API--Table API 和 SQL--统一流和批量处理。
Table API是由Scala和Java集成的查询API，它以一种非常直观的方式组合来实现关系操作（例如选择、筛选和链接）。Flink's SQL是基于实现了SQL标准的Apache Calcite实现的。
无论输入是batch input或者stream input，查询任何借口都会有相同的语义和相同的结果。

Table API和SQL接口结合就像Flink DataStream和DataSet API一样紧密在一起。你可以基于这些API，很容易的替换所有的API和库。例如，你可以使用CEP库以流的形式抽取，然后
使用Table API 来分析模式，或者你可能在运行Gelly graph算法预处理数据时用SQL查询来扫描、过滤、聚合批量table。


请注意Table API和SQL特性还不完全，处于积极开发中。不是所有操作都支持Table API、SQL和Stream、batch输入。


## Setup设置
Table API和SQL捆绑在flink-table Maven artifact。为了使用table API和SQL，你必须在你的工程中添加下面依赖。
```java
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table_2.11</artifactId>
  <version>1.6.0</version>
</dependency>

另外，你需要添加Flink’s Scala batch or streaming API的依赖。支持批查询你需要添加：

<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala_2.11</artifactId>
  <version>1.6.0</version>
</dependency>

流查询你需要添加：

<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala_2.11</artifactId>
  <version>1.6.0</version>
</dependency>
```
注意：由于Apache Calcite的一个问题，它组织用户加载器被垃圾收集，我们不建议构建一个包含flink-table以来的fat-jar。替代的，我们建议
在系统类加载器中配置Flink来包括flink-table依赖。这能通过复制flink-table.jar从./opt文件到./lib文件来实现。请参照后面的详细说明。


回到顶部
接下来
概念&通用API：Table API的API集与SQL的共享概念
Stream Table API和SQL:Table Api特定流文档或SQL，例如配置时间属性和更新结果。
Table API:Table API支持的API操作
SQL:SQL支持的操作和语法
Table Sources&Sinks：读表并将表发到外部存储系统
UDF:用户定义函数的定义和使用






