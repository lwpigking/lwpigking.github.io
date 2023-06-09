---
layout: post
title: SparkStreaming从入门到放弃~~
categories: [Spark]
description: Spark
keywords: Spark
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## SparkStreaming

### SparkStreaming与流处理

#### 流处理

##### 静态数据处理

在流处理之前，数据通常存储在数据库，文件系统或其他形式的存储系统中。应用程序根据需要查询数据或计算数据。这就是传统的静态数据处理架构。Hadoop 采用 HDFS 进行数据存储，采用 MapReduce 进行数据查询或分析，这就是典型的静态数据处理架构。

![01_data_at_rest_infrastructure](/picture/pictures/01_data_at_rest_infrastructure.png)



##### 流处理

而流处理则是直接对运动中的数据的处理，在接收数据时直接计算数据。

大多数数据都是连续的流：传感器事件，网站上的用户活动，金融交易等等 ，所有这些数据都是随着时间的推移而创建的。

接收和发送数据流并执行应用程序或分析逻辑的系统称为**流处理器**。流处理器的基本职责是确保数据有效流动，同时具备可扩展性和容错能力，Storm 和 Flink 就是其代表性的实现。

![02_stream_processing_infrastructure](/picture/pictures/02_stream_processing_infrastructure.png)

流处理带来了静态数据处理所不具备的众多优点：

 

- **应用程序立即对数据做出反应**：降低了数据的滞后性，使得数据更具有时效性，更能反映对未来的预期；
- **流处理可以处理更大的数据量**：直接处理数据流，并且只保留数据中有意义的子集，并将其传送到下一个处理单元，逐级过滤数据，降低需要处理的数据量，从而能够承受更大的数据量；
- **流处理更贴近现实的数据模型**：在实际的环境中，一切数据都是持续变化的，要想能够通过过去的数据推断未来的趋势，必须保证数据的不断输入和模型的不断修正，典型的就是金融市场、股票市场，流处理能更好的应对这些数据的连续性的特征和及时性的需求；
- **流处理分散和分离基础设施**：流式处理减少了对大型数据库的需求。相反，每个流处理程序通过流处理框架维护了自己的数据和状态，这使得流处理程序更适合微服务架构。



#### Spark Streaming

##### 简介

Spark Streaming 是 Spark 的一个子模块，用于快速构建可扩展，高吞吐量，高容错的流处理程序。具有以下特点：

+ 通过高级 API 构建应用程序，简单易用；
+ 支持多种语言，如 Java，Scala 和 Python；
+ 良好的容错性，Spark Streaming 支持快速从失败中恢复丢失的操作状态；
+ 能够和 Spark 其他模块无缝集成，将流处理与批处理完美结合；
+ Spark Streaming 可以从 HDFS，Flume，Kafka，Twitter 和 ZeroMQ 读取数据，也支持自定义数据源。

![spark-streaming-arch](/picture/pictures/spark-streaming-arch.png)

##### DStream

Spark Streaming 提供称为离散流 (DStream) 的高级抽象，用于表示连续的数据流。 DStream 可以从来自 Kafka，Flume 和 Kinesis 等数据源的输入数据流创建，也可以由其他 DStream 转化而来。**在内部，DStream 表示为一系列 RDD**。

![spark-streaming-flow](/picture/pictures/spark-streaming-flow.png)

##### Spark & Storm & Flink

storm 和 Flink 都是真正意义上的流计算框架，但 Spark Streaming 只是将数据流进行极小粒度的拆分，拆分为多个批处理，使得其能够得到接近于流处理的效果，但其本质上还是批处理（或微批处理）。



### SparkStreaming基本操作

#### 案例引入

这里先引入一个基本的案例来演示流的创建：获取指定端口上的数据并进行词频统计。项目依赖和代码实现如下：

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.12</artifactId>
    <version>2.4.3</version>
</dependency>
```

```scala
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}

object NetworkWordCount {

  def main(args: Array[String]) {

    /*指定时间间隔为 5s*/
    val sparkConf = new SparkConf().setAppName("NetworkWordCount").setMaster("local[2]")
    val ssc = new StreamingContext(sparkConf, Seconds(5))

    /*创建文本输入流,并进行词频统计*/
    val lines = ssc.socketTextStream("hadoop001", 9999)
    lines.flatMap(_.split(" ")).map(x => (x, 1)).reduceByKey(_ + _).print()

    /*启动服务*/
    ssc.start()
    /*等待服务结束*/
    ssc.awaitTermination()
  }
}
```

使用本地模式启动 Spark 程序，然后使用 `nc -lk 9999` 打开端口并输入测试数据：

```shell
[root@hadoop001 ~]#  nc -lk 9999
hello world hello spark hive hive hadoop
storm storm flink azkaban
```

此时控制台输出如下，可以看到已经接收到数据并按行进行了词频统计。

![spark-streaming-word-count-v1](/picture/pictures/spark-streaming-word-count-v1.png)

下面针对示例代码进行讲解：

##### StreamingContext

Spark Streaming 编程的入口类是 StreamingContext，在创建时候需要指明 `sparkConf` 和 `batchDuration`(批次时间)，Spark 流处理本质是将流数据拆分为一个个批次，然后进行微批处理，`batchDuration` 就是批次拆分的时间间隔。这个时间可以根据业务需求和服务器性能进行指定，如果业务要求低延迟并且服务器性能也允许，则这个时间可以指定得很短。

这里需要注意的是：示例代码使用的是本地模式，配置为 `local[2]`，这里不能配置为 `local[1]`。这是因为对于流数据的处理，Spark 必须有一个独立的 Executor 来接收数据，然后再由其他的 Executors 来处理，所以为了保证数据能够被处理，至少要有 2 个 Executors。这里我们的程序只有一个数据流，在并行读取多个数据流的时候，也需要保证有足够的 Executors 来接收和处理数据。

##### 数据源

在示例代码中使用的是 `socketTextStream` 来创建基于 Socket 的数据流，实际上 Spark 还支持多种数据源，分为以下两类：

+ **基本数据源**：包括文件系统、Socket 连接等；
+ **高级数据源**：包括 Kafka，Flume，Kinesis 等。

在基本数据源中，Spark 支持监听 HDFS 上指定目录，当有新文件加入时，会获取其文件内容作为输入流。创建方式如下：

```scala
// 对于文本文件，指明监听目录即可
streamingContext.textFileStream(dataDirectory)
// 对于其他文件，需要指明目录，以及键的类型、值的类型、和输入格式
streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
```

被监听的目录可以是具体目录，如 `hdfs://host:8040/logs/`；也可以使用通配符，如 `hdfs://host:8040/logs/2017/*`。

##### 服务的启动与停止

在示例代码中，使用 `streamingContext.start()` 代表启动服务，此时还要使用 `streamingContext.awaitTermination()` 使服务处于等待和可用的状态，直到发生异常或者手动使用 `streamingContext.stop()` 进行终止。



#### Tansformation

##### DStream与RDDs

DStream 是 Spark Streaming 提供的基本抽象。它表示连续的数据流。在内部，DStream 由一系列连续的 RDD 表示。所以从本质上而言，应用于 DStream 的任何操作都会转换为底层 RDD 上的操作。例如，在示例代码中 flatMap 算子的操作实际上是作用在每个 RDDs 上 (如下图)。因为这个原因，所以 DStream 能够支持 RDD 大部分的*transformation*算子。

![spark-streaming-dstream-ops](/picture/pictures/spark-streaming-dstream-ops.png)

##### updateStateByKey

除了能够支持 RDD 的算子外，DStream 还有部分独有的*transformation*算子，这当中比较常用的是 `updateStateByKey`。文章开头的词频统计程序，只能统计每一次输入文本中单词出现的数量，想要统计所有历史输入中单词出现的数量，可以使用 `updateStateByKey` 算子。代码如下：

```scala
object NetworkWordCountV2 {


  def main(args: Array[String]) {

    /*
     * 本地测试时最好指定 hadoop 用户名,否则会默认使用本地电脑的用户名,
     * 此时在 HDFS 上创建目录时可能会抛出权限不足的异常
     */
    System.setProperty("HADOOP_USER_NAME", "root")
      
    val sparkConf = new SparkConf().setAppName("NetworkWordCountV2").setMaster("local[2]")
    val ssc = new StreamingContext(sparkConf, Seconds(5))
    /*必须要设置检查点*/
    ssc.checkpoint("hdfs://hadoop001:8020/spark-streaming")
    val lines = ssc.socketTextStream("hadoop001", 9999)
    lines.flatMap(_.split(" ")).map(x => (x, 1))
      .updateStateByKey[Int](updateFunction _)   //updateStateByKey 算子
      .print()

    ssc.start()
    ssc.awaitTermination()
  }

  /**
    * 累计求和
    *
    * @param currentValues 当前的数据
    * @param preValues     之前的数据
    * @return 相加后的数据
    */
  def updateFunction(currentValues: Seq[Int], preValues: Option[Int]): Option[Int] = {
    val current = currentValues.sum
    val pre = preValues.getOrElse(0)
    Some(current + pre)
  }
}
```

使用 `updateStateByKey` 算子，你必须使用 `ssc.checkpoint()` 设置检查点，这样当使用 `updateStateByKey` 算子时，它会去检查点中取出上一次保存的信息，并使用自定义的 `updateFunction` 函数将上一次的数据和本次数据进行相加，然后返回。

##### 启动测试

在监听端口输入如下测试数据：

```shell
[root@hadoop001 ~]#  nc -lk 9999
hello world hello spark hive hive hadoop
storm storm flink azkaban
hello world hello spark hive hive hadoop
storm storm flink azkaban
```

此时控制台输出如下，所有输入都被进行了词频累计：

![spark-streaming-word-count-v2](/picture/pictures/spark-streaming-word-count-v2.png)

同时在输出日志中还可以看到检查点操作的相关信息：

```shell
# 保存检查点信息
19/05/27 16:21:05 INFO CheckpointWriter: Saving checkpoint for time 1558945265000 ms 
to file 'hdfs://hadoop001:8020/spark-streaming/checkpoint-1558945265000'

# 删除已经无用的检查点信息
19/05/27 16:21:30 INFO CheckpointWriter: 
Deleting hdfs://hadoop001:8020/spark-streaming/checkpoint-1558945265000
```

#### 输出操作

##### 输出API

Spark Streaming 支持以下输出操作：

| Output Operation                            | Meaning                                                      |
| :------------------------------------------ | :----------------------------------------------------------- |
| **print**()                                 | 在运行流应用程序的 driver 节点上打印 DStream 中每个批次的前十个元素。用于开发调试。 |
| **saveAsTextFiles**(*prefix*, [*suffix*])   | 将 DStream 的内容保存为文本文件。每个批处理间隔的文件名基于前缀和后缀生成：“prefix-TIME_IN_MS [.suffix]”。 |
| **saveAsObjectFiles**(*prefix*, [*suffix*]) | 将 DStream 的内容序列化为 Java 对象，并保存到 SequenceFiles。每个批处理间隔的文件名基于前缀和后缀生成：“prefix-TIME_IN_MS [.suffix]”。 |
| **saveAsHadoopFiles**(*prefix*, [*suffix*]) | 将 DStream 的内容保存为 Hadoop 文件。每个批处理间隔的文件名基于前缀和后缀生成：“prefix-TIME_IN_MS [.suffix]”。 |
| **foreachRDD**(*func*)                      | 最通用的输出方式，它将函数 func 应用于从流生成的每个 RDD。此函数应将每个 RDD 中的数据推送到外部系统，例如将 RDD 保存到文件，或通过网络将其写入数据库。 |

前面的四个 API 都是直接调用即可，下面主要讲解通用的输出方式 `foreachRDD(func)`，通过该 API 你可以将数据保存到任何你需要的数据源。

##### foreachRDD

这里我们使用 Redis 作为客户端，对文章开头示例程序进行改变，把每一次词频统计的结果写入到 Redis，并利用 Redis 的 `HINCRBY` 命令来进行词频统计。这里需要导入 Jedis 依赖：

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
```

具体实现代码如下:

```scala
import org.apache.spark.SparkConf
import org.apache.spark.streaming.dstream.DStream
import org.apache.spark.streaming.{Seconds, StreamingContext}
import redis.clients.jedis.Jedis

object NetworkWordCountToRedis {
  
    def main(args: Array[String]) {

    val sparkConf = new SparkConf().setAppName("NetworkWordCountToRedis").setMaster("local[2]")
    val ssc = new StreamingContext(sparkConf, Seconds(5))

    /*创建文本输入流,并进行词频统计*/
    val lines = ssc.socketTextStream("hadoop001", 9999)
    val pairs: DStream[(String, Int)] = lines.flatMap(_.split(" ")).map(x => (x, 1)).reduceByKey(_ + _)
     /*保存数据到 Redis*/
    pairs.foreachRDD { rdd =>
      rdd.foreachPartition { partitionOfRecords =>
        var jedis: Jedis = null
        try {
          jedis = JedisPoolUtil.getConnection
          partitionOfRecords.foreach(record => jedis.hincrBy("wordCount", record._1, record._2))
        } catch {
          case ex: Exception =>
            ex.printStackTrace()
        } finally {
          if (jedis != null) jedis.close()
        }
      }
    }
    ssc.start()
    ssc.awaitTermination()
  }
}

```

其中 `JedisPoolUtil` 的代码如下：

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

public class JedisPoolUtil {

    /* 声明为 volatile 防止指令重排序 */
    private static volatile JedisPool jedisPool = null;
    private static final String HOST = "localhost";
    private static final int PORT = 6379;

    /* 双重检查锁实现懒汉式单例 */
    public static Jedis getConnection() {
        if (jedisPool == null) {
            synchronized (JedisPoolUtil.class) {
                if (jedisPool == null) {
                    JedisPoolConfig config = new JedisPoolConfig();
                    config.setMaxTotal(30);
                    config.setMaxIdle(10);
                    jedisPool = new JedisPool(config, HOST, PORT);
                }
            }
        }
        return jedisPool.getResource();
    }
}
```

##### 代码说明

这里将上面保存到 Redis 的代码单独抽取出来，并去除异常判断的部分。精简后的代码如下：

```scala
pairs.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    val jedis = JedisPoolUtil.getConnection
    partitionOfRecords.foreach(record => jedis.hincrBy("wordCount", record._1, record._2))
    jedis.close()
  }
}
```

这里可以看到一共使用了三次循环，分别是循环 RDD，循环分区，循环每条记录，上面我们的代码是在循环分区的时候获取连接，也就是为每一个分区获取一个连接。但是这里大家可能会有疑问：为什么不在循环 RDD 的时候，为每一个 RDD 获取一个连接，这样所需要的连接数会更少。实际上这是不可行的，如果按照这种情况进行改写，如下：

```scala
pairs.foreachRDD { rdd =>
    val jedis = JedisPoolUtil.getConnection
    rdd.foreachPartition { partitionOfRecords =>
        partitionOfRecords.foreach(record => jedis.hincrBy("wordCount", record._1, record._2))
    }
    jedis.close()
}
```

此时在执行时候就会抛出 `Caused by: java.io.NotSerializableException: redis.clients.jedis.Jedis`，这是因为在实际计算时，Spark 会将对 RDD 操作分解为多个 Task，Task 运行在具体的 Worker Node 上。在执行之前，Spark 会对任务进行闭包，之后闭包被序列化并发送给每个 Executor，而 `Jedis` 显然是不能被序列化的，所以会抛出异常。

第二个需要注意的是 ConnectionPool 最好是一个静态，惰性初始化连接池 。这是因为 Spark 的转换操作本身就是惰性的，且没有数据流时不会触发写出操作，所以出于性能考虑，连接池应该是惰性的，因此上面 `JedisPool` 在初始化时采用了懒汉式单例进行惰性初始化。

##### 启动测试

在监听端口输入如下测试数据：

```shell
[root@hadoop001 ~]#  nc -lk 9999
hello world hello spark hive hive hadoop
storm storm flink azkaban
hello world hello spark hive hive hadoop
storm storm flink azkaban
```

使用 Redis Manager 查看写入结果 (如下图),可以看到与使用 `updateStateByKey` 算子得到的计算结果相同。

![spark-streaming-word-count-v3](/picture/pictures/spark-streaming-word-count-v3.png)



### 整合Flume

#### 简介

Apache Flume 是一个分布式，高可用的数据收集系统，可以从不同的数据源收集数据，经过聚合后发送到分布式计算框架或者存储系统中。Spark Straming 提供了以下两种方式用于 Flume 的整合。

#### 推送式方法

在推送式方法 (Flume-style Push-based Approach) 中，Spark Streaming 程序需要对某台服务器的某个端口进行监听，Flume 通过 `avro Sink` 将数据源源不断推送到该端口。这里以监听日志文件为例，具体整合方式如下：

##### 配置日志收集Flume

新建配置 `netcat-memory-avro.properties`，使用 `tail` 命令监听文件内容变化，然后将新的文件内容通过 `avro sink` 发送到 hadoop001 这台服务器的 8888 端口：

```properties
#指定agent的sources,sinks,channels
a1.sources = s1
a1.sinks = k1
a1.channels = c1

#配置sources属性
a1.sources.s1.type = exec
a1.sources.s1.command = tail -F /tmp/log.txt
a1.sources.s1.shell = /bin/bash -c
a1.sources.s1.channels = c1

#配置sink
a1.sinks.k1.type = avro
a1.sinks.k1.hostname = hadoop001
a1.sinks.k1.port = 8888
a1.sinks.k1.batch-size = 1
a1.sinks.k1.channel = c1

#配置channel类型
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
```

##### 项目依赖

项目采用 Maven 工程进行构建，主要依赖为 `spark-streaming` 和 `spark-streaming-flume`。

```xml
<properties>
    <scala.version>2.11</scala.version>
    <spark.version>2.4.0</spark.version>
</properties>

<dependencies>
    <!-- Spark Streaming-->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_${scala.version}</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <!-- Spark Streaming 整合 Flume 依赖-->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming-flume_${scala.version}</artifactId>
        <version>2.4.3</version>
    </dependency>
</dependencies>

```

##### Spark Streaming接收日志数据

调用 FlumeUtils 工具类的 `createStream` 方法，对 hadoop001 的 8888 端口进行监听，获取到流数据并进行打印：

```scala
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.streaming.flume.FlumeUtils

object PushBasedWordCount {
    
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf()
    val ssc = new StreamingContext(sparkConf, Seconds(5))
    // 1.获取输入流
    val flumeStream = FlumeUtils.createStream(ssc, "hadoop001", 8888)
    // 2.打印输入流的数据
    flumeStream.map(line => new String(line.event.getBody.array()).trim).print()

    ssc.start()
    ssc.awaitTermination()
  }
}
```

##### 项目打包

因为 Spark 安装目录下是不含有 `spark-streaming-flume` 依赖包的，所以在提交到集群运行时候必须提供该依赖包，你可以在提交命令中使用 `--jar` 指定上传到服务器的该依赖包，或者使用 `--packages org.apache.spark:spark-streaming-flume_2.12:2.4.3` 指定依赖包的完整名称，这样程序在启动时会先去中央仓库进行下载。

这里我采用的是第三种方式：使用 `maven-shade-plugin` 插件进行 `ALL IN ONE` 打包，把所有依赖的 Jar 一并打入最终包中。需要注意的是 `spark-streaming` 包在 Spark 安装目录的 `jars` 目录中已经提供，所以不需要打入。插件配置如下：


```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>8</source>
                <target>8</target>
            </configuration>
        </plugin>
        <!--使用 shade 进行打包-->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <configuration>
                <createDependencyReducedPom>true</createDependencyReducedPom>
                <filters>
                    <filter>
                        <artifact>*:*</artifact>
                        <excludes>
                            <exclude>META-INF/*.SF</exclude>
                            <exclude>META-INF/*.sf</exclude>
                            <exclude>META-INF/*.DSA</exclude>
                            <exclude>META-INF/*.dsa</exclude>
                            <exclude>META-INF/*.RSA</exclude>
                            <exclude>META-INF/*.rsa</exclude>
                            <exclude>META-INF/*.EC</exclude>
                            <exclude>META-INF/*.ec</exclude>
                            <exclude>META-INF/MSFTSIG.SF</exclude>
                            <exclude>META-INF/MSFTSIG.RSA</exclude>
                        </excludes>
                    </filter>
                </filters>
                <artifactSet>
                    <excludes>
                        <exclude>org.apache.spark:spark-streaming_${scala.version}</exclude>
                        <exclude>org.scala-lang:scala-library</exclude>
                        <exclude>org.apache.commons:commons-lang3</exclude>
                    </excludes>
                </artifactSet>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <transformers>
                            <transformer 
                              implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            <transformer 
                              implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
        <!--打包.scala 文件需要配置此插件-->
        <plugin>
            <groupId>org.scala-tools</groupId>
            <artifactId>maven-scala-plugin</artifactId>
            <version>2.15.1</version>
            <executions>
                <execution>
                    <id>scala-compile</id>
                    <goals>
                        <goal>compile</goal>
                    </goals>
                    <configuration>
                        <includes>
                            <include>**/*.scala</include>
                        </includes>
                    </configuration>
                </execution>
                <execution>
                    <id>scala-test-compile</id>
                    <goals>
                        <goal>testCompile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

使用 `mvn clean package` 命令打包后会生产以下两个 Jar 包，提交 ` 非 original` 开头的 Jar 即可。

![spark-streaming-flume-jar](/picture/pictures/spark-streaming-flume-jar.png)

##### 启动服务和提交作业

 启动 Flume 服务：

```shell
flume-ng agent \
--conf conf \
--conf-file /usr/app/apache-flume-1.6.0-cdh5.15.2-bin/examples/netcat-memory-avro.properties \
--name a1 -Dflume.root.logger=INFO,console
```

提交 Spark Streaming 作业：

```shell
spark-submit \
--class com.heibaiying.flume.PushBasedWordCount \
--master local[4] \
/usr/appjar/spark-streaming-flume-1.0.jar
```

##### 测试

这里使用 `echo` 命令模拟日志产生的场景，往日志文件中追加数据，然后查看程序的输出：

![spark-flume-input](/picture/pictures/spark-flume-input.png)

Spark Streaming 程序成功接收到数据并打印输出：

![spark-flume-console](/picture/pictures/spark-flume-console.png)

##### 注意事项

###### 启动顺序

这里需要注意的，不论你先启动 Spark 程序还是 Flume 程序，由于两者的启动都需要一定的时间，此时先启动的程序会短暂地抛出端口拒绝连接的异常，此时不需要进行任何操作，等待两个程序都启动完成即可。

![flume-retry](/picture/pictures/flume-retry.png)

###### 版本一致

最好保证用于本地开发和编译的 Scala 版本和 Spark 的 Scala 版本一致，至少保证大版本一致，如都是 `2.11`。

#### 拉取式方法

拉取式方法 (Pull-based Approach using a Custom Sink) 是将数据推送到 `SparkSink` 接收器中，此时数据会保持缓冲状态，Spark Streaming 定时从接收器中拉取数据。这种方式是基于事务的，即只有在 Spark Streaming 接收和复制数据完成后，才会删除缓存的数据。与第一种方式相比，具有更强的可靠性和容错保证。整合步骤如下：

##### 配置日志收集Flume

新建 Flume 配置文件 `netcat-memory-sparkSink.properties`，配置和上面基本一致，只是把 `a1.sinks.k1.type` 的属性修改为 `org.apache.spark.streaming.flume.sink.SparkSink`，即采用 Spark 接收器。

```properties
#指定agent的sources,sinks,channels
a1.sources = s1
a1.sinks = k1
a1.channels = c1

#配置sources属性
a1.sources.s1.type = exec
a1.sources.s1.command = tail -F /tmp/log.txt
a1.sources.s1.shell = /bin/bash -c
a1.sources.s1.channels = c1

#配置sink
a1.sinks.k1.type = org.apache.spark.streaming.flume.sink.SparkSink
a1.sinks.k1.hostname = hadoop001
a1.sinks.k1.port = 8888
a1.sinks.k1.batch-size = 1
a1.sinks.k1.channel = c1

#配置channel类型
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
```

##### 新增依赖

使用拉取式方法需要额外添加以下两个依赖：

```xml
<dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-library</artifactId>
    <version>2.12.8</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.5</version>
</dependency>
```

注意：添加这两个依赖只是为了本地测试，Spark 的安装目录下已经提供了这两个依赖，所以在最终打包时需要进行排除。

##### Spark Streaming接收日志数据

这里和上面推送式方法的代码基本相同，只是将调用方法改为 `createPollingStream`。

```scala
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.streaming.flume.FlumeUtils

object PullBasedWordCount {

  def main(args: Array[String]): Unit = {

    val sparkConf = new SparkConf()
    val ssc = new StreamingContext(sparkConf, Seconds(5))
    // 1.获取输入流
    val flumeStream = FlumeUtils.createPollingStream(ssc, "hadoop001", 8888)
    // 2.打印输入流中的数据
    flumeStream.map(line => new String(line.event.getBody.array()).trim).print()
    ssc.start()
    ssc.awaitTermination()
  }
}
```

##### 启动测试

启动和提交作业流程与上面相同，这里给出执行脚本，过程不再赘述。

启动 Flume 进行日志收集：

```shell
flume-ng agent \
--conf conf \
--conf-file /usr/app/apache-flume-1.6.0-cdh5.15.2-bin/examples/netcat-memory-sparkSink.properties \
--name a1 -Dflume.root.logger=INFO,console
```

提交 Spark Streaming 作业：

```shel
spark-submit \
--class com.heibaiying.flume.PullBasedWordCount \
--master local[4] \
/usr/appjar/spark-streaming-flume-1.0.jar
```

### 整合Kafka

#### 版本说明

Spark 针对 Kafka 的不同版本，提供了两套整合方案：`spark-streaming-kafka-0-8` 和 `spark-streaming-kafka-0-10`，其主要区别如下：

|                                               | [spark-streaming-kafka-0-8](https://spark.apache.org/docs/latest/streaming-kafka-0-8-integration.html) | [spark-streaming-kafka-0-10](https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html) |
| :-------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| Kafka 版本                                    | 0.8.2.1 or higher                                            | 0.10.0 or higher                                             |
| AP 状态                                       | Deprecated<br/>从 Spark 2.3.0 版本开始，Kafka 0.8 支持已被弃用 | Stable(稳定版)                                               |
| 语言支持                                      | Scala, Java, Python                                          | Scala, Java                                                  |
| Receiver DStream                              | Yes                                                          | No                                                           |
| Direct DStream                                | Yes                                                          | Yes                                                          |
| SSL / TLS Support                             | No                                                           | Yes                                                          |
| Offset Commit API(偏移量提交)                 | No                                                           | Yes                                                          |
| Dynamic Topic Subscription<br/>(动态主题订阅) | No                                                           | Yes                                                          |

本文使用的 Kafka 版本为 `kafka_2.12-2.2.0`，故采用第二种方式进行整合。

#### 项目依赖

项目采用 Maven 进行构建，主要依赖如下：

```xml
<properties>
    <scala.version>2.12</scala.version>
</properties>

<dependencies>
    <!-- Spark Streaming-->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_${scala.version}</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <!-- Spark Streaming 整合 Kafka 依赖-->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming-kafka-0-10_${scala.version}</artifactId>
        <version>2.4.3</version>
    </dependency>
</dependencies>
```



#### 整合Kafka

通过调用 `KafkaUtils` 对象的 `createDirectStream` 方法来创建输入流，完整代码如下：

```scala
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.SparkConf
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.{Seconds, StreamingContext}

/**
  * spark streaming 整合 kafka
  */
object KafkaDirectStream {

  def main(args: Array[String]): Unit = {

    val sparkConf = new SparkConf().setAppName("KafkaDirectStream").setMaster("local[2]")
    val streamingContext = new StreamingContext(sparkConf, Seconds(5))

    val kafkaParams = Map[String, Object](
      /*
       * 指定 broker 的地址清单，清单里不需要包含所有的 broker 地址，生产者会从给定的 broker 里查找其他 broker 的信息。
       * 不过建议至少提供两个 broker 的信息作为容错。
       */
      "bootstrap.servers" -> "hadoop001:9092",
      /*键的序列化器*/
      "key.deserializer" -> classOf[StringDeserializer],
      /*值的序列化器*/
      "value.deserializer" -> classOf[StringDeserializer],
      /*消费者所在分组的 ID*/
      "group.id" -> "spark-streaming-group",
      /*
       * 该属性指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下该作何处理:
       * latest: 在偏移量无效的情况下，消费者将从最新的记录开始读取数据（在消费者启动之后生成的记录）
       * earliest: 在偏移量无效的情况下，消费者将从起始位置读取分区的记录
       */
      "auto.offset.reset" -> "latest",
      /*是否自动提交*/
      "enable.auto.commit" -> (true: java.lang.Boolean)
    )
    
    /*可以同时订阅多个主题*/
    val topics = Array("spark-streaming-topic")
    val stream = KafkaUtils.createDirectStream[String, String](
      streamingContext,
      /*位置策略*/
      PreferConsistent,
      /*订阅主题*/
      Subscribe[String, String](topics, kafkaParams)
    )

    /*打印输入流*/
    stream.map(record => (record.key, record.value)).print()

    streamingContext.start()
    streamingContext.awaitTermination()
  }
}
```

##### ConsumerRecord

这里获得的输入流中每一个 Record 实际上是 `ConsumerRecord<K, V> ` 的实例，其包含了 Record 的所有可用信息，源码如下：

```scala
public class ConsumerRecord<K, V> {
    
    public static final long NO_TIMESTAMP = RecordBatch.NO_TIMESTAMP;
    public static final int NULL_SIZE = -1;
    public static final int NULL_CHECKSUM = -1;
    
    /*主题名称*/
    private final String topic;
    /*分区编号*/
    private final int partition;
    /*偏移量*/
    private final long offset;
    /*时间戳*/
    private final long timestamp;
    /*时间戳代表的含义*/
    private final TimestampType timestampType;
    /*键序列化器*/
    private final int serializedKeySize;
    /*值序列化器*/
    private final int serializedValueSize;
    /*值序列化器*/
    private final Headers headers;
    /*键*/
    private final K key;
    /*值*/
    private final V value;
    .....   
}
```

##### 生产者属性

在示例代码中 `kafkaParams` 封装了 Kafka 消费者的属性，这些属性和 Spark Streaming 无关，是 Kafka 原生 API 中就有定义的。其中服务器地址、键序列化器和值序列化器是必选的，其他配置是可选的。其余可选的配置项如下：

###### fetch.min.byte

消费者从服务器获取记录的最小字节数。如果可用的数据量小于设置值，broker 会等待有足够的可用数据时才会把它返回给消费者。

###### fetch.max.wait.ms

broker 返回给消费者数据的等待时间。

###### max.partition.fetch.bytes

分区返回给消费者的最大字节数。

###### session.timeout.ms

消费者在被认为死亡之前可以与服务器断开连接的时间。

###### auto.offset.reset

该属性指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下该作何处理：

- latest(默认值) ：在偏移量无效的情况下，消费者将从其启动之后生成的最新的记录开始读取数据；
- earliest ：在偏移量无效的情况下，消费者将从起始位置读取分区的记录。

###### enable.auto.commit

是否自动提交偏移量，默认值是 true,为了避免出现重复数据和数据丢失，可以把它设置为 false。

###### client.id

客户端 id，服务器用来识别消息的来源。

###### max.poll.records

单次调用 `poll()` 方法能够返回的记录数量。

###### receive.buffer.bytes 和 send.buffer.byte

这两个参数分别指定 TCP socket 接收和发送数据包缓冲区的大小，-1 代表使用操作系统的默认值。



##### 位置策略

Spark Streaming 中提供了如下三种位置策略，用于指定 Kafka 主题分区与 Spark 执行程序 Executors 之间的分配关系：

+ **PreferConsistent** : 它将在所有的 Executors 上均匀分配分区；

+ **PreferBrokers** : 当 Spark 的 Executor 与 Kafka Broker 在同一机器上时可以选择该选项，它优先将该 Broker 上的首领分区分配给该机器上的 Executor；
+ **PreferFixed** : 可以指定主题分区与特定主机的映射关系，显示地将分区分配到特定的主机，其构造器如下：

```scala
@Experimental
def PreferFixed(hostMap: collection.Map[TopicPartition, String]): LocationStrategy =
  new PreferFixed(new ju.HashMap[TopicPartition, String](hostMap.asJava))

@Experimental
def PreferFixed(hostMap: ju.Map[TopicPartition, String]): LocationStrategy =
  new PreferFixed(hostMap)
```



##### 订阅方式

Spark Streaming 提供了两种主题订阅方式，分别为 `Subscribe` 和 `SubscribePattern`。后者可以使用正则匹配订阅主题的名称。其构造器分别如下：

```scala
/**
  * @param 需要订阅的主题的集合
  * @param Kafka 消费者参数
  * @param offsets(可选): 在初始启动时开始的偏移量。如果没有，则将使用保存的偏移量或 auto.offset.reset 属性的值
  */
def Subscribe[K, V](
    topics: ju.Collection[jl.String],
    kafkaParams: ju.Map[String, Object],
    offsets: ju.Map[TopicPartition, jl.Long]): ConsumerStrategy[K, V] = { ... }

/**
  * @param 需要订阅的正则
  * @param Kafka 消费者参数
  * @param offsets(可选): 在初始启动时开始的偏移量。如果没有，则将使用保存的偏移量或 auto.offset.reset 属性的值
  */
def SubscribePattern[K, V](
    pattern: ju.regex.Pattern,
    kafkaParams: collection.Map[String, Object],
    offsets: collection.Map[TopicPartition, Long]): ConsumerStrategy[K, V] = { ... }
```

在示例代码中，我们实际上并没有指定第三个参数 `offsets`，所以程序默认采用的是配置的 `auto.offset.reset` 属性的值 latest，即在偏移量无效的情况下，消费者将从其启动之后生成的最新的记录开始读取数据。

##### 提交偏移量

在示例代码中，我们将 `enable.auto.commit` 设置为 true，代表自动提交。在某些情况下，你可能需要更高的可靠性，如在业务完全处理完成后再提交偏移量，这时候可以使用手动提交。想要进行手动提交，需要调用 Kafka 原生的 API :

+ `commitSync`:  用于异步提交；
+ `commitAsync`：用于同步提交。



#### 启动测试

##### 创建主题

###### 启动Kakfa

Kafka 的运行依赖于 zookeeper，需要预先启动，可以启动 Kafka 内置的 zookeeper，也可以启动自己安装的：

```shell
# zookeeper启动命令
bin/zkServer.sh start

# 内置zookeeper启动命令
bin/zookeeper-server-start.sh config/zookeeper.properties
```

启动单节点 kafka 用于测试：

```shell
# bin/kafka-server-start.sh config/server.properties
```

###### 创建topic

```shell
# 创建用于测试主题
bin/kafka-topics.sh --create \
                    --bootstrap-server hadoop001:9092 \
                    --replication-factor 1 \
                    --partitions 1  \
                    --topic spark-streaming-topic

# 查看所有主题
 bin/kafka-topics.sh --list --bootstrap-server hadoop001:9092
```

###### 创建生产者

这里创建一个 Kafka 生产者，用于发送测试数据：

```shell
bin/kafka-console-producer.sh --broker-list hadoop001:9092 --topic spark-streaming-topic
```

###### 本地模式测试

这里我直接使用本地模式启动 Spark Streaming 程序。启动后使用生产者发送数据，从控制台查看结果。

从控制台输出中可以看到数据流已经被成功接收，由于采用 `kafka-console-producer.sh` 发送的数据默认是没有 key 的，所以 key 值为 null。同时从输出中也可以看到在程序中指定的 `groupId` 和程序自动分配的 `clientId`。

![spark-straming-kafka-console](/picture/pictures/spark-straming-kafka-console.png)
