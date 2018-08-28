## 状态后端

用Data Stream API编写的程序状态通常以各种形式保存：

* Windows 会在收集元素或聚合之前触发
* 转换函数可以使用键值对状态接口来存储值
* 转换函数可以通过CheckpointedFunction接口确保局部变量的容错性

详情请参阅流 API 指南中的[State & Fault Tolerance](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/index.html)。

激活检查点时，检查点会持续保持此类状态，以防止数据丢失并始终如一地恢复。这个状态如何在内部表示，如何保持以及在何处保持取决于所选择的状态后端。

### 可用的状态后端

Flink 捆绑了这些状态后端：

* MemoryStateBackend
* FsStateBackend
* RocksDBStateBackend

如果没有修改默认配置，系统将使用MemoryStateBackend。

#### MemoryStateBackend

MemoryStateBackend将数据作为Java堆栈栈的对象在内部保存。键/值对状态和窗口算子包含有存储值和触发器等的哈希表。

在检查点上，此状态后端将对状态进行快照，并将其作为检查点确认消息的一部分发送到 JobManager（master），JobManager也将其存储在其堆栈上。

内存状态后端可以被配置为使用异步快照。尽管我们强烈鼓励使用异步快照来避免阻塞管道，但是请注意，目前默认情况下启用了此功能。为了禁用该特性，用户可以在构造函数中将对应的布尔标志设置为 false（这应该只用于调试）来实例化 MemoryStateBackend，例如：
```
new MemoryStateBackend(MAX_MEM_STATE_SIZE, false);
```
    
MemoryStateBackend 的局限性：

* 默认情况下，每个状态的大小限制为 5 MB。 可以在 MemoryStateBackend 的构造函数中增加此值。
* 无论配置的最大状态大小如何，状态都不能大于akka帧的大小（请参阅[配置](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/config.html)）。
* 聚合状态必须适合 JobManager 内存。

建议MemoryStateBackend 用于：

* 本地开发和调试
* 状态很少的作业，例如仅包含一次记录功能的作业（Map，FlatMap，Filter，...），卡夫卡的消费者需要很少的状态。

#### FsStateBackend

该FsStateBackend 配置有文件系统URL（类型，地址，路径），如 “hdfs://namenode:40010/flink/checkpoints” 或者 “file:///data/flink/checkpoints”.

FsStateBackend 将正在运行的数据保存在 TaskManager 的内存中。 在检查点时，它将状态快照写入配置的文件系统目录中。 最小元数据存储在 JobManager 的内存中（另外在高可用性模式下，存储在元数据检查点中）。

默认情况下，FsStateBackend 使用异步快照，以避免在编写状态检查点时阻塞处理管道。要禁用该特性，用户可以在构造函数中将相应的布尔标志设置为 false 来实例化 FsStateBackend，例如：

```
 new FsStateBackend(path, false);
```   
建议FsStateBackend：

* 具有大状态，长窗口，大键 / 值状态的作业。
* 所有高可用性设置。

#### RocksDBStateBackend

该RocksDBStateBackend 配置有文件系统 URL（类型，地址，路径），如 “hdfs://namenode:40010/flink/checkpoints” 或者 “file:///data/flink/checkpoints”.

RocksDBStateBackend 将 [RocksDB](https://rocksdb.org/) 数据库中正在运行的数据保存在（默认情况下） TaskManager 数据目录中。 在检查点上，整个RocksDB数据库将被存储到检查点配置的文件系统和目录中。最小元数据存储在 JobManager 的内存中（在高可用性模式下，存储在元数据检查点中）。

RocksDBStateBackend 始终执行异步快照。

RocksDBStateBackend 的局限性在于：

* 由于RocksDB的JNI桥接API基于byte[]，因此每个密钥和每个值的最大支持大小为2^31个字节。 

  重要提示：在RocksDB中使用合并操作的状态（例如ListState）可以静默地累积 > 2^31字节的值大小，然后在下次检索时失败。 这是目前RocksDB JNI的一个限制。

我们建议RocksDBStateBackend：

* 具有非常大的状态，长窗口，大键 / 值状态的作业。
* 所有高可用性设置。

请注意，可以保留的状态量仅受可用磁盘空间量的限制。与将状态保持在内存中的 FsStateBackend 相比，这允许保持非常大的状态。 但是，这也意味着使用此状态后端可以实现的最大吞吐量更低。 对此后端的所有 读/写 都必须通过反/序列化来检索/存储状态对象，这比像基于堆栈栈的后端那样始终使用堆栈栈表示更要好。。

RocksDBStateBackend 是目前唯一提供增量检查点的后端（详情见[Tuning Checkpoints and Large State](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/large_state_tuning.html)）。

### 配置状态后端

如果没有修改默认配置，则默认状态后端是作业管理器。 如果要为群集上的所有作业建立不同的默认值，可以通过在 flink-conf.yaml 中定义新的默认状态后端来实现。 可以基于每个作业覆盖默认状态后端，如下所示。

#### 设置每个作业状态后端

每个作业状态后端StreamExecutionEnvironment在作业上设置，如下例所示：

```java
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"));
```
```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"))
```

#### 设置默认状态后端


默认的状态后端可以在flink-conf.yaml中通过使用配置密钥state.backend来进行配置

配置内容的可能包括 jobmanager（MemoryStateBackend），filesystem（FsStateBackend），rocksdb（RocksDBStateBackend），或实现 [FsStateBackendFactory](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/checkpoints.html#directory-structure) 的类的完全限定类名，例如RocksDBStateBackend的org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory。

该state.checkpoints.dir选项定义所有后端写入检查点数据和元数据文件的目录。详情请见[目录结构](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/checkpoints.html#directory-structure)。

配置文件中的示例部分可能如下所示：
```
# The backend that will be used to store operator state checkpoints

state.backend: filesystem

# Directory for storing checkpoints

state.checkpoints.dir: hdfs://namenode:40010/flink/checkpoints
```