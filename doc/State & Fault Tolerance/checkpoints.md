# 检查点

## 概述

检查点在允许其状态和对应的流位置可恢复时，可使状态在Flink中具有容错性，从而使应用程序具有与正常运行时相同的语义。

有关如何为程序启用和配置检查点的信息，请参阅[Checkpointing](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/checkpointing.html)。

## 保留检查点

默认情况下，检查点仅用于恢复失败的job，并且不会保留。程序运行结束时会删除它们。但是，可以配置要保留的定期检查点。通过配置配置文件，可以在job失败或程序运行结束时保留的检查点。通过保留的检查点恢复失败的job。

```
CheckpointConfig config = env.getCheckpointConfig();
config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```
该ExternalizedCheckpointCleanup模式配置取消job时检查点发生的情况：
* ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION：取消job时保留检查点。请注意，在这种情况下，必须在取消后手动清理检查点状态。
* ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION：取消job时删除检查点。只有在job失败时，检查点状态才可用。

## 目录结构

与[保存点](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html)类似，检查点由元数据文件和一些其他数据文件组成，具体取决于state backend。 元数据文件和数据文件存储在state.checkpoints.dir配置文件所指定的存储路径中，也可以在代码中为每个job指定存储路径。

### 通过配置文件全局配置
```
state.checkpoints.dir: hdfs:///checkpoints/
```
### 在构造state backend时为每个job配置
```
env.setStateBackend(new RocksDBStateBackend("hdfs:///checkpoints-data/");
```
## 与保存点的差异

检查点与[保存点](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html)有一些差异。
* 使用state backend特定的（低级）数据格式，可以是增量式的。
* 不支持 Flink 的某些特定功能，如rescaling。

## 从保留的检查点恢复

通过使用检查点的元数据文件，可以从检查点恢复失败的job，就像从保存点恢复一样（请参阅[保存点恢复指南](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/cli.html#restore-a-savepoint)）。请注意，如果元数据文件不是存储在默认路径，则jobmanager需要访问state.checkpoints.dir配置文件所指定的元数据文件的存储路径（请参阅[目录结构](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/checkpoints.html#directory-structure) ）。
```
$ bin/flink run -s :checkpointMetaDataPath [:runArgs]
```
