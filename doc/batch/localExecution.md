## 0. 本地执行  
Flink可以运行在单台机器，甚至单台Java虚拟机里面，这样方便用户在本地去测试和调试Flink程序。   
本节概述了Flink的本地执行机制，本地环境和执行器允许您在本地Java虚拟机中运行Flink程序，或者在任何JVM中作为现有程序的一部分，大多数示例都可以通过单击IDE的“Run”按钮在本地启动。   

Flink支持两种不同类型的本地执行方式：      
LocalExecutionEnvironment启动在整个Flink运行时，它包括一个JobManager和一个TaskManager，以及内存管理和在集群模式下执行的所有内部算法。   
CollectionEnvironment可在Java集合上执行Flink程序，这种模式不会启动在完整的Flink运行时，因此执行的开销非常低，而且是轻量级的。
例如可以执行DataSet.map()-转换操作将会通过将map()函数应用于Java列表中的所有元素。



### 1. 调试

如果您在本地运行Flink程序，你也可以像调试其他Java程序一样调试你的程序。
可以使用System.out.println()写出一些内部变量，或者使用调试器。
还可以在map()、reduce()和所有其他方法中设置断点。有关Java API中测试和本地调试工具，请参阅Java API文档中的调试部分。

### 2. Maven依赖
如果你在Maven项目中开发你的程序，你必须使用这个依赖项添加flink-clients模块:
```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients_2.11</artifactId>
  <version>1.6.0</version>
</dependency>
```

### 3. 本地环境
LocalEnvironment是Flink程序的本地执行句柄，使用它在本地JVM中运行程序——独立运行或嵌入到其他程序中。
本地环境通过方法ExecutionEnvironment.createLocalEnvironment()实例化。
默认情况下，它将使用与您的计算机具有CPU内核(硬件上下文)一样多的本地线程来执行。
您可以选择指定所需的并行度，本地环境也可以使用 enableLogging()或 disablelelogging()配置为是否输出日志到控制台。  
在大多数情况下，调用ExecutionEnvironment.getExecutionEnvironment()是更好的方式。当程序在本地(命令行窗口之外)启动时，该方法返回LocalEnvironment。
当[命令行窗口](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/cli.html)调用程序时，该方法返回预配置的集群执行环境。


```java
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

    DataSet<String> data = env.readTextFile("file:///path/to/file");

    data.filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("file:///path/to/result");

    JobExecutionResult res = env.execute();
}
```
执行完成后返回的JobExecutionResult对象包含程序运行时和累加器结果，LocalEnvironment还允许将自定义配置值传递给Flink。

```
Configuration conf = new Configuration();
conf.setFloat(ConfigConstants.TASK_MANAGER_MEMORY_FRACTION_KEY, 0.5f);
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment(conf);
```

_注意:本地执行环境不会启动任何web前端来监视执行。_


### 4. 集合环境
使用CollectionEnvironment在Java集合上执行是运行Flink程序的一个低开销方法，这种模式的典型用例是自动测试、调试和代码重用。
用户可以使用为批处理而实现的算法，也可以使用更具交互性的情况。Flink程序稍作修改的变体可以在Java应用服务器中用于处理发来的请求。


**4.1 基于集合执行的框架**


```java
public static void main(String[] args) throws Exception {
    // initialize a new Collection-based execution environment
    final ExecutionEnvironment env = new CollectionEnvironment();

    DataSet<User> users = env.fromCollection( /* get elements from a Java Collection */);

    /* Data Set transformations ... */

    // retrieve the resulting Tuple2 elements into a ArrayList.
    Collection<...> result = new ArrayList<...>();
    resultDataSet.output(new LocalCollectionOutputFormat<...>(result));

    // kick off execution.
    env.execute();

    // Do some work with the resulting ArrayList (=Collection).
    for(... t : result) {
        System.err.println("Result = "+t);
    }
}
```
flink-examples-batch模块包含一个完整的示例，称为CollectionExecutionExample。   
请注意基于集合的Flink程序的执行只能在适合JVM堆的小数据集上进行，集合上的执行不是多线程的，仅使用一个线程。

