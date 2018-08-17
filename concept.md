* [Flink是什么](doc/quickstark/what-is-flink.md)
  * quickstark
    * [简介](doc/quickstark/what-is-flink.md)
    * [Flink standalone 安装部署](doc/quickstark/FlinkDeploy.md)
    * [Flink on yarn 安装部署](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/deployment/yarn_setup.html)
    * [高可用 HA](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/jobmanager_high_availability.html)

    * [快速构建java工程](doc/quickstark/CreateJavaProject.md)
    
  * [基本API介绍](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/api_concepts.html)
  * batch
    * [概览 认领 by _coderrr 翻译中](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/)
    * [transformations](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/dataset_transformations.html)
    * [Fault Tolerance](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/fault_tolerance.html)
    * [Iterations](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/iterations.html)
    * [Zipping Elements](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/zip_elements_guide.html)
    * [Connectors](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/connectors.html)
    * [Hadoop Compatibility](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/batch/hadoop_compatibility.html)
    * [本地执行](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/local_execution.html)
    * [集群执行](doc/batch/ClusterExecution.md)
  * streaming
      * [概览]（认领 by wayblink,翻译中）(https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/datastream_api.html)
      * 事件时间
          * [事件时间](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/event_time.html)
          * [产生时间戳和水印](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/event_timestamps_watermarks.html)
          * [预定义时间戳抽取器和水印发舍器](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/event_timestamp_extractors.html)
      * State & Fault Tolerance
          * [State & Fault Tolerance](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/)
          * [Working with State](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/state.html)
          * [The Broadcast State Pattern](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/broadcast_state.html)   
          * [Checkpointing](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/checkpointing.html)
          * [Queryable State](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/queryable_state.html)
          * [State Backends](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/state_backends.html)
          * [Custom Serialization](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/custom_serialization.html)
      * Operators
        * [Operators](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/)   
        * [windows](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html)
        * [Joining](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/joining.html)
        * [Process Function](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/process_function.html)
        * [Asynchronous I/O](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/asyncio.html)
      * Streaming Connectors
        * [Streaming Connectors](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/connectors/)
        * [Fault Tolerance Guarantees](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/connectors/guarantees.html)
        * [kafka](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/connectors/kafka.html)
        * [es](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/connectors/elasticsearch.html)
        * [HDFS Connector](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/connectors/filesystem_sink.html)
      * Side Outputs
        * [Side Outputs](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/side_output.html)
        * []()
  * table
    * [概览](doc/table/TableOverview.md)
    * [概念 & 常用API](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/common.html)
    * [Streaming Concepts](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/streaming.html)   
    * [Connect to External Systems](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/connect.html)
    * [Table API](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/tableApi.html)
    * [SQL](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/sql.html)
    * [User-defined Sources & Sinks](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/sourceSinks.html)
    * [User-defined Functions](doc/table/UDF.md)   
    * [SQL Client Beta](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/sqlClient.html)
  * Data Types & Serialization
    * [overview](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/types_serialization.html)
    * [Custom Serializers](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/custom_serializers.html)
  * Managing Execution   
    * [Execution Configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/execution_configuration.html)
    * [Program Packaging](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/packaging.html)
    * [Parallel Execution](doc/ManagExecution/ParallelExecution.md)
    * [Execution Plans](doc/ManagExecution/ExecutionPlans.md)
    * [Restart Strategies](doc/ManagExecution/RestartStrategies.md)   

  * CEP
    * [Event Processing (CEP)](doc/CEP/FlinkCEPOfficeWeb.md)

  * State & Fault Tolerance
    * [checkpoints](认领 by heitao,翻译中)(doc/State & Fault Tolerance/checkpoints.md)
    * [savepoints](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html)
    * [State Backends](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/state_backends.html)   
    * [Tuning Checkpoints and Large State](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/large_state_tuning.html)

  * Debugging & Monitoring
    * [Metrics](https://ci.apache.org/projects/flink/flink-docs-release-1.6/monitoring/metrics.html)    
    * [logging](https://ci.apache.org/projects/flink/flink-docs-release-1.6/monitoring/logging.html)
    * [historyserver](https://ci.apache.org/projects/flink/flink-docs-release-1.6/monitoring/historyserver.html)
    * [Monitoring Checkpointing](https://ci.apache.org/projects/flink/flink-docs-release-1.6/monitoring/checkpoint_monitoring.html)   
    * [Monitoring Back Pressure](https://ci.apache.org/projects/flink/flink-docs-release-1.6/monitoring/back_pressure.html)
    * [Monitoring REST API](https://ci.apache.org/projects/flink/flink-docs-release-1.6/monitoring/rest_api.html)
    * [Debugging Windows & Event Time](https://ci.apache.org/projects/flink/flink-docs-release-1.6/monitoring/debugging_event_time.html)
    * [Debugging Classloading](https://ci.apache.org/projects/flink/flink-docs-release-1.6/monitoring/debugging_classloading.html)
    * [Application Profiling](https://ci.apache.org/projects/flink/flink-docs-release-1.6/monitoring/application_profiling.html)   

  * 栗子
    * [batch](https://github.com/crestofwave1/flinkExamples/tree/master/src/main/java/Batch)
    * [streaming](https://github.com/crestofwave1/flinkExamples/tree/master/src/main/java/Streaming)
    * [table](https://github.com/crestofwave1/flinkExamples/tree/master/src/main/java/TableSQL)
    
    
  * 源码
    * ing
    
  * meetup分享
    * 2018.08.11 北京flink meetup
        链接:https://pan.baidu.com/s/1CLp2aFfef1M15Qauya4rCQ 密码:93mr
* TODO 欢迎继续补充
