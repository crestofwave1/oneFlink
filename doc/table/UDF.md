
# Flink用户自定义函数

用户自定义函数是非常重要的一个特征，因为他极大地扩展了查询的表达能力。

在大多数场景下，用户自定义函数在使用之前是必须要注册的。对于Scala的Table API，udf是不需要注册的。
调用TableEnvironment的registerFunction()方法来实现注册。Udf注册成功之后，会被插入TableEnvironment的function catalog，这样table API和sql就能解析他了。
本文会主要讲三种udf：
* ScalarFunction
* TableFunction
* AggregateFunction

## 1. Scalar Functions 标量函数

标量函数，是指指返回一个值的函数。标量函数是实现讲0，1，或者多个标量值转化为一个新值。

实现一个标量函数需要继承ScalarFunction，并且实现一个或者多个evaluation方法。标量函数的行为就是通过evaluation方法来实现的。evaluation方法必须定义为public，命名为eval。evaluation方法的输入参数类型和返回值类型决定着标量函数的输入参数类型和返回值类型。evaluation方法也可以被重载实现多个eval。同时evaluation方法支持变参数，例如：eval(String... strs)。

下面给出一个标量函数的例子。例子实现的事一个hashcode方法。
```java
public class HashCode extends ScalarFunction {
  private int factor = 12;
  
  public HashCode(int factor) {
      this.factor = factor;
  }
  
  public int eval(String s) {
      return s.hashCode() * factor;
  }
}

BatchTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register the function
tableEnv.registerFunction("hashCode", new HashCode(10));

// use the function in Java Table API
myTable.select("string, string.hashCode(), hashCode(string)");

// use the function in SQL API
tableEnv.sqlQuery("SELECT string, HASHCODE(string) FROM MyTable");

````

默认情况下evaluation方法的返回值类型是由flink类型抽取工具决定。对于基础类型，简单的POJOS是足够的，但是更复杂的类型，自定义类型，组合类型，会报错。这种情况下，返回值类型的TypeInformation，需要手动指定，方法是重载
ScalarFunction#getResultType()。

下面给一个例子，通过复写ScalarFunction#getResultType()，将long型的返回值在代码生成的时候翻译成Types.TIMESTAMP。

```java
public static class TimestampModifier extends ScalarFunction {
  public long eval(long t) {
    return t % 1000;
  }

  public TypeInformation<?> getResultType(signature: Class<?>[]) {
    return Types.TIMESTAMP;
  }
}
```

## 2. Table Functions 表函数

与标量函数相似之处是输入可以0，1，或者多个参数，但是不同之处可以输出任意数目的行数。返回的行也可以包含一个或者多个列。

为了自定义表函数，需要继承TableFunction，实现一个或者多个evaluation方法。表函数的行为定义在这些evaluation方法内部，函数名为eval并且必须是public。TableFunction可以重载多个eval方法。Evaluation方法的输入参数类型，决定着表函数的输入类型。Evaluation方法也支持变参，例如：eval(String... strs)。返回表的类型取决于TableFunction的基本类型。Evaluation方法使用collect(T)发射输出的rows。

在Table API中，表函数在scala语言中使用方法如下：.join(Expression) 或者 .leftOuterJoin(Expression)，在java语言中使用方法如下：.join(String) 或者.leftOuterJoin(String)。

Join操作算子会使用表值函数(操作算子右边的表)产生的所有行进行(cross) join 外部表(操作算子左边的表)的每一行。

leftOuterJoin操作算子会使用表值函数(操作算子右边的表)产生的所有行进行(cross) join 外部表(操作算子左边的表)的每一行，并且在表函数返回一个空表的情况下会保留所有的outer rows。

在sql语法中稍微有点区别：
cross join用法是LATERAL TABLE(<TableFunction>)。
LEFT JOIN用法是在join条件中加入ON TRUE。

下面的理智讲的是如何使用表值函数。
```java
// The generic type "Tuple2<String, Integer>" determines the schema of the returned table as (String, Integer).
public class Split extends TableFunction<Tuple2<String, Integer>> {
    private String separator = " ";
    
    public Split(String separator) {
        this.separator = separator;
    }
    
    public void eval(String str) {
        for (String s : str.split(separator)) {
            // use collect(...) to emit a row
            collect(new Tuple2<String, Integer>(s, s.length()));
        }
    }
}

BatchTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);
Table myTable = ...         // table schema: [a: String]

// Register the function.
tableEnv.registerFunction("split", new Split("#"));

// Use the table function in the Java Table API. "as" specifies the field names of the table.
myTable.join("split(a) as (word, length)").select("a, word, length");
myTable.leftOuterJoin("split(a) as (word, length)").select("a, word, length");

// Use the table function in SQL with LATERAL and TABLE keywords.
join.md
tableEnv.sqlQuery("SELECT a, word, length FROM MyTable, LATERAL TABLE(split(a)) as T(word, length)");
// LEFT JOIN a table function (equivalent to "leftOuterJoin" in Table API).
tableEnv.sqlQuery("SELECT a, word, length FROM MyTable LEFT JOIN LATERAL TABLE(split(a)) as T(word, length) ON TRUE");

```

需要注意的是PROJO类型不需要一个确定的字段顺序。意味着你不能使用as修改表函数返回的pojo的字段的名字。

默认情况下TableFunction返回值类型是由flink类型抽取工具决定。对于基础类型，简单的POJOS是足够的，但是更复杂的类型，自定义类型，组合类型，会报错。这种情况下，返回值类型的TypeInformation，需要手动指定，方法是重载
TableFunction#getResultType()。

下面的例子，我们通过复写TableFunction#getResultType()方法使得表返回类型是RowTypeInfo(String, Integer)。
```java
public class CustomTypeSplit extends TableFunction<Row> {
    public void eval(String str) {
        for (String s : str.split(" ")) {
            Row row = new Row(2);
            row.setField(0, s);
            row.setField(1, s.length);
            collect(row);
        }
    }

    @Override
    public TypeInformation<Row> getResultType() {
        return Types.ROW(Types.STRING(), Types.INT());
    }
}
```
## 3. Aggregation Functions 聚合函数

用户自定义聚合函数聚合一张表(一行或者多行，一行有一个或者多个属性)为一个标量的值。



上图中是讲的一张饮料的表这个表有是那个字段五行数据，现在要做的事求出所有饮料的最高价。

聚合函数需要继承AggregateFunction。聚合函数工作方式如下：
首先，需要一个accumulator，这个是保存聚合中间结果的数据结构。调用AggregateFunction函数的createAccumulator()方法来创建一个空的accumulator.
随后，每个输入行都会调用accumulate()方法来更新accumulator。一旦所有的行被处理了，getValue()方法就会被调用，计算和返回最终的结果。

对于每个AggregateFunction，下面三个方法都是比不可少的：
createAccumulator()
accumulate()
getValue()

flink的类型抽取机制不能识别复杂的数据类型，比如，数据类型不是基础类型或者简单的pojos类型。所以，类似于ScalarFunction 和TableFunction，AggregateFunction提供了方法去指定返回结果类型的TypeInformation，用的是AggregateFunction#getResultType()。Accumulator类型用的是AggregateFunction#getAccumulatorType()。

除了上面的方法，这里有一些可选的方法。尽管有些方法是让系统更加高效的执行查询，另外的一些在特定的场景下是必须的。例如，merge()方法在会话组窗口上下文中是必须的。当一行数据是被视为跟两个回话窗口相关的时候，两个会话窗口的accumulators需要被join。

AggregateFunction的下面几个方法，根据使用场景的不同需要被实现：
retract()：在bounded OVER窗口的聚合方法中是需要实现的。
merge()：在很多batch 聚合和会话窗口聚合是必须的。
resetAccumulator(): 在大多数batch聚合是必须的。

AggregateFunction的所有方法都是需要被声明为public，而不是static。定义聚合函数需要实现org.apache.flink.table.functions.AggregateFunction同时需要实现一个或者多个accumulate方法。该方法可以被重载为不同的数据类型，并且支持变参。

在这里就不贴出来AggregateFunction的源码了。

下面举个求加权平均的栗子
为了计算加权平均值，累加器需要存储已累积的所有数据的加权和及计数。在栗子中定义一个WeightedAvgAccum类作为accumulator。尽管，retract(), merge(), 和resetAccumulator()方法在很多聚合类型是不需要的，这里也给出了栗子。
```java

/**
 * Accumulator for WeightedAvg.
 */
public static class WeightedAvgAccum {
    public long sum = 0;
    public int count = 0;
}

/**
 * Weighted Average user-defined aggregate function.
 */
public static class WeightedAvg extends AggregateFunction<Long, WeightedAvgAccum> {

    @Override
    public WeightedAvgAccum createAccumulator() {
        return new WeightedAvgAccum();
    }

    @Override
    public Long getValue(WeightedAvgAccum acc) {
        if (acc.count == 0) {
            return null;
        } else {
            return acc.sum / acc.count;
        }
    }

    public void accumulate(WeightedAvgAccum acc, long iValue, int iWeight) {
        acc.sum += iValue * iWeight;
        acc.count += iWeight;
    }

    public void retract(WeightedAvgAccum acc, long iValue, int iWeight) {
        acc.sum -= iValue * iWeight;
        acc.count -= iWeight;
    }
    
    public void merge(WeightedAvgAccum acc, Iterable<WeightedAvgAccum> it) {
        Iterator<WeightedAvgAccum> iter = it.iterator();
        while (iter.hasNext()) {
            WeightedAvgAccum a = iter.next();
            acc.count += a.count;
            acc.sum += a.sum;
        }
    }
    
    public void resetAccumulator(WeightedAvgAccum acc) {
        acc.count = 0;
        acc.sum = 0L;
    }
}

// register function
StreamTableEnvironment tEnv = ...
tEnv.registerFunction("wAvg", new WeightedAvg());

// use function
tEnv.sqlQuery("SELECT user, wAvg(points, level) AS avgPoints FROM userScores GROUP BY user");

```
## 4. 实现udf的最佳实践经验

Table API和SQL 代码生成器内部会尽可能多的尝试使用原生值。用户定义的函数可能通过对象创建、强制转换(casting)和拆装箱((un)boxing)引入大量开销。因此，强烈推荐参数和返回值的类型定义为原生类型而不是他们包装类型(boxing class)。Types.DATE 和Types.TIME可以用int代替。Types.TIMESTAMP可以用long代替。

我们建议用户自定义函数使用java编写而不是scala编写，因为scala的类型可能会有不被flink类型抽取器兼容。

用Runtime集成UDFs

有时候udf需要获取全局runtime信息或者在进行实际工作之前做一些设置和清除工作。Udf提供了open()和close()方法，可以被复写，功能类似Dataset和DataStream API的RichFunction方法。

Open()方法是在evaluation方法调用前调用一次。Close()是在evaluation方法最后一次调用后调用。

Open()方法提共一个FunctionContext，FunctionContext包含了udf执行环境的上下文，比如，metric group，分布式缓存文件，全局的job参数。

通过调用FunctionContext的相关方法，可以获取到相关的信息：

方法描述
* getMetricGroup() - 并行子任务的指标组
* getCachedFile(name) -分布式缓存文件的本地副本
* getJobParameter(name, defaultValue) - 给定key全局job参数。

下面，给出的例子就是通过FunctionContext在一个标量函数中获取全局job的参数。
```java
public class HashCode extends ScalarFunction {

    private int factor = 0;

    @Override
    public void open(FunctionContext context) throws Exception {
        // access "hashcode_factor" parameter
        // "12" would be the default value if parameter does not exist
        factor = Integer.valueOf(context.getJobParameter("hashcode_factor", "12")); 
    }

    public int eval(String s) {
        return s.hashCode() * factor;
    }
}

ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// set job parameter
Configuration conf = new Configuration();
conf.setString("hashcode_factor", "31");
env.getConfig().setGlobalJobParameters(conf);

// register the function
tableEnv.registerFunction("hashCode", new HashCode());

// use the function in Java Table API
myTable.select("string, string.hashCode(), hashCode(string)");

// use the function in SQL
tableEnv.sqlQuery("SELECT string, HASHCODE(string) FROM MyTable");
```