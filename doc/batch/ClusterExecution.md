##  集群执行
Flink程序可以分布运行在许多机器的集群上，将程序发送到集群执行有两种方式:

### 1. 命令行界面
命令行界面允许您将打包的程序(jar)提交到集群(或单机设置)。有关详细信息，请参阅[命令行界面](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/cli.html)文档。

### 2. 远程环境
远程环境允许您直接在集群上执行Flink Java程序，远程环境指向您希望在其上执行程序的集群。

### 3. Maven Dependency
如果您正在将您的程序开发成Maven项目，那么您必须使用这个依赖项添加flink-clients模块:
```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients_2.11</artifactId>
  <version>1.6.0</version>
</dependency>
```

示例，下面说明了远程环境的使用:

```java
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment
        .createRemoteEnvironment("flink-master", 8081, "/home/user/udfs.jar");

    DataSet<String> data = env.readTextFile("hdfs://path/to/file");

    data
        .filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("hdfs://path/to/result");

    env.execute();
}
```
注意，该程序包含用户定义的代码，因此需要一个JAR文件，其中附带代码的类。远程环境的构造函数获取到JAR文件的路径。