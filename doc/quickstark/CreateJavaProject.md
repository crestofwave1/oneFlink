
## java project

本小结主要是教你如何构建一个java project。

## requirements

构建工程前要求：Maven 3.0.4 (or higher) 并且 Java 8.x

## 创建工程

```
$ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-java      \
      -DarchetypeVersion=1.6.0
```

也可以执行。

```
curl https://flink.apache.org/q/quickstart.sh | bash -s 1.6.0
```
## 查看你的工程

执行完上面的命令之后，会在你的工作目录下生成一个新的目录。

```
$ tree quickstart/
quickstart/
├── pom.xml
└── src
    └── main
        ├── java
        │   └── org
        │       └── myorg
        │           └── quickstart
        │               ├── BatchJob.java
        │               └── StreamingJob.java
        └── resources
            └── log4j.properties

```

该样例工程是官方给出的，内部包含一个BatchJob和一个StreamingJob。

## 编译打包

```scala
mvn clean package
```

打包提交的时候需要在pom.xml里修改你的mainClass。