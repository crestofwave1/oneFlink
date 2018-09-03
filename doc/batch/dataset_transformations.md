## 0. DataSet转换
本文档对数据集上可用的转换进行了深入研究，有关Flink Java API的一般介绍，请参阅[编程指南](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/index.html)。

要压缩索引密集的数据集中的元素，请参考[Zip元素指南](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/zip_elements_guide.html)



### 1. Map

Map转换在数据集的每个元素上应用用户定义的映射函数,它实现了一对一的映射，也就是说，函数必须返回一个元素。
以下代码将整数对数据集转换为整数数据集:
```java
// MapFunction that adds two integer values
public class IntAdder implements MapFunction<Tuple2<Integer, Integer>, Integer> {
  @Override
  public Integer map(Tuple2<Integer, Integer> in) {
    return in.f0 + in.f1;
  }
}

// [...]
DataSet<Tuple2<Integer, Integer>> intPairs = // [...]
DataSet<Integer> intSums = intPairs.map(new IntAdder());
```

### 2. FlatMap
FlatMap转换在数据集的每个元素上应用用户定义的map映射函数。map函数的这个变体可以为每个输入元素返回任意多个结果元素(包括空元素)。  
以下代码将文本行数据集转换为单词数据集:
```java
// FlatMapFunction that tokenizes a String by whitespace characters and emits all String tokens.
public class Tokenizer implements FlatMapFunction<String, String> {
  @Override
  public void flatMap(String value, Collector<String> out) {
    for (String token : value.split("\\W")) {
      out.collect(token);
    }
  }
}

// [...]
DataSet<String> textLines = // [...]
DataSet<String> words = textLines.flatMap(new Tokenizer());
```


### 3. MapPartition
MapPartition在单个函数调用中转换并行分区。map-partition函数获取可迭代的分区，可以生成任意数量的结果值。
每个分区中的元素数量取决于并行度和以前的操作。

以下代码将文本行数据集转换为每个分区的计数数据集:
```java
public class PartitionCounter implements MapPartitionFunction<String, Long> {

  public void mapPartition(Iterable<String> values, Collector<Long> out) {
    long c = 0;
    for (String s : values) {
      c++;
    }
    out.collect(c);
  }
}

// [...]
DataSet<String> textLines = // [...]
DataSet<Long> counts = textLines.mapPartition(new PartitionCounter());
```


### 4. Filter
过滤器转换发生在数据集的每个元素上应用用户定义的过滤器函数，只保留那些函数返回true的元素。

以下代码从数据集中删除所有小于零的整数:
```java
// FilterFunction that filters out all Integers smaller than zero.
public class NaturalNumberFilter implements FilterFunction<Integer> {
  @Override
  public boolean filter(Integer number) {
    return number >= 0;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> naturalNumbers = intNumbers.filter(new NaturalNumberFilter());
```
**重要提示**:系统假设函数不修改应用谓词的元素。违反这个假设会导致不正确的结果。



### 5. 元祖数据集的投影

项目转换删除或移动元组数据集的元组字段。project(int…)方法选择应该由其索引保留的Tuple字段，
并在输出Tuple中定义它们的顺序。投影不需要定义用户函数。

下面的代码展示了在数据集中应用项目转换的不同方法:
```java
DataSet<Tuple3<Integer, Double, String>> in = // [...]
// converts Tuple3<Integer, Double, String> into Tuple2<String, Integer>
DataSet<Tuple2<String, Integer>> out = in.project(2,0);

```
#### 5.1 投影型提示

注意，Java编译器不能推断项目操作符的返回类型。如果您在项目操作符的结果上调用另一个操作符，这可能会导致问题，例如:
```java
DataSet<Tuple5<String,String,String,String,String>> ds = ....
DataSet<Tuple1<String>> ds2 = ds.project(0).distinct(0);
```

这个问题可以通过这样提示项目操作符的返回类型来解决:
```java
DataSet<Tuple1<String>> ds2 = ds.<Tuple1<String>>project(0).distinct(0);
```


### 6. 分组数据集上的转换
reduce操作可以对分组数据集进行操作,可以通过多种方式指定要用于分组的键:

- key表达式
- 一个key选择器函数
- 一个或多个字段位置键(仅限Tuple数据集)
- 案例类字段(仅适用于案例类)
- 请查看reduce示例以了解如何指定分组键。


### 7. 减少对分组数据集

应用于分组数据集的Reduce转换使用用户定义的Reduce函数将每个组缩减为单个元素。对于每组输入元素，reduce函数依次将对元素组合成一个元素，直到每组只剩下一个元素为止。

注意，对于ReduceFunction，返回对象的键控字段应该与输入值匹配。这是因为reduce是隐式组合的，当传递给reduce运算符时，从combine运算符发出的对象再次按键分组。



#### 7.1 按键表达式分组的数据集上的Reduce
key表达式指定数据集中每个元素的一个或多个字段。每个键表达式要么是公共字段的名称，要么是getter方法的名称。
点可以用来向下钻取物体。关键表达式“*”选择所有字段。
下面的代码展示了如何使用key表达式对POJO数据集进行分组并使用reduce函数对其进行缩减。
```java
// some ordinary POJO
public class WC {
  public String word;
  public int count;
  // [...]
}

// ReduceFunction that sums Integer attributes of a POJO
public class WordCounter implements ReduceFunction<WC> {
  @Override
  public WC reduce(WC in1, WC in2) {
    return new WC(in1.word, in1.count + in2.count);
  }
}

// [...]
DataSet<WC> words = // [...]
DataSet<WC> wordCounts = words
                         // DataSet grouping on field "word"
                         .groupBy("word")
                         // apply ReduceFunction on grouped DataSet
                         .reduce(new WordCounter());
```

### 8. 减少key选择器函数分组的数据集
key选择函数从数据集中的每个元素中提取一个键值。提取的键值用于对数据集进行分组。   
下面的代码展示了如何使用key选择器函数对POJO数据集进行分组并使用reduce函数对其进行缩减。   
```java
// some ordinary POJO
public class WC {
  public String word;
  public int count;
  // [...]
}

// ReduceFunction that sums Integer attributes of a POJO
public class WordCounter implements ReduceFunction<WC> {
  @Override
  public WC reduce(WC in1, WC in2) {
    return new WC(in1.word, in1.count + in2.count);
  }
}

// [...]
DataSet<WC> words = // [...]
DataSet<WC> wordCounts = words
                         // DataSet grouping on field "word"
                         .groupBy(new SelectWord())
                         // apply ReduceFunction on grouped DataSet
                         .reduce(new WordCounter());

public class SelectWord implements KeySelector<WC, String> {
  @Override
  public String getKey(Word w) {
    return w.word;
  }
}
```

### 9. 按字段位置key分组的数据集上的Reduce(只适用于元组数据集)
字段位置键指定作为分组键使用的元组数据集的一个或多个字段。下面的代码展示了如何使用字段位置键并应用reduce函数.
```java
DataSet<Tuple3<String, Integer, Double>> tuples = // [...]
DataSet<Tuple3<String, Integer, Double>> reducedTuples = tuples
                                         // group DataSet on first and second field of Tuple
                                         .groupBy(0, 1)
                                         // apply ReduceFunction on grouped DataSet
                                         .reduce(new MyTupleReducer());
```


### 10.减少按Case类字段分组的数据集
在使用Case类时，您还可以使用字段的名称指定分组键:
```java
Not  supported .
```


### 11. 在分组数据集进行GroupReduce
应用于分组数据集的GroupReduce转换会为每个组调用用户定义的group-reduce函数。这与Reduce之间的区别是，用户定义的函数一次获取整个组。
该函数在组的所有元素上使用Iterable调用，并可以返回任意数量的结果元素。


#### 11.1 根据字段位置键对数据集进行分组(只对元组数据集进行分组)
下面的代码展示了如何从按整数分组的数据集中删除重复的字符串。
```java
public class DistinctReduce
         implements GroupReduceFunction<Tuple2<Integer, String>, Tuple2<Integer, String>> {

  @Override
  public void reduce(Iterable<Tuple2<Integer, String>> in, Collector<Tuple2<Integer, String>> out) {

    Set<String> uniqStrings = new HashSet<String>();
    Integer key = null;

    // add all strings of the group to the set
    for (Tuple2<Integer, String> t : in) {
      key = t.f0;
      uniqStrings.add(t.f1);
    }

    // emit all unique strings.
    for (String s : uniqStrings) {
      out.collect(new Tuple2<Integer, String>(key, s));
    }
  }
}

// [...]
DataSet<Tuple2<Integer, String>> input = // [...]
DataSet<Tuple2<Integer, String>> output = input
                           .groupBy(0)            // group DataSet by the first tuple field
                           .reduceGroup(new DistinctReduce());  // apply GroupReduceFunction
```



#### 11.2 根据键表达式、键选择器函数或Case类字段对数据集进行分组

与Reduce转换中的
[键表达式](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/dataset_transformations.html#reduce-on-dataset-grouped-by-key-expression)、
[键选择器函数](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/dataset_transformations.html#reduce-on-dataset-grouped-by-keyselector-function)
和[case类字段](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/dataset_transformations.html#reduce-on-dataset-grouped-by-case-class-fields)类似。


#### 11.3 GroupReduce分类组
一个group-reduce函数使用Iterable访问一个组的元素。可选地，Iterable可以按照指定的顺序分发组的元素。在许多情况下，这有助于降低用户定义的group-reduce函数的复杂性并提高其效率。  
下面的代码展示了另一个示例，如何删除按整数分组并按字符串排序的数据集中的重复字符串。

```java
// GroupReduceFunction that removes consecutive identical elements
public class DistinctReduce
         implements GroupReduceFunction<Tuple2<Integer, String>, Tuple2<Integer, String>> {

  @Override
  public void reduce(Iterable<Tuple2<Integer, String>> in, Collector<Tuple2<Integer, String>> out) {
    Integer key = null;
    String comp = null;

    for (Tuple2<Integer, String> t : in) {
      key = t.f0;
      String next = t.f1;

      // check if strings are different
      if (com == null || !next.equals(comp)) {
        out.collect(new Tuple2<Integer, String>(key, next));
        comp = next;
      }
    }
  }
}

// [...]
DataSet<Tuple2<Integer, String>> input = // [...]
DataSet<Double> output = input
                         .groupBy(0)                         // group DataSet by first field
                         .sortGroup(1, Order.ASCENDING)      // sort groups on second tuple field
                         .reduceGroup(new DistinctReduce());
```

注意:如果在reduce操作之前使用操作符的基于排序的执行策略来建立分组排序，那么分组排序通常是无开销的。




#### 11.4 可以合并的GroupReduceFunctions
与reduce函数相反，group-reduce函数不是隐式组合的。为了使一个组约简函数能够组合，必须实现GroupCombineFunction接口。  
重要提示:GroupCombineFunction接口的泛型输入和输出类型必须与GroupReduceFunction的泛型输入类型相等，如下例所示:
```java
// Combinable GroupReduceFunction that computes a sum.
public class MyCombinableGroupReducer implements
  GroupReduceFunction<Tuple2<String, Integer>, String>,
  GroupCombineFunction<Tuple2<String, Integer>, Tuple2<String, Integer>>
{
  @Override
  public void reduce(Iterable<Tuple2<String, Integer>> in,
                     Collector<String> out) {

    String key = null;
    int sum = 0;

    for (Tuple2<String, Integer> curr : in) {
      key = curr.f0;
      sum += curr.f1;
    }
    // concat key and sum and emit
    out.collect(key + "-" + sum);
  }

  @Override
  public void combine(Iterable<Tuple2<String, Integer>> in,
                      Collector<Tuple2<String, Integer>> out) {
    String key = null;
    int sum = 0;

    for (Tuple2<String, Integer> curr : in) {
      key = curr.f0;
      sum += curr.f1;
    }
    // emit tuple with key and sum
    out.collect(new Tuple2<>(key, sum));
  }
}
```
### 12. 对分组数据集进行分组组合

群组合变换是可组合群预函数中组合步骤的广义形式。因为它允许将输入类型I与任意输出类型O相结合，相比之下，GroupReduce的组合步骤只允许从输入类型I合并到输出类型I。
这是因为GroupReduceFunction中的reduce步骤需要输入类型I。

在某些应用程序中，在执行其他转换(例如减少数据大小)之前，最好将数据集组合成一种中间格式。这可以通过使用很少的成本进行组合群转换来实现。   

注意:分组数据集上的GroupCombine在内存中执行，使用贪心策略，该策略可能不是一次处理所有数据，而是在多个步骤中处理。
它也在没有数据交换的单独分区上执行，就像在GroupReduce转换中一样。这可能导致部分结果。    

下面的示例演示如何使用CombineGroup转换来实现另一种WordCount实现。   

```java
DataSet<String> input = [..] // The words received as input

DataSet<Tuple2<String, Integer>> combinedWords = input
  .groupBy(0) // group identical words
  .combineGroup(new GroupCombineFunction<String, Tuple2<String, Integer>() {

    public void combine(Iterable<String> words, Collector<Tuple2<String, Integer>>) { // combine
        String key = null;
        int count = 0;

        for (String word : words) {
            key = word;
            count++;
        }
        // emit tuple with word and count
        out.collect(new Tuple2(key, count));
    }
});

DataSet<Tuple2<String, Integer>> output = combinedWords
  .groupBy(0)                              // group by words again
  .reduceGroup(new GroupReduceFunction() { // group reduce with full data exchange

    public void reduce(Iterable<Tuple2<String, Integer>>, Collector<Tuple2<String, Integer>>) {
        String key = null;
        int count = 0;

        for (Tuple2<String, Integer> word : words) {
            key = word;
            count++;
        }
        // emit tuple with word and count
        out.collect(new Tuple2(key, count));
    }
});
```
上面的替代WordCount实现演示了GroupCombine在执行GroupReduce转换之前如何组合单词。
上面的例子只是一个概念的证明。注意，combine步骤如何改变数据集的类型，通常在执行GroupReduce之前需要额外的映射转换。



### 13. 聚合分组元组数据集
有一些常用的聚合操作。聚合转换提供了以下内置的聚合函数:
- Sum
- Min,and
- Max

聚合转换只能应用于元组数据集，并且只支持用于分组的字段位置键。    

下面的代码展示了如何对按字段位置键分组的数据集应用聚合转换:

```java
DataSet<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input
                                   .groupBy(1)        // group DataSet on second field
                                   .aggregate(SUM, 0) // compute sum of the first field
                                   .and(MIN, 2);      // compute minimum of the third field
```


要在一个数据集上应用多个聚合，必须在第一个聚合之后使用.and()函数.这意味着
aggregate(SUM，0).and(MIN, 2)会产生一个字段0和原始数据集的最小字段2的和。
与之相反，.aggregate(SUM, 0).aggregate(MIN, 2)会在一个聚合上应用一个聚合。
在给定的例子中，它将在计算按字段1分组的字段0的和之后产生字段2的最小值。

注意:聚合函数的集合将在将来进行扩展。




### 14. 分组元组数据集上的MinBy / MaxBy
MinBy (MaxBy)转换为每组元组选择一个元组。选定的元组是一个或多个指定字段的值为最小(最大值)的元组。用于比较的字段必须是有效的关键字段，即具有可比性。
如果多个元组具有最小(最大)字段值，则返回这些元组的任意元组。   

下面的代码展示了如何从数据集<Tuple3<Integer, String, Double>>中选择具有整数最小值的元组和具有相同字符串值的每组元组的双字段:   
```java
DataSet<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input
                                   .groupBy(1)   // group DataSet on second field
                                   .minBy(0, 2); // select tuple with minimum values for first and third field.
```




### 15. 在完整数据集reduce
Reduce转换对数据集的所有元素应用用户定义的Reduce函数。reduce函数随后将对元素组合成一个元素，直到只剩下一个元素。   

下面的代码展示了如何对整数数据集的所有元素求和:    
```java
// ReduceFunction that sums Integers
public class IntSummer implements ReduceFunction<Integer> {
  @Override
  public Integer reduce(Integer num1, Integer num2) {
    return num1 + num2;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> sum = intNumbers.reduce(new IntSummer());
```

使用Reduce转换减少整个数据集意味着最终的Reduce操作不能并行执行。但是，reduce函数是自动组合的，因此reduce转换不会限制大多数用例的可伸缩性。

### 16. 在完整数据集上进行GroupReduce
GroupReduce转换将用户定义的group-reduce函数应用到数据集中的所有元素上。group-reduce可以遍历数据集的所有元素，并返回任意数量的结果元素。

下面的例子展示了如何在一个完整的数据集上应用GroupReduce转换:   
```java

DataSetDataSe <Integer> input = // [...]
// apply a (preferably combinable) GroupReduceFunction to a DataSet
DataSet<Double> output = input.reduceGroup(new MyGroupReducer());
```

注意:如果group-reduce函数不能组合，则在完整数据集上的GroupReduce转换不能并行执行。
因此，这可能是一个非常密集的计算操作。请参阅上面关于“Combinable GroupReduceFunctions”的段落，了解如何实现组合群约简函数。



### 17.在完整的数据集中进行分组组合
完整数据集上的GroupCombine工作方式与分组数据集上的GroupCombine类似。
数据在所有节点上进行分区，然后以贪婪的方式进行组合(即一次只组合到内存中的数据)。

### 18.聚合整个元组数据集
有一些常用的聚合操作。聚合转换提供了以下内置的聚合函数:
- Sum,
- Min, and
- Max.
聚合转换只能应用于元组数据集。

下面的代码展示了如何在完整数据集上应用聚合转换:   

```java

DataSetDataSe <Tuple2<Integer, Double>> input = // [...]
DataSet<Tuple2<Integer, Double>> output = input
                                     .aggregate(SUM, 0)    // compute sum of the first field
                                     .and(MIN, 1);    // compute minimum of the second field
```
Note: Extending the set of supported aggregation functions is on our roadmap.


注意:扩展受支持的聚合函数集在我们的路线图中。

### 19. MinBy / MaxBy 在全元素数据集
MinBy (MaxBy)转换从元组数据集中选择单个元组。选定的元组是一个或多个指定字段的值为最小(最大值)的元组。用于比较的字段必须是有效的关键字段，
即,具有可比性。如果多个元组具有最小(最大)字段值，则返回这些元组的任意元组。

下面的代码展示了如何从数据集<Tuple3<Integer, String, Double>>中选择具有整数和双字段最大值的元组:
```java
DataSet<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input
                                   .maxBy(0, 2); // select tuple with maximum values for first and third field.
```

### 20.去重
:
去重的转换是计算源数据集的去重元素的数据集，以下代码从数据集中删除所有重复的元素:

```java
DataSet<Tuple2<Integer, Double>> input = // [...]
DataSet<Tuple2<Integer, Double>> output = input.distinct();
```

还可以使用以下方法更改数据集中元素的区分方式:   

- 一个或多个字段位置键(仅限Tuple数据集)，
- 一个key选择器函数
- 一个key表达式。


#### 20.1 属性位置键去重
```java
DataSet<Tuple2<Integer, Double, String>> input = // [...]
DataSet<Tuple2<Integer, Double, String>> output = input.distinct(0,2);

```


#### 20.2 KeySelector函数去重
```java
private static class AbsSelector implements KeySelector<Integer, Integer> {
private static final long serialVersionUID = 1L;
	@Override
	public Integer getKey(Integer t) {
    	return Math.abs(t);
	}
}
DataSet<Integer> input = // [...]
DataSet<Integer> output = input.distinct(new AbsSelector());
```

#### 20.3 key表达式去重
```java

// some ordinary POJO// som 
public class CustomType {
  public String aName;
  public int aNumber;
  // [...]
}

DataSet<CustomType> input = // [...]
DataSet<CustomType> output = input.distinct("aName", "aNumber");
```
还可以通过通配符指示使用所有字段:  
```java
DataSet<CustomType> input = // [...]
DataSet<CustomType> output = input.distinct("*");
```


### 21. Join

连接转换将两个数据集连接到一个数据集。两个数据集的元素连接在一个或多个键上，可以使用这些键来指定:   

- 一个key表达式
- 一个key选择器函数
- 一个或多个字段位置键(仅限Tuple数据集)。
- Case类字段
执行连接转换有几种不同的方法，如下所示。


#### 21.1 默认join(加入Tuple2)
默认的连接转换产生一个新的带有两个字段的元组数据集。每个元组包含第一个元组字段中第一个输入数据集的连接元素和第二个字段中第二个输入数据集的匹配元素。
下面的代码显示了使用字段位置键的默认连接转换:    
```java
public static class User { public String name; public int zip; }
public static class Store { public Manager mgr; public int zip; }
DataSet<User> input1 = // [...]
DataSet<Store> input2 = // [...]
// result dataset is typed as Tuple2
DataSet<Tuple2<User, Store>>
            result = input1.join(input2)
                           .where("zip")       // key of the first input (users)
                           .equalTo("zip");    // key of the second input (stores)
```

#### 21.2 Join 和 Join 函数
join转换还可以调用用户定义的join函数来处理联接元组。join函数接收第一个输入数据集的一个元素和第二个输入数据集的一个元素，并准确地返回一个元素。   

下面的代码使用key选择器函数执行数据集与自定义java对象的join和元组数据集的join，并演示如何使用用户定义的join函数:    
```java
// some POJO
public class Rating {
  public String name;
  public String category;
  public int points;
}

// Join function that joins a custom POJO with a Tuple
public class PointWeighter
         implements JoinFunction<Rating, Tuple2<String, Double>, Tuple2<String, Double>> {

  @Override
  public Tuple2<String, Double> join(Rating rating, Tuple2<String, Double> weight) {
    // multiply the points and rating and construct a new output tuple
    return new Tuple2<String, Double>(rating.name, rating.points * weight.f1);
  }
}

DataSet<Rating> ratings = // [...]
DataSet<Tuple2<String, Double>> weights = // [...]
DataSet<Tuple2<String, Double>>
            weightedRatings =
            ratings.join(weights)

                   // key of the first input
                   .where("category")

                   // key of the second input
                   .equalTo("f0")

                   // applying the JoinFunction on joining pairs
                   .with(new PointWeighter());
```



#### 21.3 Join 和 flat-join 函数
与Map和FlatMap类似，FlatJoin的行为方式与Join相同，但是它不返回一个元素，而是返回(collect)、0、1或更多元素。   
```java
public class PointWeighter
         implements FlatJoinFunction<Rating, Tuple2<String, Double>, Tuple2<String, Double>> {
  @Override
  public void join(Rating rating, Tuple2<String, Double> weight,
	  Collector<Tuple2<String, Double>> out) {
	if (weight.f1 > 0.1) {
		out.collect(new Tuple2<String, Double>(rating.name, rating.points * weight.f1));
	}
  }
}

DataSet<Tuple2<String, Double>>
            weightedRatings =
            ratings.join(weights) // [...]
```



#### 21.4 join投影(只适用于Java/Python)

join转换可以使用如下所示的投影构造结果元组:   
```java
DataSet<Tuple3<Integer, Byte, String>> input1 = // [...]
DataSet<Tuple2<Integer, Double>> input2 = // [...]
DataSet<Tuple4<Integer, String, Double, Byte>>
            result =
            input1.join(input2)
                  // key definition on first DataSet using a field position key
                  .where(0)
                  // key definition of second DataSet using a field position key
                  .equalTo(0)
                  // select and reorder fields of matching tuples
                  .projectFirst(0,2).projectSecond(1).projectFirst(1);
```

projectFirst(int…)和projectSecond(int…)选择应该装配到输出元组中的第一个和第二个连接输入的字段。索引的顺序定义了输出元组中字段的顺序,
join投影也适用于非元组数据集。在这种情况下，必须在没有参数的情况下调用projectFirst()或projectSecond()，以便向输出元组添加一个已连接的元素。



#### 21.5 join数据集大小提示
为了指导优化器选择正确的执行策略，您可以提示连接数据集的大小，如下所示:
```java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>>
            result1 =
            // hint that the second DataSet is very small
            input1.joinWithTiny(input2)
                  .where(0)
                  .equalTo(0);

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>>
            result2 =
            // hint that the second DataSet is very large
            input1.joinWithHuge(input2)
                  .where(0)
                  .equalTo(0);

```


#### 21.6 join算法提示
Flink运行时可以以各种方式执行join。在不同的情况下，每一种可能的方法都优于其他方法。系统试图自动选择一种合理的方式，
但允许您手动选择一种策略，以防需要强制执行特定的连接方式。
```java
DataSet<SomeType> input1 = // [...]
DataSet<AnotherType> input2 = // [...]

DataSet<Tuple2<SomeType, AnotherType> result =
      input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
            .where("id").equalTo("key");

```


以下是一些提示:    

- OPTIMIZER_CHOOSES:等同于完全不给出任何提示，由系统决定。

- BROADCAST_HASH_FIRST:广播第一个输入并从中构建一个哈希表，由第二个输入进行探查。如果第一个输入非常小，那么这是一个很好的策略。

- BROADCAST_HASH_SECOND:广播第二个输入并从中构建一个哈希表，第一个输入会探测到这个哈希表。如果第二个输入非常小，这是一个很好的策略。

- REPARTITION_HASH_FIRST:系统划分每个输入(除非输入已经分区)并从第一个输入构建一个哈希表。如果第一个输入小于第二个输入，那么这个策略是好的，但是两个输入都仍然很大。注意:这是系统使用的默认回退策略，如果不能进行大小估计，也不能重用预先存在的分区和排序订单。

- REPARTITION_HASH_SECOND:系统划分每个输入(除非输入已经分区)并从第二个输入构建一个哈希表。如果第二个输入比第一个小，这个策略是好的，但是两个输入仍然很大。

- REPARTITION_SORT_MERGE:系统分区(洗牌)每个输入(除非输入已经分区)，并对每个输入进行排序(除非已经排序)。输入由已排序输入的流合并连接。如果一个或两个输入都已排序，这种策略是好的。




### 22. 外连接
外部连接转换在两个数据集上执行左、右或完整的外部连接。外部连接类似于常规(内部)连接，并创建键上相等的所有元素对。
此外，如果在另一侧没有找到匹配的键，则保留“外部”一侧(如果满了，则保留左、右或两者)的记录。
将匹配的元素对(或一个元素和另一个输入的空值)提供给JoinFunction，使这对元素变成单个元素，或者提供给FlatJoinFunction，使这对元素变成任意多(不包括任何元素)的元素。

两个数据集的元素连接在一个或多个键上，可以使用这些键来指定:  

- 一个关键表达式
- 一个key选择器函数
- 一个或多个字段位置键(仅限Tuple数据集)。
- Case类字段

外部连接只支持Java和Scala数据集API。


#### 22.1 外连接join函数
外部连接转换调用用户定义的连接函数来处理连接元组。联接函数接收第一个输入数据集的一个元素和第二个输入数据集的一个元素，并准确地返回一个元素。
根据外部连接的类型(左、右、满)，连接函数的两个输入元素之一可以为空。

下面的代码使用自定义java对象执行数据集的左外连接，使用键选择器函数执行元组数据集，并演示如何使用用户定义的连接函数:    
```java
// some POJO
public class Rating {
  public String name;
  public String category;
  public int points;
}

// Join function that joins a custom POJO with a Tuple
public class PointAssigner
         implements JoinFunction<Tuple2<String, String>, Rating, Tuple2<String, Integer>> {

  @Override
  public Tuple2<String, Integer> join(Tuple2<String, String> movie, Rating rating) {
    // Assigns the rating points to the movie.
    // NOTE: rating might be null
    return new Tuple2<String, Double>(movie.f0, rating == null ? -1 : rating.points;
  }
}

DataSet<Tuple2<String, String>> movies = // [...]
DataSet<Rating> ratings = // [...]
DataSet<Tuple2<String, Integer>>
            moviesWithPoints =
            movies.leftOuterJoin(ratings)

                   // key of the first input
                   .where("f0")

                   // key of the second input
                   .equalTo("name")

                   // applying the JoinFunction on joining pairs
                   .with(new PointAssigner());
```

#### 22.2 外连接 Flat-Join函数
与Map和FlatMap类似，具有flat-join函数的外部连接的行为与具有连接函数的外部连接的行为相同，但是它不返回一个元素，而是返回(collection)、0、1或更多元素。

```java
public class PointAssigner
         implements FlatJoinFunction<Tuple2<String, String>, Rating, Tuple2<String, Integer>> {
  @Override
  public void join(Tuple2<String, String> movie, Rating rating
    Collector<Tuple2<String, Integer>> out) {
  if (rating == null ) {
    out.collect(new Tuple2<String, Integer>(movie.f0, -1));
  } else if (rating.points < 10) {
    out.collect(new Tuple2<String, Integer>(movie.f0, rating.points));
  } else {
    // do not emit
  }
}

DataSet<Tuple2<String, Integer>>
            moviesWithPoints =
            movies.leftOuterJoin(ratings) // [...]
```





#### 22.3 join算法提示
Flink运行时可以以各种方式执行外部连接。在不同的情况下，每一种可能的方法都优于其他方法。
系统试图自动选择一种合理的方式，但允许您手动选择一种策略，以防您需要强制执行一种特定的方式来执行外部连接。   
```java
DataSet<SomeType> input1 = // [...]
DataSet<AnotherType> input2 = // [...]

DataSet<Tuple2<SomeType, AnotherType> result1 =
      input1.leftOuterJoin(input2, JoinHint.REPARTITION_SORT_MERGE)
            .where("id").equalTo("key");

DataSet<Tuple2<SomeType, AnotherType> result2 =
      input1.rightOuterJoin(input2, JoinHint.BROADCAST_HASH_FIRST)
            .where("id").equalTo("key");
```


下面是一些提示:

- OPTIMIZER_CHOOSES:等同于完全不给出任何提示，由系统决定。
- BROADCAST_HASH_FIRST:广播第一个输入并从中构建一个哈希表，由第二个输入进行探查。如果第一个输入非常小，那么这是一个很好的策略。
- BROADCAST_HASH_SECOND:广播第二个输入并从中构建一个哈希表，第一个输入会探测到这个哈希表。如果第二个输入非常小，这是一个很好的策略。
- REPARTITION_HASH_FIRST:系统划分每个输入(除非输入已经分区)并从第一个输入构建一个哈希表。如果第一个输入小于第二个输入，那么这个策略是好的，但是两个输入都仍然很大。
- REPARTITION_HASH_SECOND:系统划分每个输入(除非输入已经分区)并从第二个输入构建一个哈希表。如果第二个输入比第一个小，这个策略是好的，但是两个输入仍然很大。
- REPARTITION_SORT_MERGE:系统分区(洗牌)每个输入(除非输入已经分区)，并对每个输入进行排序(除非已经排序)。输入由已排序输入的流合并连接。如果一个或两个输入都已排序，这种策略是好的。


注意:并不是所有的执行策略都被每个外部连接类型所支持。

- LeftOuterJoin支持:    
    - OPTIMIZER_CHOOSES
    - BROADCAST_HASH_SECOND
    - REPARTITION_HASH_SECOND
    - REPARTITION_SORT_MERGE   
 
- RightOuterJoin支持:
    - OPTIMIZER_CHOOSES
    -BROADCAST_HASH_FIRST
    - REPARTITION_HASH_FIRST
    - REPARTITION_SORT_MERGE

- FullOuterJoin支持:    
    - OPTIMIZER_CHOOSES
    - REPARTITION_SORT_MERGE
    
    
### 23. 笛卡尔乘积
cross转换将两个数据集组合成一个数据集。它构建两个输入数据集元素的所有两两组合，即它建立了一个笛卡尔积。
cross转换要么在每一对元素上调用用户定义的cross函数，要么输出Tuple2。两种模式如下所示。

注意:交叉可能是一个非常计算密集型的操作，甚至可以挑战大型计算集群!    



#### 23.1 cross 与用户定义函数
交叉转换可以调用用户定义的交叉函数。交叉函数接收第一个输入的一个元素和第二个输入的一个元素，并返回一个结果元素。

下面的代码展示了如何使用cross函数对两个数据集应用cross转换:


```java
public class Coord {
  public int id;
  public int x;
  public int y;
}

// CrossFunction computes the Euclidean distance between two Coord objects.
public class EuclideanDistComputer
         implements CrossFunction<Coord, Coord, Tuple3<Integer, Integer, Double>> {

  @Override
  public Tuple3<Integer, Integer, Double> cross(Coord c1, Coord c2) {
    // compute Euclidean distance of coordinates
    double dist = sqrt(pow(c1.x - c2.x, 2) + pow(c1.y - c2.y, 2));
    return new Tuple3<Integer, Integer, Double>(c1.id, c2.id, dist);
  }
}

DataSet<Coord> coords1 = // [...]
DataSet<Coord> coords2 = // [...]
DataSet<Tuple3<Integer, Integer, Double>>
            distances =
            coords1.cross(coords2)
                   // apply CrossFunction
                   .with(new EuclideanDistComputer());
```


#### 23.2 cross投影
交cross 转换还可以使用投影构造结果元组，如下所示:

```java
DataSet<Tuple3<Integer, Byte, String>> input1 = // [...]
DataSet<Tuple2<Integer, Double>> input2 = // [...]
DataSet<Tuple4<Integer, Byte, Integer, Double>
            result =
            input1.cross(input2)
                  // select and reorder fields of matching tuples
                  .projectSecond(0).projectFirst(1,0).projectSecond(1);
```

cross投影中的字段选择与join结果的投影工作方式相同。




#### 23.3 cross数据集大小提示
为了指导优化器选择正确的执行策略，您可以提示要交叉数据集的大小，如下所示:

```java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple4<Integer, String, Integer, String>>
            udfResult =
                  // hint that the second DataSet is very small
            input1.crossWithTiny(input2)
                  // apply any Cross function (or projection)
                  .with(new MyCrosser());

DataSet<Tuple3<Integer, Integer, String>>
            projectResult =
                  // hint that the second DataSet is very large
            input1.crossWithHuge(input2)
                  // apply a projection (or any Cross function)
                  .projectFirst(0,1).projectSecond(1);
```


### 24. CoGroup

CoGroup转换共同处理两个数据集的组。两个数据集在一个已定义的键上分组，共享相同键的两个数据集的组一起传递给一个用户定义CoGroup函数。
如果对于特定的键，只有一个数据集有一个组，则用这个组和一个空组调用 co-group函数。co-group函数可以分别迭代两个组的元素并返回任意数量的结果元素。

与Reduce、GroupReduce和Join类似，可以使用不同的键选择方法定义键。




#### 24.1 数据集CoGroup
这个示例展示了如何按字段位置键(只对Tuple数据集进行分组)。对于pojo类型和键表达式也可以这样做。
```java
// Some CoGroupFunction definition
class MyCoGrouper
         implements CoGroupFunction<Tuple2<String, Integer>, Tuple2<String, Double>, Double> {

  @Override
  public void coGroup(Iterable<Tuple2<String, Integer>> iVals,
                      Iterable<Tuple2<String, Double>> dVals,
                      Collector<Double> out) {

    Set<Integer> ints = new HashSet<Integer>();

    // add all Integer values in group to set
    for (Tuple2<String, Integer>> val : iVals) {
      ints.add(val.f1);
    }

    // multiply each Double value with each unique Integer values of group
    for (Tuple2<String, Double> val : dVals) {
      for (Integer i : ints) {
        out.collect(val.f1 * i);
      }
    }
  }
}

// [...]
DataSet<Tuple2<String, Integer>> iVals = // [...]
DataSet<Tuple2<String, Double>> dVals = // [...]
DataSet<Double> output = iVals.coGroup(dVals)
                         // group first DataSet on first tuple field
                         .where(0)
                         // group second DataSet on first tuple field
                         .equalTo(0)
                         // apply CoGroup function on each pair of groups
                         .with(new MyCoGrouper());
```




### 25. 合并

两个具有相同类型的数据集才能生成并集。多个数据集的联合可以通过多个联合调用实现，如下所示:    
```java

DataSetDataSe <Tuple2<String, Integer>> vals1 = // [...]
DataSet<Tuple2<String, Integer>> vals2 = // [...]
DataSet<Tuple2<String, Integer>> vals3 = // [...]
DataSet<Tuple2<String, Integer>> unioned = vals1.union(vals2).union(vals3);

```

### 26. 重新平衡

均衡地重新平衡数据集的并行分区，以消除数据倾斜。    

```java

DataSetDataSe <String> in = // [...]
// rebalance DataSet and apply a Map transformation.
DataSet<Tuple2<String, String>> out = in.rebalance()
                                        .map(new Mapper());
```

### 27. 哈希分区
哈希对给定键上的数据集进行分区。键可以指定为位置键、表达式键和键选择器函数(请参阅Reduce示例以了解如何指定键)。   
```java

DataSetDataSe <Tuple2<String, Integer>> in = // [...]
// hash-partition DataSet by String value and apply a MapPartition transformation.
DataSet<Tuple2<String, String>> out = in.partitionByHash(0)
                                        .mapPartition(new PartitionMapper());
```

### 28. 范围分割
对给定键上的数据集进行范围分区。键可以指定为位置键、表达式键和键选择器函数(请参阅[Reduce示例](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/dataset_transformations.html#reduce-on-grouped-dataset)以了解如何指定键)。   

```java
DataSet<Tuple2<String, Integer>> in = // [...]
// range-partition DataSet by String value and apply a MapPartition transformation.
DataSet<Tuple2<String, String>> out = in.partitionByRange(0)
                                        .mapPartition(new PartitionMapper());
```


### 29. 分区排序
以指定的顺序对指定字段上的数据集的所有分区进行本地排序。字段可以指定为字段表达式或字段位置(请参阅[Reduce示例](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/dataset_transformations.html#reduce-on-grouped-dataset)以了解如何指定键)。   
通过链接sortPartition()调用，可以在多个字段上对分区进行排序。



### 30. First-n
返回数据集的第n(任意)个元素。First-n可以应用于常规数据集、分组数据集或分组排序数据集。
分组键可以指定为键选择器函数或字段位置键(请参阅[Reduce示例](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/dataset_transformations.html#reduce-on-grouped-dataset)以了解如何指定键)。

```java
DataSet<Tuple2<String, Integer>> in = // [...]
// Return the first five (arbitrary) elements of the DataSet
DataSet<Tuple2<String, Integer>> out1 = in.first(5);

// Return the first two (arbitrary) elements of each String group
DataSet<Tuple2<String, Integer>> out2 = in.groupBy(0)
                                          .first(2);

// Return the first three elements of each String group ordered by the Integer field
DataSet<Tuple2<String, Integer>> out3 = in.groupBy(0)
                                          .sortGroup(1, Order.ASCENDING)
                                          .first(3);
```