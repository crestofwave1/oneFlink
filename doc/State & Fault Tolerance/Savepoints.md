# 保存点

## 概述
保存点是外部存储的自包含检查点，可用于停止、恢复、更新 Flink 程序。 用户通过 Flink 的检查点机制来创建流程序状态的（非增量）快照，并将检查点数据和元数据写入外部文件系统。

本小节介绍了保存点的触发、恢复和处理所涉及的步骤。 有关 Flink 如何处理状态和故障的更多详细信息，请查看 Streaming Programs 页面中的 State。

注意：为了能够在不同作业版本和 不同Flink 版本之间顺利升级，请务必查看assigning IDs to your operators相关内容 (https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html#assigning-operator-ids)的部分。

## 给算子分配 ID

为了将来能够升级你的程序，强烈建议用户根据本节内容做一些修改。 必须更改是通过该uid(String)方法手动指定算子 ID从而确定每个算子的状态。
```
DataStream<String> stream = env.
  // Stateful source (e.g. Kafka) with ID
  .addSource(new StatefulSource())
  .uid("source-id") // ID for the source operator
  .shuffle()
  // Stateful mapper with ID
  .map(new StatefulMapper())
  .uid("mapper-id") // ID for the mapper
  // Stateless printing sink
  .print(); // Auto-generated ID
```

如果用户没有给个算子分配 ID，则Flik会根据程序的结构自动给每个算子生成一个ID。 只要这些 ID 不变，用户就可以从保存点自动恢复。因此，强烈建议手动分配这些 ID。

## 保存点状态
用户可以将保存点视为Operator ID 到 每个有状态算子的映射：
```
Operator ID | State
------------+------------------------
source-id   | State of StatefulSource
mapper-id   | State of StatefulMapper

```

在上面的例子中，打印的sink是无状态的，因此包含在保存点状态中。 默认情况下，会将每一个保存点映射回新程序。

## 算子
用户可以使用命令行客户端触发保存点，取消带有保存点的作业，从保存点恢复和处理保存点。

Flink> = 1.2.0版本可以使用 WebUI 从保存点恢复作业
### 触发保存点
触发保存点时，会创建一个新的保存点目录来存储数据和元数据。 用户可以通过修改默认配置或使用触发器命令指定存储路径（请参阅:targetDirectory参数）。

注意：存储路径必须是 JobManager（s）和 TaskManager（例如分布式文件系统上的位置）可访问的位置。
例如，使用FsStateBackend或RocksDBStateBackend：
```
# Savepoint target directory
/savepoints/

# Savepoint directory
/savepoints/savepoint-:shortjobid-:savepointid/

# Savepoint file contains the checkpoint meta data
/savepoints/savepoint-:shortjobid-:savepointid/_metadata

# Savepoint state
/savepoints/savepoint-:shortjobid-:savepointid/...
```

注意： 虽然看起来好像可以移动保存点，但由于_metadata文件中的绝对路径，目前无法进行保存。 请按照 FLINK-5778 取消此限制。
请注意，如果使用MemoryStateBackend，则元数据和保存点状态将存储在_metadata文件中。所以用户可以移动文件并从任何位置恢复。

#### 触发保存点
```
$ bin/flink savepoint :jobId [:targetDirectory]
```
上述命令将会为jobid 触发一个保存点，并返回创建的保存点的路径。 用户需要此路径来还原和部署保存点。

### 使用 YARN 触发保存点
```
$ bin/flink savepoint :jobId [:targetDirectory] -yid :yarnAppId
```
上述命令将会为ID为jobId的job 和ID为yarnAppId的YARN application 触发一个保存点，并返回创建的保存点的路径。

#### 取消保存点的作业
```
$ bin/flink cancel -s [:targetDirectory] :jobId
```

上述命令将会为触发具有 ID 的作业的保存点:jobid，并取消作业。 此外，用户可以指定存储保存点的路径。该路径必须是 JobManager 和 TaskManager 可访问的。

## 从保存点恢复
```
$ bin/flink run -s :savepointPath [:runArgs]
```

上述命令将会提交作业并指定要从中恢复的保存点。 用户可以指定保存点目录或_metadata文件的路径。

### 允许未恢复状态
默认情况下，resume 操作将尝试将保存点的所有状态映射回要恢复的程序。 如果删除了算子，则可以通过--allowNonRestoredState（short -n:) 选项跳过无法映射到新程序的状态：
```
$ bin/flink run -s :savepointPath -n [:runArgs]
```

## 处理保存点

```
$ bin/flink savepoint -d :savepointPath
```

上述命令将会删除存储在savepointPath的保存点。

请注意，可以通过常规文件系统操作手动删除保存点，而不会影响其他保存点或检查点（请记住，每个保存点都是自包含的）。  Flink 1.2以上版本可以使用上面的 savepoint 命令执行。

## 配置

用户可以通过state.savepoints.dir密钥配置默认保存点存储路径。 触发保存点时，保存点的元数据信息将会保存到该路径。 用户可以通过使用下面的命令指定存储路径来（请参阅:targetDirectory参数）。
```
# Default savepoint target directory
state.savepoints.dir: hdfs:///flink/savepoints
```
如果既未配置缺省值也未指定保存点存储路径，则触发保存点将失败。

注意：保存点存储路径必须是 JobManager（s）和 TaskManager（例如分布式文件系统上的位置）可访问的位置。
## 常问问题

### 我应该为所有的算子分配 ID 吗？

根据经验，是这样的。 严格地说，使用uid方法将 ID 分配给作业中的有状态算子就足够了，这样保存点就会仅包含有状态的算子，无状态的算子不是保存点的一部分。

在实际中，建议给所有算子分配ID，因为 Flink 的一些内置算子（如 Window 算子）也是有状态的，并且不清楚哪些内置算子实际上是有状态的，哪些不是。 如果用户完全确定该算子是无状态的，则可以跳过该uid方法。

### 如果我在工作中添加一个需要状态的新算子会怎么样？

当用户向作业添加新算子时，它将初始化为没有保存任何状态。保存点包含每个有状态算子的状态。 无状态算子不在保存点的范围内。 新算子的类似于无状态算子。

### 如果我删除一个状态的算子会怎么样？

默认从保存点恢复的时候，会尝试恢复所有的状态。从一个包含了被删除算子的状态的保存点进行作业恢复将会失败。
用户可以通过使用 run 命令设置--allowNonRestoredState（short :) 跳过从保存点（savepoint）进行恢复作业：
```
$ bin/flink run -s :savepointPath -n [:runArgs]
```

### 如果我在工作中重新排序有状态算子会怎么样？

如果用户为这些算子分配了 ID，它们将照常恢复。
如果用户没有分配 ID，则重新排序后，有状态算子的自动生成 ID 很可能会更改。 这将导致用户无法从以前的保存点恢复。

### 如果我添加，删除或重新排序在我的工作中没有状态的算子会怎么样？

如果为有状态算子分配了 ID，则无状态算子不会影响保存点还原。
如果用户没有分配 ID，则重新排序后，有状态算子的自动生成 ID 很可能会更改。 这将导致用户从保存点恢复失败。

### 当我在恢复时更改程序的并行性时会怎么样？

如果使用 Flink 1.2.0以上版本触发保存点，并且没有使用 Checkpointed 等以及过期的 API，则只需从保存点恢复程序并指定新的并行度即可。

如果从 低于1.2.0的版本触发的保存点或使用过期的 API，则首先必须将作业和保存点迁移到 Flink 1.2.0以上版本，才能更改并行度。 请参阅升级作业和 Flink 版本指南(http://flink.iteblog.com/ops/upgrading.html)。