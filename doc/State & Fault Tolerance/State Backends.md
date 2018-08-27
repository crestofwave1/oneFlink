## 状态后端

用Data Stream API编写的程序状态

通常以各种形式保存：

* Windows 会在触发元素或聚合之前收集元素或聚合
* 转换函数可以使用键 / 值状态接口来存储值
* 转换函数可以实现CheckpointedFunction接口以使其局部变量具有容错能力

另请参阅流 API 指南中的[State & Fault Tolerance](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/index.html)。

激活检查点时，检查点会持续保持此类状态，以防止数据丢失并始终如一地恢复。 国家如何在内部表示，以及在检查点上如何以及在何处持续取决于所选择的州后端。
### 可用的状态后端
开箱即用，Flink 捆绑了这些状态后端：

* MemoryStateBackend
* FsStateBackend
* RocksDBStateBackend

如果没有配置其他任何内容，系统将使用 MemoryStateBackend。

#### MemoryStateBackend
该 MemoryStateBackend 保存数据在内部作为 Java 堆的对象。 键 / 值状态和窗口运算符包含存储值，触发器等的哈希表。

在检查点时，此状态后端将对状态进行快照，并将其作为检查点确认消息的一部分发送到 JobManager（主服务器），JobManager 也将其存储在其堆上。

可以将 MemoryStateBackend 配置为使用异步快照。 虽然我们强烈建议使用异步快照来避免阻塞管道，但请注意，默认情况下，此功能目前处于启用状态。 要禁用此功能，用户可以MemoryStateBackend在构造函数中将相应的布尔标志实例化为false（这应该仅用于调试），例如：
```
new MemoryStateBackend(MAX_MEM_STATE_SIZE, false);
```
    
MemoryStateBackend 的局限性：

* 默认情况下，每个州的大小限制为 5 MB。 可以在 MemoryStateBackend 的构造函数中增加此值。
* 无论配置的最大状态大小如何，状态都不能大于 akka 帧大小（请参阅[配置](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/config.html)）。
* 聚合状态必须适合 JobManager 内存。
鼓励 MemoryStateBackend 用于：

* 本地开发和调试
* 几乎没有状态的作业，例如仅包含一次记录功能的作业（Map，FlatMap，Filter，...）。 卡夫卡消费者需要很少的国家。

#### FsStateBackend

所述 FsStateBackend 配置有文件系统 URL（类型，地址，路径），如 “HDFS：// 名称节点：40010 / 弗林克 / 检查点” 或“文件：/// 数据 / 弗林克 / 检查点”。

FsStateBackend 将正在运行的数据保存在 TaskManager 的内存中。 在检查点时，它将状态快照写入配置的文件系统和目录中的文件。 最小元数据存储在 JobManager 的内存中（或者，在高可用性模式下，存储在元数据检查点中）。

FsStateBackend 默认使用异步快照，以避免在编写状态检查点时阻塞处理管道。 要禁用此功能，用户可以FsStateBackend在构造函数集中使用相应的布尔标志来实例化 a false，例如：

```
 new FsStateBackend(path, false);
```   
鼓励 FsStateBackend：

* 具有大状态，长窗口，大键 / 值状态的作业。
* 所有高可用性设置。

#### RocksDBStateBackend

所述 RocksDBStateBackend 配置有文件系统 URL（类型，地址，路径），如 “HDFS：// 名称节点：40010 / 弗林克 / 检查点” 或“文件：/// 数据 / 弗林克 / 检查点”。

RocksDBStateBackend 将 [RocksDB](https://rocksdb.org/) 数据库中的飞行中数据保存在（默认情况下）存储在 TaskManager 数据目录中。 在检查点时，整个 RocksDB 数据库将被检查点到配置的文件系统和目录中。 最小元数据存储在 JobManager 的内存中（或者，在高可用性模式下，存储在元数据检查点中）。

RocksDBStateBackend 始终执行异步快照。

RocksDBStateBackend 的局限性：

* 由于 RocksDB 的 JNI 桥接 API 基于 byte []，因此每个密钥和每个值的最大支持大小为 2 ^ 31 个字节。 重要提示：在 RocksDB 中使用合并操作的状态（例如 ListState）可以静默地累积 > 2 ^ 31 字节的值大小，然后在下次检索时失败。 这是目前 RocksDB JNI 的一个限制。

我们鼓励 RocksDBStateBackend：

* 具有非常大的状态，长窗口，大键 / 值状态的作业。
* 所有高可用性设置。

请注意，您可以保留的状态量仅受可用磁盘空间量的限制。 与将状态保持在内存中的 FsStateBackend 相比，这允许保持非常大的状态。 但是，这也意味着使用此状态后端可以实现的最大吞吐量更低。 对此后端的所有读 / 写都必须通过去 / 序列化来检索 / 存储状态对象，这比使用堆基表示正在进行的堆上表示更昂贵。

RocksDBStateBackend 是目前唯一提供增量检查点的后端（见这里[Tuning Checkpoints and Large State](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/large_state_tuning.html)）。

### 配置状态后端

如果您不指定任何内容，则默认状态后端是作业管理器。 如果要为群集上的所有作业建立不同的默认值，可以通过在 flink-conf.yaml 中定义新的默认状态后端来实现。 可以基于每个作业覆盖默认状态后端，如下所示。

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
可以flink-conf.yaml使用配置密钥在配置中配置默认状态后端state.backend。

config 条目的可能值包括 jobmanager（MemoryStateBackend），filesystem（FsStateBackend），rocksdb（RocksDBStateBackend），或实现状态后端工厂 [FsStateBackendFactory](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/checkpoints.html#directory-structure) 的类的完全限定类名，例如org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory RocksDBStateBackend。

该state.checkpoints.dir选项定义所有后端写入检查点数据和元数据文件的目录。 您可以[目录结构](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/checkpoints.html#directory-structure)到有关检查点目录结构的更多详细信息。

配置文件中的示例部分可能如下所示：
```
# The backend that will be used to store operator state checkpoints

state.backend: filesystem


# Directory for storing checkpoints

state.checkpoints.dir: hdfs://namenode:40010/flink/checkpoints
```