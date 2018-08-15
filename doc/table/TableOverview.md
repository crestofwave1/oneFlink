# 表相关概述
## 1. Table API & SQL

为了统一流式处理和批处理，Apache Flink提供两种API操作，Table API和SQL。Table API是Scala和Java语言集成查询的API，它以一种更直观的方式来组合复杂的关系操作查询，例如选择，过滤和连接。Flink的SQL操作基于实现了SQL标准的[Apache Calcite](https://calcite.apache.org/)。Flink中指定的查询在这两个接口中所使用的语法和取得的结果都是一样的，无论它的输入来自于批量输入(DataSet)或者是流输入(DataStream)



Table API和SQL接口彼此都是紧密结合在一起的，就像Flink中的DataStream和DataSet API一样。你可以在这些API接口上和建立在这些库上的API中任意进行切换。例如，你先可以使用[ CEP library ](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/libs/cep.html)来提取DataStream中的patterns，再使用Table API来分析这些patterns，或者可以用[ Gelly graph algorithm ](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/libs/gelly/)对数据进行预处理之前使用SQL扫描，过滤和聚合多表批处理。



请注意：Table API和SQL功能尚未完成，正处于开发阶段中。目前并不是所有的[Table API，SQL]和[流式，批量]功能的组合都支持所有操作。



## 2. Setup

Table API和SQL目前都绑定在Maven仓库中的flink-table包中。如果你要使用Table API和SQL你必须在你的项目中添加如下依赖：

```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table_2.11</artifactId>
  <version>1.6.0</version>
</dependency>
```

除此之外，若要使用scala批处理或流式处理API，你需要添加依赖。

如使用批量查询的话需要添加如下依赖：

```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala_2.11</artifa想·ctId>
  <version>1.6.0</version>
</dependency
```

如使用流式查询你需要添加如下依赖：

```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala_2.11</artifactId>
  <version>1.6.0</version>
</dependency>
```

注意：由于Apache Calcite存在一个issue，它主要是防止用户类加载器被GC收集，我们不推荐使用包含flink-table的这种臃肿jar包依赖方式。反而我们推荐的是将flink-table依赖配置到系统加载器中。也就是说，你可以将flink-table.jar 从./opt目录下拷贝到./lib目录下。详情请见[ 此处 ](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/linking.html)

