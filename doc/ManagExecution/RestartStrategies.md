
# RestartStrategies

Flink支持不同的重启策略，可以控制在发生故障时如何重新启动作业。 可以使用默认重新启动策略启动集群，该策略在未定义任何特定的作业的重新启动策略时始终使用。 
如果使用重新启动策略提交作业，此策略将覆盖群集的默认设置。

## 1. 概览

Flink的默认重启策略可以通过flink-conf.yaml配置。前提是必须开启checkpoint，如果checkpoint未开启，将不会有执行策略这个概念。如果checkpoint被激活了同时重启策略没有配置，
那么固定延迟策略将使用Integer.MAX_VALUE作为重启尝试次数。

可以通过ExecutionEnvironment或者StreamExecutionEnvironment 调用setRestartStrategy设置重启策略。

下面举个例子，重启三次，每次间隔10s钟。
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
```

## 2. 重启策略详解

## 2.1 固定延迟重启策略

固定延迟重启策略用给定重启次数重启job。一旦执行次数达到了最大限制，job最终也会失败。在两次重拾中间有一个固定的时间间隔。

flink-conf.yaml里面默认配置了该策略。

```
restart-strategy: fixed-delay
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 10 s
```

restart-strategy.fixed-delay.attempts: 
默认值是1或者如果开启了checkpoint那么值是Integer.MAX_VALUE。
该配置就是控制job失败后尝试的最大次数。

restart-strategy.fixed-delay.delay: 
默认值是akka.ask.timeout或者开启checkpoint的话就是10s钟。

该变量控制两次尝试重启的时间间隔。

在配置文件里配置就比较固定，适合默认配置。如果每个应用程序想有自己的特殊的配置，也可以在代码里编程，如下：
```java

ExecutionEnvironmentExecu  env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
```

## 2.2 故障率重启策略

故障率重启策略在故障后重新启动作业，但是当超过故障率（每个时间间隔的故障）时，作业最终会失败。 在两次连续重启尝试之间，重启策略等待一段固定的时间。

通过在flink-conf.yaml中设置以下配置参数，默认启用此策略。

```
restart-strategy: failure-rate
restart-strategy.failure-rate.max-failures-per-interval: 3
restart-strategy.failure-rate.failure-rate-interval: 5 min
restart-strategy.failure-rate.delay: 10 s
```

* restart-strategy.failure-rate.max-failures-per-interval: 

在job宣告最终失败之前，在给定的时间间隔内重启job的最大次数。默认值是 1 。

* restart-strategy.failure-rate.failure-rate-interval: 

测量故障率的时间间隔。默认是1min。

* restart-strategy.failure-rate.delay: 

在两次连续重试之间的延时。默认值是akka.ask,timeout。

当然，该策略也可以在代码里定制：
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per interval
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
));
```

## 2.3 无重启策略

可以在flink-conf.yaml中将flink的重启策略设置为none。

也可以在代码里为每个job单独设置：
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
```

















