#Flink DataStream API 编程指南

Flink中的DataStream程序是对数据流进行转换（例如，过滤、更新状态、定义窗口、聚合）的常用方式。数据流来源于多种数据源（例如，消息队列，socket流，文件）。通过sinks返回结果，例如将数据写入文件或标准输出（如命令行终端）。Flink程序可以运行在各种上下文环境中，独立或嵌入于其他程序中。
执行过程可以在本地JVM也可以在由许多机器组成的集群上。

关于Flink API的基本概念介绍请参阅[基本概念]。

为了创建你的Flink DataStream程序，我们鼓励你从[解构Flink程序]开始，并逐渐添加你自己的[transformations]。本节其余部分作为附加操作和高级功能的参考。



##示例程序

以下是基于流式窗口进行word count的一个完整可运行的程序示例，它从一个宽度为5秒的网络socket窗口中统计单词个数。你可以复制并粘贴代码用以本地运行。


*Java*
```
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(0)
                .timeWindow(Time.seconds(5))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
```


*Scala*

```
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time

object WindowWordCount {
  def main(args: Array[String]) {

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val text = env.socketTextStream("localhost", 9999)

    val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
      .map { (_, 1) }
      .keyBy(0)
      .timeWindow(Time.seconds(5))
      .sum(1)

    counts.print

    env.execute("Window Stream WordCount")
  }
}
```

要运行示例程序，首先在终端启动netcat作为输入流：

```
nc -lk 9999
```

输入一些单词，回车换行输入新一行的单词。这些输入将作为示例程序的输入。如果要使得某个单词的计数大于1，请在5秒钟内重复输入相同的单词（如果5秒钟输入相同单词对你来说太快，可以把示例程序中的窗口时间调大）。


##Data Sources

Sources是程序的输入数据来源。你可以使用StreamExecutionEnvironment.addSource(sourceFunction)将一个Source添加到程序中。
Flink提供一系列预实现的数据源函数，用户可以通过实现SourceFunction接口来自定义非并行的Source，也可以通过实现ParallelSourceFunction接口或者继承RichParallelSourceFunction来自定义并行的Source。

StreamExecutionEnvironment提供了几个预先定义好的流式数据源，如下：

基于文件:

readTextFile(path) - 读取文本文件, 即符合TextInputFormat规范的文件，逐行地以字符串类型返回。

readFile(fileInputFormat, path) - 以指定分文件格式读取文件（一次）

readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo) - 以上两个方法实际上内部调用这个方法。
它根据给定的fileInputFormat和读取路径读取文件。
根据提供的watchType，这个source可以定期（每隔interval毫秒）监测给定路径的新数据（FileProcessingMode.PROCESS_CONTINUOUSLY），或者处理一次路径对应文件的数据并退出（FileProcessingMode.PROCESS_ONCE）。
你可以通过pathFilter进一步排除掉需要处理的文件。

实现:

在具体实现上，Flink文件读取过程分为两个子任务，目录监控和数据读取。
分别以一个独立实体执行。
目录监控任务是单线程的而数据读取任务是多线程并行的。后者的并行度等于job的并行度。
监控任务的作用是扫描目录（间歇或者一次，由watchType决定），查找需要处理的文件并进行分片（split），并将分片分配给下游reader。
Reader负责实际读取数据，每一个分片指只会被一个reader读取，一个reader可以逐个读取多个分片。

重要提示:

如果watchType设置为FileProcessingMode.PROCESS_CONTINUOUSLY，当一个文件被修改，它的内容会被整个重新处理，这会打破"exactly-once"语义，因为在文件末尾附加数据将导致其所有内容被重新处理。

如果watchType设置为FileProcessingMode.PROCESS_ONCE，source只会扫描文件一次并退出，不去等待reader完成内容读取。
当然，reader会持续读取完整个文件，关闭source不会引起更多的检查点。这会导致如果一个节点失败需要更长的时间恢复，因为任务会从最后一个检查点开始重新执行。

基于Socket:

socketTextStream -  从socket读取。元素可以用分隔符切分。

基于集合:

fromCollection(Collection) - 从Java的Java.util.Collection创建数据流。集合中的所有元素类型必须相同。

fromCollection(Iterator, Class) - 从一个迭代器中创建数据流。Class指定了该迭代器返回元素的类型。

fromElements(T ...) - 从给定的对象序列中创建数据流。所有对象类型必须相同。

fromParallelCollection(SplittableIterator, Class) - 从一个迭代器中创建并行数据流。Class指定了该迭代器返回元素的类型。

generateSequence(from, to) - 创建一个生成指定区间范围内的数字序列的并行数据流。

自定义:

addSource - 例如，你可以addSource(new FlinkKafkaConsumer08<>(...))以从Apache Kafka读取数据。


##DataStream Transformations

请参看[Operators]来了解流转化的概述。


##Data Sinks

Data Sinks消费数据流并将它们推到文件，socket，外部系统或者打印。Flink有各种内置的输出格式，封装在数据流操作中。

writeAsText() / TextOutputFormat - 将元素以字符串形式输出。字符串通过调用每个元素的toString()方法获得。

writeAsCsv(...) / CsvOutputFormat - 将元素以逗号分割的csv文件输出，行和域的分割符可以配置。每个域的值通过调用每个元素的toString()方法获得。

print() / printToErr() - 打印每个元素的toString()值到标准输出/错误输出流。可以配置前缀信息添加到输出，以区分不同print的结果。如果并行度大于1，则task id也会添加到输出前缀上。

writeUsingOutputFormat() / FileOutputFormat - 自定义文件输出的方法/基类。支持自定义的对象到字节的转换。

writeToSocket - 根据SerializationSchema把元素写到socket

addSink - 调用自定义sink function。Flink自带了很多连接其他系统的连接器（connectors）（如Apache Kafka），这些连接器都实现了sink function。

请注意，write*（）方法主要是用于调试的。它们不参与Flink的检查点机制，这意味着这些规范方法具有"at-least-once"语义。数据刷新到目标系统取决于OutputFormat的实现。这意味着并非所有发送到OutputFormat的元素都会立即在目标系统中可见。此外，在失败的情况下，这些记录可能会丢失。

为了可靠，在把流写到文件系统时，使用flink-connector-filesystem来实现exactly-once。此外，通过.addSink(...)方法自定义的实现可以参与Flink的检查点机制以实现exactly-once语义。


##迭代 

迭代流程序实现一个step方法并将其嵌入到IterativeStream。因为一个数据流程序永远不会结束，因此没有最大迭代次数。
事实上，你需要指定流的哪一部分是用于回馈到迭代过程以及哪一部分会通过切片或者过滤向下游发送。这里我们展示一个使用过滤的例子。首先我们定义一个IterativeStream。


```
IterativeStream<Integer> iteration = input.iterate();
```

然后，我们要使用一系列transformation来设定循环中会执行的逻辑。这里我们只是用一个map转换。

```
DataStream<Integer> iterationBody = iteration.map(/* 这会执行多次 */);
```

要关闭迭代并定义迭代尾部，需要调用IterativeStream的closeWith(feedbackStream)方法。
传给closeWith方法的数据流将被反馈给迭代的头部。
一种常见的模式是使用filter来分离流中需要反馈的部分和需要继续发往下游的部分。
这些filter可以定义“终止”逻辑，以控制元素是流向下游而不是反馈迭代。

```
iteration.closeWith(iterationBody.filter(/* 流的一部分 */));
DataStream<Integer> output = iterationBody.filter(/* 流的其他部分 */);
```

例如，下面的程序是持续的从一系列整数中减1直到0：

```
DataStream<Long> someIntegers = env.generateSequence(0, 1000);

IterativeStream<Long> iteration = someIntegers.iterate();

DataStream<Long> minusOne = iteration.map(new MapFunction<Long, Long>() {
  @Override
  public Long map(Long value) throws Exception {
    return value - 1 ;
  }
});

DataStream<Long> stillGreaterThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value > 0);
  }
});

iteration.closeWith(stillGreaterThanZero);

DataStream<Long> lessThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value <= 0);
  }
});

```



##执行参数

StreamExecutionEnvironment包含一个ExecutionConfig属性用于给任务配置指定的运行时参数。

更多参数的解释请参阅[执行参数]。一些参数是只应用于DataStream API的：

setAutoWatermarkInterval(long milliseconds): 设置自动发射水印的间隔。你可以通过getAutoWatermarkInterval()获取当前的发射间隔。


##容错


[状态和检查点] 描述了如何开启和配置Flink的checkpointing机制。


##延迟控制

默认情况下，元素不会通过一个一个通过网络传输（这样会带来不必要的通信），而是缓冲起来。缓冲（实际是在机器之间传输）的大小可以在Flink配置文件中设置。
尽管这种方法有利于优化吞吐量，但在输入流不够快的情况下会造成延迟问题。为了控制吞吐量和延迟，你可以在execution environment（或单个operator）上使用env.setBufferTimeout(timeoutMillis)来设置缓冲区填满的最大等待时间。
如果超过该最大等待时间，即使缓冲区未满，也会被自动发送出去。该最大等待时间默认值为100 ms。


使用:

*Java*
```
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.generateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
```

*Scala*
```
val env: LocalStreamEnvironment = StreamExecutionEnvironment.createLocalEnvironment
env.setBufferTimeout(timeoutMillis)

env.generateSequence(1,10).map(myMap).setBufferTimeout(timeoutMillis)
```

为了最大化吞吐量，可以设置setBufferTimeout(-1)，这样就没有了超时机制，缓冲区只有在满时才会发送出去。为了最小化延迟，可以把超时设置为接近0的值（例如5或10 ms）。应避免将该超时设置为0，因为这样可能导致性能严重下降。


##调试

在分布式集群在运行数据流程序之前，最好确保程序可以符合预期工作。因此，实现数据分析程序通常需要一个渐进的过程：检查结果，调试和改进。

Flink提供了诸多特性来大幅简化数据分析程序的开发：你可以在IDE中进行本地调试，注入测试数据，收集结果数据。本节给出一些如何简化Flink程序开发的指导。

###本地执行环境

LocalStreamEnvironment会在其所在的JVM进程中启动一个Flink引擎. 如果你在IDE中启动LocalEnvironment，你可以在你的代码中设置断点，轻松调试你的程序。

一个LocalEnvironment的创建和使用示例如下：

*Java*
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

DataStream<String> lines = env.addSource(/* some source */);
// build your program

env.execute();
```


*Scala*

```
valval  envenv  ==  StreamExecutionEnvironmentStrea .createLocalEnvironment()

val lines = env.addSource(/* some source */)
// build your program

env.execute()
```

###基于集合的数据Sources

Flink提供了基于Java集合实现的特殊数据sources用于测试。一旦程序通过测试，它的sources和sinks可以方便的替换为从外部系统读写的sources和sinks。

基于集合的数据Sources可以像这样使用：

*Java*
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// Create a DataStream from a list of elements
DataStream<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataStream from any Java collection
List<Tuple2<String, Integer>> data = ...
DataStream<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataStream from an Iterator
Iterator<Long> longIt = ...
DataStream<Long> myLongs = env.fromCollection(longIt, Long.class);
```

*Scala*
```
valval  envenv  ==  StreamExecutionEnvironmentStrea .createLocalEnvironment()

// Create a DataStream from a list of elements
val myInts = env.fromElements(1, 2, 3, 4, 5)

// Create a DataStream from any Collection
val data: Seq[(String, Int)] = ...
val myTuples = env.fromCollection(data)

// Create a DataStream from an Iterator
val longIt: Iterator[Long] = ...
val myLongs = env.fromCollection(longIt)
```
*注意*：
目前，集合数据source要求数据类型和迭代器实现Serializable。并行度 = 1）。

###迭代的数据Sink

Flink还提供了一个sink来收集DataStream的测试和调试结果。它可以这样使用：

*Java*
```
import org.apache.flink.streaming.experimental.DataStreamUtils

DataStream<Tuple2<String, Integer>> myResult = ...
Iterator<Tuple2<String, Integer>> myOutput = DataStreamUtils.collect(myResult)
```

*Scala*
```
import org.apache.flink.streaming.experimental.DataStreamUtils
import scala.collection.JavaConverters.asScalaIteratorConverter

val myResult: DataStream[(String, Int)] = ...
val myOutput: Iterator[(String, Int)] = DataStreamUtils.collect(myResult.javaStream).asScala
```

*注意：*
Flink 1.5.0的flink-streaming-contrib模块已经移除，它的类被迁移到了flink-streaming-java和flink-streaming-scala。


#下一步

Operators 算子: 流式算子的专门介绍。
Event Time 事件时间: Flink时间概念介绍。
State & Fault Tolerance 状态和容错: 如何开发有状态的应用。
Connectors 连接器: 可使用的输入输出连接器的描述。



