

#############################
Spark Streaming 编程指南
#############################


*****************************
概述
*****************************


Spark Streaming 是对核心 Spark API 的一个扩展，它能够实现对实时数据流的流式处理，并具有很好的可扩展性、高吞吐量和容错性。Spark Streaming 支持从多种数据源提取数据，如：Kafka、Flume、Twitter、ZeroMQ、Kinesis 以及 TCP 套接字，并且可以提供一些高级 API 来表达复杂的处理算法，如：map、reduce、join 和 window 等。最后，Spark Streaming 支持将处理完的数据推送到文件系统、数据库或者实时仪表盘中展示。实际上，你完全可以将 Spark 的机器学习（machine learning） 和 图计算（graph processing）的算法应用于 Spark Streaming 的数据流当中。

.. image:: imgs/streaming-arch.png
  :scale: 90 %
  :align: center

下图展示了 Spark Streaming 的内部工作原理。Spark Streaming 接收实时输入数据流并将数据划分为一个个小的批次供 Spark Engine 处理，最终生成多个批次的结果流。

.. image:: imgs/streaming-flow.png
  :scale: 90 %
  :align: center

Spark Streaming 为这种持续的数据流提供了一个高级抽象，即：discretized stream(离散数据流) 或 DStream。DStream 既可以从输入数据源创建而来，如：Kafka、Flume或者Kinesis，也可以从其他 DStream 上应用一些高级操作得到。在 Spark 内部，一个 DStream 代表一个 RDD 序列。

本文档将向你展示如何用 DStream 进行 Spark Streaming 编程。Spark Streaming 支持 Scala、Java 和 Python（始于Spark 1.2），本文档的示例包括这三种语言。

注意：对 Python 来说，有一部分 API 尚不支持，或者是和 Scala、Java 不同。本文档中会用高亮形式来注明这部分 Python API。


*****************************
一个小例子
*****************************

在深入 Spark Streaming 编程细节之前，我们先来看看一个简单的小例子以便有个感性认识。假设我们在一个 TCP 端口上监听一个数据服务器的数据，并对收到的文本数据中的单词计数。以下你所需的全部工作：


**Scala**

首先，我们需要导入Spark Streaming 的相关class的一些包，以及一些支持 StreamingContext 隐式转换的包（这些隐式转换能给DStream之类的class增加一些有用的方法）。StreamingContext 是 Spark Streaming 的入口。我们将会创建一个本地 StreamingContext 对象，包含两个执行线程，并将批次间隔设为1秒。

.. code-block:: Scala

  import org.apache.spark._
  import org.apache.spark.streaming._
  import org.apache.spark.streaming.StreamingContext._ // 从Spark 1.3之后这行就可以不需要了

  // 创建一个local StreamingContext，包含2个工作线程，并将批次间隔设为1秒
  // master至少需要2个CPU核，以避免出现任务饿死的情况
  val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
  val ssc = new StreamingContext(conf, Seconds(1))

利用这个上下文对象（StreamingContext），我们可以创建一个DStream，该DStream代表从前面的TCP数据源流入的数据流，同时TCP数据源是由主机名（如：hostnam）和端口（如：9999）来描述的。

.. code-block:: Scala

  // 创建一个连接到hostname:port的DStream，如：localhost:9999
  val lines = ssc.socketTextStream("localhost", 9999)

这里的 lines 就是从数据server接收到的数据流。其中每一条记录都是一行文本。接下来，我们就需要把这些文本行按空格分割成单词。

.. code-block:: Scala

  // 将每一行分割成多个单词
  val words = lines.flatMap(_.split(" "))

flatMap 是一种 “一到多”（one-to-many）的映射算子，它可以将源DStream中每一条记录映射成多条记录，从而产生一个新的DStream对象。在本例中，lines中的每一行都会被flatMap映射为多个单词，从而生成新的words DStream对象。然后，我们就能对这些单词进行计数了。

.. code-block:: Scala

  import org.apache.spark.streaming.StreamingContext._ // Spark 1.3之后不再需要这行
  // 对每一批次中的单词进行计数
  val pairs = words.map(word => (word, 1))
  val wordCounts = pairs.reduceByKey(_ + _)

  // 将该DStream产生的RDD的头十个元素打印到控制台上
  wordCounts.print()

words这个DStream对象经过map算子（一到一的映射）转换为一个包含（word, 1）键值对的DStream对象pairs，再对pairs使用reduce算子，得到每个批次中各个单词的出现频率。最后，wordCounts.print() 将会每秒（前面设定的批次间隔）打印一些单词计数到控制台上。

注意，执行以上代码后，Spark Streaming 只是将计算逻辑设置好，此时并未真正的开始处理数据。要启动之前的处理逻辑，我们还需要如下调用：

.. code-block:: Scala

  ssc.start()            // 启动流式计算
  ssc.awaitTermination()  // 等待直到计算终止

完整的代码可以在 Spark Streaming 的例子 NetworkWordCount 中找到。

如果你已经有一个 Spark 包（下载在这里downloaded，自定义构建在这里built），就可以执行按如下步骤运行这个例子。

首先，你需要运行 netcat（Unix-like系统都会有这个小工具），将其作为data server

.. code-block:: Shell

  $ nc -lk 9999

然后，在另一个终端，按如下指令执行这个例子

.. code-block:: Shell

  $ ./bin/run-example streaming.NetworkWordCount localhost 9999

好了，现在你尝试可以在运行 netcat 的终端里敲几个单词，你会发现这些单词以及相应的计数会出现在启动 Spark Streaming 例子的终端屏幕上。看上去应该和下面这个示意图类似：

# TERMINAL 1:
# Running Netcat

$ nc -lk 9999

hello world


...

# TERMINAL 2: RUNNING NetworkWordCount$ ./bin/run-example streaming.NetworkWordCount localhost 9999
...
-------------------------------------------
Time: 1357008430000 ms
-------------------------------------------
(hello,1)
(world,1)
...


*****************************
基本概念
*****************************

下面，我们在之前的小栗子基础上，继续深入了解一下 Spark Streaming 的一些基本概念。

链接依赖项
=============================

和 Spark 类似，Spark Streaming 也能在 Maven 库中找到。如果你需要编写 Spark Streaming 程序，你就需要将以下依赖加入到你的 SBT 或 Maven 工程依赖中。

**Maven**

.. code-block:: XML

  <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-streaming_2.10</artifactId>
      <version>1.6.1</version>
  </dependency>

**SBT**

.. code-block:: TEXT

  libraryDependencies += "org.apache.spark" % "spark-streaming_2.11" % "2.2.1"

还有，对于从 Kafka、Flume 以及 Kinesis 这类数据源提取数据的流式应用来说，还需要额外增加相应的依赖项，下表列出了各种数据源对应的额外依赖项：

=============     ==========
数据源             Maven构件
=============     ==========
Kafka             spark-streaming-kafka_2.11
Flume             spark-streaming-flume_2.11
Kinesis           spark-streaming-kinesis-asl_2.11 [Amazon Software License]
=============     ==========

最新的依赖项信息（包括源代码和 Maven 构件）请参考 Maven repository。


初始化 StreamingContext
=============================

要初始化任何一个 Spark Streaming 程序，都需要在入口代码中创建一个 StreamingContext 对象。

**Scala**

而 StreamingContext 对象需要一个 SparkConf 对象作为其构造参数。

.. code-block:: Scala

  import org.apache.spark._
  import org.apache.spark.streaming._

  val conf = new SparkConf().setAppName(appName).setMaster(master)
  val ssc = new StreamingContext(conf, Seconds(1))

上面代码中的 appName 是你给该应用起的名字，这个名字会展示在 Spark 集群的 web UI上。而 master 是 Spark, Mesos or YARN cluster URL，如果支持本地测试，你也可以用”local[*]”为其赋值。通常在实际工作中，你不应该将master参数硬编码到代码里，而是应用通过spark-submit的参数来传递master的值（launch the application with spark-submit ）。不过对本地测试来说，”local[*]”足够了（该值传给master后，Spark Streaming将在本地进程中，启动n个线程运行，n与本地系统CPU core数相同）。注意，StreamingContext在内部会创建一个  SparkContext 对象（SparkContext是所有Spark应用的入口，在StreamingContext对象中可以这样访问：ssc.sparkContext）。

StreamingContext 还有另一个构造参数，即：批次间隔，这个值的大小需要根据应用的具体需求和可用的集群资源来确定。详见Spark性能调优（ Performance Tuning）。

StreamingContext 对象也可以通过已有的 SparkContext 对象来创建，示例如下：

.. code-block:: Scala

  import org.apache.spark.streaming._

  val sc = ...                // 已有的SparkContext
  val ssc = new StreamingContext(sc, Seconds(1))


**Java**

A JavaStreamingContext object can be created from a SparkConf object.

.. code-block:: Java

  import org.apache.spark.*;
  import org.apache.spark.streaming.api.java.*;

  SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
  JavaStreamingContext ssc = new JavaStreamingContext(conf, new Duration(1000));

The appName parameter is a name for your application to show on the cluster UI. master is a Spark, Mesos or YARN cluster URL, or a special “local[*]” string to run in local mode. In practice, when running on a cluster, you will not want to hardcode master in the program, but rather launch the application with spark-submit and receive it there. However, for local testing and unit tests, you can pass “local[*]” to run Spark Streaming in-process. Note that this internally creates a JavaSparkContext (starting point of all Spark functionality) which can be accessed as ssc.sparkContext.

The batch interval must be set based on the latency requirements of your application and available cluster resources. See the Performance Tuning section for more details.

A JavaStreamingContext object can also be created from an existing JavaSparkContext.

.. code-block:: Java

  import org.apache.spark.streaming.api.java.*;

  JavaSparkContext sc = ...   //existing JavaSparkContext
  JavaStreamingContext ssc = new JavaStreamingContext(sc, Durations.seconds(1));

**Python**

A StreamingContext object can be created from a SparkContext object.

.. code-block:: Python

  from pyspark import SparkContext
  from pyspark.streaming import StreamingContext

  sc = SparkContext(master, appName)
  ssc = StreamingContext(sc, 1)

The appName parameter is a name for your application to show on the cluster UI. master is a Spark, Mesos or YARN cluster URL, or a special “local[*]” string to run in local mode. In practice, when running on a cluster, you will not want to hardcode master in the program, but rather launch the application with spark-submit and receive it there. However, for local testing and unit tests, you can pass “local[*]” to run Spark Streaming in-process (detects the number of cores in the local system).

The batch interval must be set based on the latency requirements of your application and available cluster resources. See the Performance Tuning section for more details.


StreamingContext 对象创建后，你还需要如下步骤：

1. 创建 DStream 对象，并定义好输入数据源。
2. 基于数据源 DStream 定义好计算逻辑和输出。
3. 调用 streamingContext.start() 启动接收并处理数据。
4. 调用 streamingContext.awaitTermination() 等待流式处理结束（不管是手动结束，还是发生异常错误）
5. 你可以主动调用 streamingContext.stop() 来手动停止处理流程。

需要关注的重点:

* 一旦 streamingContext 启动，就不能再对其计算逻辑进行添加或修改。
* 一旦 streamingContext 被 stop 掉，就不能 restart。
* 单个 JVM 虚机 同一时间只能包含一个 active 的 StreamingContext。
* StreamingContext.stop() 也会把关联的 SparkContext 对象 stop 掉，如果不想把 SparkContext 对象也 stop 掉，可以将StreamingContext.stop 的可选参数 stopSparkContext 设为false。
* 一个 SparkContext 对象可以和多个 StreamingContext 对象关联，只要先对前一个StreamingContext.stop(sparkContext=false)，然后再创建新的StreamingContext对象即可。


离散数据流(DStreams)
=============================

离散数据流（DStream）是 Spark Streaming 最基本的抽象。它代表了一种连续的数据流，要么从某种数据源提取数据，要么从其他数据流映射转换而来。DStream 内部是由一系列连续的RDD组成的，每个RDD都是不可变、分布式的数据集（详见Spark编程指南 – Spark Programming Guide）。每个 RDD 都包含了特定时间间隔内的一批数据，如下图所示：

.. image:: imgs/streaming-dstream.png
  :scale: 90 %
  :align: center

任何作用于 DStream 的算子，其实都会被转化为对其内部 RDD 的操作。例如，在前面的例子中，我们将 lines 这个 DStream 转成 words DStream 对象，其实作用于 lines 上的 flatMap 算子，会施加于 lines 中的每个 RDD 上，并生成新的对应的 RDD，而这些新生成的 RDD 对象就组成了 words 这个 DStream 对象。其过程如下图所示：

.. image:: imgs/streaming-dstream-ops.png
  :scale: 90 %
  :align: center

底层的 RDD 转换仍然是由 Spark 引擎来计算。DStream 的算子将这些细节隐藏了起来，并为开发者提供了更为方便的高级API。后续会详细讨论这些高级算子。


输入DStream和接收器
=============================

输入 DStream 代表从某种流式数据源流入的数据流。在之前的例子里，lines 对象就是输入 DStream，它代表从 netcat server收到的数据流。每个输入DStream（除文件数据流外）都和一个接收器（Receiver – Scala doc, Java doc）相关联，而接收器则是专门从数据源拉取数据到内存中的对象。

Spark Streaming 主要提供两种内建的流式数据源：

* 基础数据源（Basic sources）: 在 StreamingContext API 中可直接使用的源，如：文件系统，套接字连接或者Akka actor。
* 高级数据源（Advanced sources）: 需要依赖额外工具类的源，如：Kafka、Flume、Kinesis、Twitter等数据源。这些数据源都需要增加额外的依赖，详见依赖链接（linking）这一节。

本节中，我们将会从每种数据源中挑几个继续深入讨论。

.. attention:: 如果你需要同时从多个数据源拉取数据，那么你就需要创建多个 DStream 对象（详见后续的性能调优这一小节）。多个 DStream 对象其实也就同时创建了多个数据流接收器。但是请注意，Spark的 worker/executor 都是长期运行的，因此它们都会各自占用一个分配给 Spark Streaming 应用的 CPU。所以，在运行nSpark Streaming 应用的时候，需要注意分配足够的CPU core（本地运行时，需要足够的线程）来处理接收到的数据，同时还要足够的CPU core来运行这些接收器。

**要点**

* 如果本地运行 Spark Streaming 应用，记得不能将 master 设为 ”local” 或 “local[1]”。这两个值都只会在本地启动一个线程。而如果此时你使用一个包含接收器（如：套接字、Kafka、Flume等）的输入DStream，那么这一个线程只能用于运行这个接收器，而处理数据的逻辑就没有线程来执行了。因此，本地运行时，一定要将 master 设为 ”local[n]”，其中 n > 接收器的个数（有关master的详情请参考Spark Properties）。

* 将 Spark Streaming 应用置于集群中运行时，同样，分配给该应用的 CPU core 数必须大于接收器的总数。否则，该应用就只会接收数据，而不会处理数据。

基础数据源
---------------------

前面的小栗子中，我们已经看到，使用ssc.socketTextStream(…) 可以从一个TCP连接中接收文本数据。而除了TCP套接字外，StreamingContext API 还支持从文件或者Akka actor中拉取数据。
* 文件数据流（File Streams）: 可以从任何兼容HDFS API（包括：HDFS、S3、NFS等）的文件系统，创建方式如下：
    **Scala**

  .. code-block:: Scala

    streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)

    **Java**

  .. code-block:: Java

    streamingContext.fileStream<KeyClass, ValueClass, InputFormatClass>(dataDirectory);

    **Python**

  .. code-block:: Python

    streamingContext.textFileStream(dataDirectory)

* Spark Streaming将监视该dataDirectory目录，并处理该目录下任何新建的文件（目前还不支持嵌套目录）。注意：  streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
    * 各个文件数据格式必须一致。
    * dataDirectory中的文件必须通过moving或者renaming来创建。
    * 一旦文件move进dataDirectory之后，就不能再改动。所以如果这个文件后续还有写入，这些新写入的数据不会被读取。
* Python API fileStream目前暂时不可用，Python目前只支持textFileStream。另外，文件数据流不是基于接收器的，所以不需要为其单独分配一个CPU core。对于简单的文本文件，更简单的方式是调用 streamingContext.textFileStream(dataDirectory)。
* 基于自定义Actor的数据流（Streams based on Custom Actors）: DStream可以由Akka actor创建得到，只需调用 streamingContext.actorStream(actorProps, actor-name)。详见自定义接收器（Custom Receiver Guide）。actorStream暂时不支持Python API。
* RDD队列数据流（Queue of RDDs as a Stream）: 如果需要测试Spark Streaming应用，你可以创建一个基于一批RDD的DStream对象，只需调用 streamingContext.queueStream(queueOfRDDs)。RDD会被一个个依次推入队列，而DStream则会依次以数据流形式处理这些RDD的数据。
关于套接字、文件以及Akka actor数据流更详细信息，请参考相关文档：StreamingContext for Scala,JavaStreamingContext for Java, and StreamingContext for Python。

高级数据源
---------------------

Python API 自 Spark 1.6.1 起，Kafka、Kinesis、Flume 和 MQTT 这些数据源将支持 Python。

使用这类数据源需要依赖一些额外的代码库，有些依赖还挺复杂的（如：Kafka、Flume）。因此为了减少依赖项版本冲突问题，各个数据源DStream的相关功能被分割到不同的代码包中，只有用到的时候才需要链接打包进来。例如，如果你需要使用Twitter的tweets作为数据源，你需要以下步骤：
1. Linking: 将spark-streaming-twitter_2.10工件加入到SBT/Maven项目依赖中。
2. Programming: 导入TwitterUtils class，然后调用 TwitterUtils.createStream 创建一个DStream，具体代码见下放。
3. Deploying: 生成一个uber Jar包，并包含其所有依赖项（包括 spark-streaming-twitter_2.10及其自身的依赖树），再部署这个Jar包。部署详情请参考部署这一节（Deploying section）。
* Scala
* Java
import org.apache.spark.streaming.twitter._

TwitterUtils.createStream(ssc, None)

注意，高级数据源在spark-shell中不可用，因此不能用spark-shell来测试基于高级数据源的应用。如果真有需要的话，你需要自行下载相应数据源的Maven工件及其依赖项，并将这些Jar包部署到spark-shell的classpath中。

下面列举了一些高级数据源：

* Kafka: Spark Streaming 2.2.1 可兼容 Kafka 0.8.2.1。详见 Kafka Integration Guide。
* Flume: Spark Streaming 2.2.1 可兼容 Flume 1.6.0 。详见Flume Integration Guide。
* Kinesis: Spark Streaming 2.2.1 可兼容 Kinesis Client Library 1.2.1。详见Kinesis Integration Guide。

自定义数据源
------------------------

Python API 自定义数据源目前还不支持Python。

输入DStream也可以用自定义的方式创建。你需要做的只是实现一个自定义的接收器（receiver），以便从自定义的数据源接收数据，然后将数据推入Spark中。详情请参考自定义接收器指南（Custom Receiver Guide）。

接收器可靠性
------------------------

从可靠性角度来划分，大致有两种数据源。其中，像Kafka、Flume这样的数据源，它们支持对所传输的数据进行确认。系统收到这类可靠数据源过来的数据，然后发出确认信息，这样就能够确保任何失败情况下，都不会丢数据。因此我们可以将接收器也相应地分为两类：

1. 可靠接收器（Reliable Receiver） – 可靠接收器会在成功接收并保存好Spark数据副本后，向可靠数据源发送确认信息。
2. 不可靠接收器（Unreliable Receiver） – 不可靠接收器不会发送任何确认信息。不过这种接收器常用语于不支持确认的数据源，或者不想引入数据确认的复杂性的数据源。

自定义接收器指南（Custom Receiver Guide）中详细讨论了如何写一个可靠接收器。


DStream支持的transformation算子
==================================

和 RDD 类似，DStream 也支持从输入 DStream 经过各种 transformation 算子映射成新的 DStream。DStream 支持很多 RDD 上常见的 transformation 算子，一些常用的见下表：

==================================        =====================
Transformation算子                         用途
==================================        =====================
map(func)                                 返回会一个新的DStream，并将源DStream中每个元素通过func映射为新的元素
flatMap(func)                             和map类似，不过每个输入元素不再是映射为一个输出，而是映射为0到多个输出
filter(func)                              返回一个新的DStream，并包含源DStream中被func选中（func返回true）的元素
repartition(numPartitions)                更改DStream的并行度（增加或减少分区数）
union(otherStream)                        返回新的DStream，包含源DStream和otherDStream元素的并集
count()                                   返回一个包含单元素RDDs的DStream，其中每个元素是源DStream中各个RDD中的元素个数
reduce(func)                              返回一个包含单元素RDDs的DStream，其中每个元素是通过源RDD中各个RDD的元素经func（func输入两个参数并返回一个同类型结果数据）聚合得到的结果。func必须满足结合律，以便支持并行计算。
countByValue()                            如果源DStream包含的元素类型为K，那么该算子返回新的DStream包含元素为(K, Long)键值对，其中K为源DStream各个元素，而Long为该元素出现的次数。
reduceByKey(func, [numTasks])             如果源DStream 包含的元素为 (K, V) 键值对，则该算子返回一个新的也包含(K, V)键值对的DStream，其中V是由func聚合得到的。注意：默认情况下，该算子使用Spark的默认并发任务数（本地模式为2，集群模式下由spark.default.parallelism 决定）。你可以通过可选参数numTasks来指定并发任务个数。
join(otherStream, [numTasks])             如果源DStream包含元素为(K, V)，同时otherDStream包含元素为(K, W)键值对，则该算子返回一个新的DStream，其中源DStream和otherDStream中每个K都对应一个 (K, (V, W))键值对元素。
cogroup(otherStream, [numTasks])          如果源DStream包含元素为(K, V)，同时otherDStream包含元素为(K, W)键值对，则该算子返回一个新的DStream，其中每个元素类型为包含(K, Seq[V], Seq[W])的tuple。
transform(func)                           返回一个新的DStream，其包含的RDD为源RDD经过func操作后得到的结果。利用该算子可以对DStream施加任意的操作。
updateStateByKey(func)                    返回一个包含新”状态”的DStream。源DStream中每个key及其对应的values会作为func的输入，而func可以用于对每个key的“状态”数据作任意的更新操作。
==================================        =====================


下面我们会挑几个transformation算子深入讨论一下。

updateStateByKey算子
--------------------------------

updateStateByKey 算子支持维护一个任意的状态。要实现这一点，只需要两步：

1. 定义状态 – 状态数据可以是任意类型。
2. 定义状态更新函数 – 定义好一个函数，其输入为数据流之前的状态和新的数据流数据，且可其更新步骤1中定义的输入数据流的状态。

在每一个批次数据到达后，Spark都会调用状态更新函数，来更新所有已有key（不管key是否存在于本批次中）的状态。如果状态更新函数返回None，则对应的键值对会被删除。

举例如下。假设你需要维护一个流式应用，统计数据流中每个单词的出现次数。这里将各个单词的出现次数这个整型数定义为状态。我们接下来定义状态更新函数如下：

**Scala**

.. code-block:: Scala

  def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
      val newCount = ...  // 将新的计数值和之前的状态值相加，得到新的计数值
      Some(newCount)
  }

该状态更新函数可以作用于一个包括(word, 1) 键值对的DStream上（见本文开头的小栗子）。

.. code-block:: Scala

  val runningCounts = pairs.updateStateByKey[Int](updateFunction _)

该状态更新函数会为每个单词调用一次，且相应的newValues是一个包含很多个”1″的数组（这些1来自于(word,1)键值对），而runningCount包含之前该单词的计数。本例的完整代码请参考 StatefulNetworkWordCount.scala。

**Java**

.. code-block:: Java

  Function2<List<Integer>, Optional<Integer>, Optional<Integer>> updateFunction =
    (values, state) -> {
      Integer newSum = ...  // add the new values with the previous running count to get the new count
      return Optional.of(newSum);
    };

This is applied on a DStream containing words (say, the pairs DStream containing (word, 1) pairs in the quick example).

.. code-block:: Java

  JavaPairDStream<String, Integer> runningCounts = pairs.updateStateByKey(updateFunction);

The update function will be called for each word, with newValues having a sequence of 1’s (from the (word, 1) pairs) and the runningCount having the previous count. For the complete Java code, take a look at the example JavaStatefulNetworkWordCount.java.

**Python**

.. code-block:: Python

  def updateFunction(newValues, runningCount):
      if runningCount is None:
          runningCount = 0
      return sum(newValues, runningCount)  # add the new values with the previous running count to get the new count

This is applied on a DStream containing words (say, the pairs DStream containing (word, 1) pairs in the earlier example).

.. code-block:: Python

  runningCounts = pairs.updateStateByKey(updateFunction)

The update function will be called for each word, with newValues having a sequence of 1’s (from the (word, 1) pairs) and the runningCount having the previous count. For the complete Python code, take a look at the example stateful_network_wordcount.py.


注意，调用 updateStateByKey 前需要配置检查点目录，后续对此有详细的讨论，见检查点（checkpointing）这节。

transform 算子
------------------------

transform算子（及其变体transformWith）可以支持任意的RDD到RDD的映射操作。也就是说，你可以用tranform算子来包装任何DStream API所不支持的RDD算子。例如，将DStream每个批次中的RDD和另一个Dataset进行关联（join）操作，这个功能DStream API并没有直接支持。不过你可以用transform来实现这个功能，可见transform其实为DStream提供了非常强大的功能支持。比如说，你可以用事先算好的垃圾信息，对DStream进行实时过滤。

**Scala**

.. code-block:: Scala

  val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // 包含垃圾信息的RDD

  val cleanedDStream = wordCounts.transform(rdd => {
    rdd.join(spamInfoRDD).filter(...) // 将DStream中的RDD和spamInfoRDD关联，并实时过滤垃圾数据
    ...
  })

**Java**

.. code-block:: Java

  import org.apache.spark.streaming.api.java.*;
  // RDD containing spam information
  JavaPairRDD<String, Double> spamInfoRDD = jssc.sparkContext().newAPIHadoopRDD(...);

  JavaPairDStream<String, Integer> cleanedDStream = wordCounts.transform(rdd -> {
    rdd.join(spamInfoRDD).filter(...); // join data stream with spam information to do data cleaning
    ...
  });

**Python**

.. code-block:: Python

  spamInfoRDD = sc.pickleFile(...)  # RDD containing spam information

  # join data stream with spam information to do data cleaning
  cleanedDStream = wordCounts.transform(lambda rdd: rdd.join(spamInfoRDD).filter(...))

.. attention:: 这里transform包含的算子，其调用时间间隔和批次间隔是相同的。所以你可以基于时间改变对RDD的操作，如：在不同批次，调用不同的RDD算子，设置不同的RDD分区或者广播变量等。

基于窗口（window）的算子
Spark Streaming 同样也提供基于时间窗口的计算，也就是说，你可以对某一个滑动时间窗内的数据施加特定tranformation算子。如下图所示：

.. image:: imgs/streaming-dstream-window.png
  :scale: 90 %
  :align: center

如上图所示，每次窗口滑动时，源 DStream 中落入窗口的 RDDs 就会被合并成新的 windowed DStream。在上图的例子中，这个操作会施加于3个RDD单元，而滑动距离是2个RDD单元。由此可以得出任何窗口相关操作都需要指定一下两个参数:

* （窗口长度）window length – 窗口覆盖的时间长度（上图中为3）
* （滑动距离）sliding interval – 窗口启动的时间间隔（上图中为2）

注意，这两个参数都必须是 DStream 批次间隔（上图中为1）的整数倍.

下面咱们举个栗子。假设，你需要扩展前面的那个小栗子，你需要每隔10秒统计一下前30秒内的单词计数。为此，我们需要在包含(word, 1)键值对的DStream上，对最近30秒的数据调用reduceByKey算子。不过这些都可以简单地用一个 reduceByKeyAndWindow搞定。

**Scala**

.. code-block:: Scala

  // 每隔10秒归约一次最近30秒的数据
  val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))


**Java**

.. code-block:: Java

  // Reduce last 30 seconds of data, every 10 seconds
  JavaPairDStream<String, Integer> windowedWordCounts = pairs.reduceByKeyAndWindow((i1, i2) -> i1 + i2, Durations.seconds(30), Durations.seconds(10));

**Python**

.. code-block:: Python

  # Reduce last 30 seconds of data, every 10 seconds
  windowedWordCounts = pairs.reduceByKeyAndWindow(lambda x, y: x + y, lambda x, y: x - y, 30, 10)

以下列出了常用的窗口算子。所有这些算子都有前面提到的那两个参数 – 窗口长度 和 滑动距离。

============================================================================      =============
Transformation窗口算子                                                              用途
============================================================================      =============
window(windowLength, slideInterval)                                               将源DStream窗口化，并返回转化后的DStream
countByWindow(windowLength,slideInterval)                                         返回数据流在一个滑动窗口内的元素个数
reduceByWindow(func, windowLength,slideInterval)                                  基于数据流在一个滑动窗口内的元素，用func做聚合，返回一个单元素数据流。func必须满足结合律，以便支持并行计算。
reduceByKeyAndWindow(func,windowLength, slideInterval, [numTasks])                基于(K, V)键值对DStream，将一个滑动窗口内的数据进行聚合，返回一个新的包含(K,V)键值对的DStream，其中每个value都是各个key经过func聚合后的结果。注意：如果不指定numTasks，其值将使用Spark的默认并行任务数（本地模式下为2，集群模式下由 spark.default.parallelism决定）。当然，你也可以通过numTasks来指定任务个数。
reduceByKeyAndWindow(func, invFunc,windowLength,slideInterval, [numTasks])        和前面的reduceByKeyAndWindow() 类似，只是这个版本会用之前滑动窗口计算结果，递增地计算每个窗口的归约结果。当新的数据进入窗口时，这些values会被输入func做归约计算，而这些数据离开窗口时，对应的这些values又会被输入 invFunc 做”反归约”计算。举个简单的例子，就是把新进入窗口数据中各个单词个数“增加”到各个单词统计结果上，同时把离开窗口数据中各个单词的统计个数从相应的统计结果中“减掉”。不过，你的自己定义好”反归约”函数，即：该算子不仅有归约函数（见参数func），还得有一个对应的”反归约”函数（见参数中的 invFunc）。和前面的reduceByKeyAndWindow() 类似，该算子也有一个可选参数numTasks来指定并行任务数。注意，这个算子需要配置好检查点（checkpointing）才能用。
countByValueAndWindow(windowLength,slideInterval, [numTasks])                     基于包含(K, V)键值对的DStream，返回新的包含(K, Long)键值对的DStream。其中的Long value都是滑动窗口内key出现次数的计数。和前面的reduceByKeyAndWindow() 类似，该算子也有一个可选参数numTasks来指定并行任务数。
============================================================================      =============

Join 算子
-----------------------------

最后，值得一提的是，你在Spark Streaming中做各种关联（join）操作非常简单。

流-流（Stream-stream）关联
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

一个数据流可以和另一个数据流直接关联。

**Scala**

.. code-block:: Scala

  val stream1: DStream[String, String] = ...
  val stream2: DStream[String, String] = ...
  val joinedStream = stream1.join(stream2)

**Java**

.. code-block:: Java

  JavaPairDStream<String, String> stream1 = ...
  JavaPairDStream<String, String> stream2 = ...
  JavaPairDStream<String, Tuple2<String, String>> joinedStream = stream1.join(stream2);

**Python**

.. code-block:: Python

  stream1 = ...
  stream2 = ...
  joinedStream = stream1.join(stream2)

上面代码中，stream1的每个批次中的RDD会和stream2相应批次中的RDD进行join。同样，你可以类似地使用 leftOuterJoin, rightOuterJoin, fullOuterJoin 等。此外，你还可以基于窗口来join不同的数据流，其实现也很简单，如下；）

**Scala**

.. code-block:: Scala

  val windowedStream1 = stream1.window(Seconds(20))
  val windowedStream2 = stream2.window(Minutes(1))
  val joinedStream = windowedStream1.join(windowedStream2)

**Java**

.. code-block:: Java

  JavaPairDStream<String, String> windowedStream1 = stream1.window(Durations.seconds(20));
  JavaPairDStream<String, String> windowedStream2 = stream2.window(Durations.minutes(1));
  JavaPairDStream<String, Tuple2<String, String>> joinedStream = windowedStream1.join(windowedStream2);

**Python**

.. code-block:: Python

  windowedStream1 = stream1.window(20)
  windowedStream2 = stream2.window(60)
  joinedStream = windowedStream1.join(windowedStream2)


流-数据集(stream-dataset)关联
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

其实这种情况已经在前面的DStream.transform算子中介绍过了，这里再举个基于滑动窗口的例子。

**Scala**

.. code-block:: Scala

  val dataset: RDD[String, String] = ...
  val windowedStream = stream.window(Seconds(20))...
  val joinedStream = windowedStream.transform { rdd => rdd.join(dataset) }

**Java**

.. code-block:: Java

  JavaPairRDD<String, String> dataset = ...
  JavaPairDStream<String, String> windowedStream = stream.window(Durations.seconds(20));
  JavaPairDStream<String, String> joinedStream = windowedStream.transform(rdd -> rdd.join(dataset));

**Python**

.. code-block:: Python

  dataset = ... # some RDD
  windowedStream = stream.window(20)
  joinedStream = windowedStream.transform(lambda rdd: rdd.join(dataset))

实际上，在上面代码里，你可以动态地该表join的数据集（dataset）。传给tranform算子的操作函数会在每个批次重新求值，所以每次该函数都会用最新的dataset值，所以不同批次间你可以改变dataset的值。

完整的DStream transformation算子列表见API文档。Scala请参考 DStream 和 PairDStreamFunctions. Java请参考 JavaDStream 和 JavaPairDStream. Python见 DStream。


DStream输出算子
=============================

输出算子可以将 DStream 的数据推送到外部系统，如：数据库或者文件系统。因为输出算子会将最终完成转换的数据输出到外部系统，因此只有输出算子调用时，才会真正触发 DStream transformation 算子的真正执行（这一点类似于RDD 的action算子）。目前所支持的输出算子如下表：

=====================================     ================
输出算子                                    用途
=====================================     ================
print()                                   在驱动器（driver）节点上打印DStream每个批次中的头十个元素。Python API 对应的Python API为 pprint()
saveAsTextFiles(prefix, [suffix])         将DStream的内容保存到文本文件。每个批次一个文件，各文件命名规则为 “prefix-TIME_IN_MS[.suffix]”
saveAsObjectFiles(prefix, [suffix])       将DStream内容以序列化Java对象的形式保存到顺序文件中。每个批次一个文件，各文件命名规则为 “prefix-TIME_IN_MS[.suffix]”Python API 暂不支持Python
saveAsHadoopFiles(prefix, [suffix])       将DStream内容保存到Hadoop文件中。每个批次一个文件，各文件命名规则为 “prefix-TIME_IN_MS[.suffix]”Python API 暂不支持Python
foreachRDD(func)                          这是最通用的输出算子了，该算子接收一个函数func，func将作用于DStream的每个RDD上。func应该实现将每个RDD的数据推到外部系统中，比如：保存到文件或者写到数据库中。注意，func函数是在streaming应用的驱动器进程中执行的，所以如果其中包含RDD的action算子，就会触发对DStream中RDDs的实际计算过程。
=====================================     ================

使用foreachRDD的设计模式
------------------------------

DStream.foreachRDD是一个非常强大的原生工具函数，用户可以基于此算子将DStream数据推送到外部系统中。不过用户需要了解如何正确而高效地使用这个工具。以下列举了一些常见的错误。

通常，对外部系统写入数据需要一些连接对象（如：远程server的TCP连接），以便发送数据给远程系统。因此，开发人员可能会不经意地在Spark驱动器（driver）进程中创建一个连接对象，然后又试图在Spark worker节点上使用这个连接。如下例所示：

**Scala**

.. code-block:: Scala

  dstream.foreachRDD { rdd =>
    val connection = createNewConnection()  // 这行在驱动器（driver）进程执行
    rdd.foreach { record =>
      connection.send(record) // 而这行将在worker节点上执行
    }
  }

**Java**

.. code-block:: Java

  dstream.foreachRDD(rdd -> {
    Connection connection = createNewConnection(); // executed at the driver
    rdd.foreach(record -> {
      connection.send(record); // executed at the worker
    });
  });

**Python**

.. code-block:: Python

  def sendRecord(rdd):
      connection = createNewConnection()  # executed at the driver
      rdd.foreach(lambda record: connection.send(record))
      connection.close()

  dstream.foreachRDD(sendRecord)

这段代码是错误的，因为它需要把连接对象序列化，再从驱动器节点发送到worker节点。而这些连接对象通常都是不能跨节点（机器）传递的。比如，连接对象通常都不能序列化，或者在另一个进程中反序列化后再次初始化（连接对象通常都需要初始化，因此从驱动节点发到worker节点后可能需要重新初始化）等。解决此类错误的办法就是在worker节点上创建连接对象。

然而，有些开发人员可能会走到另一个极端 – 为每条记录都创建一个连接对象，例如：

**Scala**

.. code-block:: Scala

  dstream.foreachRDD { rdd =>
    rdd.foreach { record =>
      val connection = createNewConnection()
      connection.send(record)
      connection.close()
    }
  }

**Java**

.. code-block:: Java

  dstream.foreachRDD(rdd -> {
    rdd.foreach(record -> {
      Connection connection = createNewConnection();
      connection.send(record);
      connection.close();
    });
  });

**Python**

.. code-block:: Python

  def sendRecord(record):
      connection = createNewConnection()
      connection.send(record)
      connection.close()

  dstream.foreachRDD(lambda rdd: rdd.foreach(sendRecord))

一般来说，连接对象是有时间和资源开销限制的。因此，对每条记录都进行一次连接对象的创建和销毁会增加很多不必要的开销，同时也大大减小了系统的吞吐量。一个比较好的解决方案是使用 rdd.foreachPartition – 为RDD的每个分区创建一个单独的连接对象，示例如下：

**Scala**

.. code-block:: Scala

  dstream.foreachRDD { rdd =>
    rdd.foreachPartition { partitionOfRecords =>
      val connection = createNewConnection()
      partitionOfRecords.foreach(record => connection.send(record))
      connection.close()
    }
  }

**Java**

.. code-block:: Java

  dstream.foreachRDD(rdd -> {
    rdd.foreachPartition(partitionOfRecords -> {
      Connection connection = createNewConnection();
      while (partitionOfRecords.hasNext()) {
        connection.send(partitionOfRecords.next());
      }
      connection.close();
    });
  });

**Python**

.. code-block:: Python

  def sendPartition(iter):
      connection = createNewConnection()
      for record in iter:
          connection.send(record)
      connection.close()

  dstream.foreachRDD(lambda rdd: rdd.foreachPartition(sendPartition))

这样一来，连接对象的创建开销就摊到很多条记录上了。

最后，还有一个更优化的办法，就是在多个RDD批次之间复用连接对象。开发者可以维护一个静态连接池来保存连接对象，以便在不同批次的多个RDD之间共享同一组连接对象，示例如下：

**Scala**

.. code-block:: Scala

  dstream.foreachRDD { rdd =>
    rdd.foreachPartition { partitionOfRecords =>
      // ConnectionPool 是一个静态的、懒惰初始化的连接池
      val connection = ConnectionPool.getConnection()
      partitionOfRecords.foreach(record => connection.send(record))
      ConnectionPool.returnConnection(connection)  // 将连接返还给连接池，以便后续复用之
    }
  }

**Java**

.. code-block:: Java

  dstream.foreachRDD(rdd -> {
    rdd.foreachPartition(partitionOfRecords -> {
      // ConnectionPool is a static, lazily initialized pool of connections
      Connection connection = ConnectionPool.getConnection();
      while (partitionOfRecords.hasNext()) {
        connection.send(partitionOfRecords.next());
      }
      ConnectionPool.returnConnection(connection); // return to the pool for future reuse
    });
  });

**Python**

.. code-block:: Python

  def sendPartition(iter):
      # ConnectionPool is a static, lazily initialized pool of connections
      connection = ConnectionPool.getConnection()
      for record in iter:
          connection.send(record)
      # return to the pool for future reuse
      ConnectionPool.returnConnection(connection)

  dstream.foreachRDD(lambda rdd: rdd.foreachPartition(sendPartition))

.. attention:: 连接池中的连接应该是懒惰创建的，并且有确定的超时时间，超时后自动销毁。这个实现应该是目前发送数据最高效的实现方式。

**其他要点:**

* DStream的转化执行也是懒惰的，需要输出算子来触发，这一点和RDD的懒惰执行由action算子触发很类似。特别地，DStream输出算子中包含的RDD action算子会强制触发对所接收数据的处理。因此，如果你的Streaming应用中没有输出算子，或者你用了dstream.foreachRDD(func)却没有在func中调用RDD action算子，那么这个应用只会接收数据，而不会处理数据，接收到的数据最后只是被简单地丢弃掉了。
* 默认地，输出算子只能一次执行一个，且按照它们在应用程序代码中定义的顺序执行。

DataFrame 和 SQL 算子
=============================

在Streaming应用中可以调用DataFrames and SQL来处理流式数据。开发者可以用通过StreamingContext中的SparkContext对象来创建一个SQLContext，并且，开发者需要确保一旦驱动器（driver）故障恢复后，该SQLContext对象能重新创建出来。同样，你还是可以使用懒惰创建的单例模式来实例化SQLContext，如下面的代码所示，这里我们将最开始的那个小栗子做了一些修改，使用DataFrame和SQL来统计单词计数。其实就是，将每个RDD都转化成一个DataFrame，然后注册成临时表，再用SQL查询这些临时表。

**Scala**

.. code-block:: Scala

  /** streaming应用中调用DataFrame算子 */

  val words: DStream[String] = ...

  words.foreachRDD { rdd =>

    // 获得SQLContext单例
    val sqlContext = SQLContext.getOrCreate(rdd.sparkContext)
    import sqlContext.implicits._

    // 将RDD[String] 转为 DataFrame
    val wordsDataFrame = rdd.toDF("word")

    // DataFrame注册为临时表
    wordsDataFrame.registerTempTable("words")

    // 再用SQL语句查询，并打印出来
    val wordCountsDataFrame =
      sqlContext.sql("select word, count(*) as total from words group by word")
    wordCountsDataFrame.show()
  }

这里有完整代码：source code。

**Java**

.. code-block:: Java

  /** Java Bean class for converting RDD to DataFrame */
  public class JavaRow implements java.io.Serializable {
    private String word;

    public String getWord() {
      return word;
    }

    public void setWord(String word) {
      this.word = word;
    }
  }

  ...

  /** DataFrame operations inside your streaming program */

  JavaDStream<String> words = ...

  words.foreachRDD((rdd, time) -> {
    // Get the singleton instance of SparkSession
    SparkSession spark = SparkSession.builder().config(rdd.sparkContext().getConf()).getOrCreate();

    // Convert RDD[String] to RDD[case class] to DataFrame
    JavaRDD<JavaRow> rowRDD = rdd.map(word -> {
      JavaRow record = new JavaRow();
      record.setWord(word);
      return record;
    });
    DataFrame wordsDataFrame = spark.createDataFrame(rowRDD, JavaRow.class);

    // Creates a temporary view using the DataFrame
    wordsDataFrame.createOrReplaceTempView("words");

    // Do word count on table using SQL and print it
    DataFrame wordCountsDataFrame =
      spark.sql("select word, count(*) as total from words group by word");
    wordCountsDataFrame.show();
  });

这里有完整代码：source code。

**Python**

.. code-block:: Python

  # Lazily instantiated global instance of SparkSession
  def getSparkSessionInstance(sparkConf):
      if ("sparkSessionSingletonInstance" not in globals()):
          globals()["sparkSessionSingletonInstance"] = SparkSession \
              .builder \
              .config(conf=sparkConf) \
              .getOrCreate()
      return globals()["sparkSessionSingletonInstance"]

  ...

  # DataFrame operations inside your streaming program

  words = ... # DStream of strings

  def process(time, rdd):
      print("========= %s =========" % str(time))
      try:
          # Get the singleton instance of SparkSession
          spark = getSparkSessionInstance(rdd.context.getConf())

          # Convert RDD[String] to RDD[Row] to DataFrame
          rowRdd = rdd.map(lambda w: Row(word=w))
          wordsDataFrame = spark.createDataFrame(rowRdd)

          # Creates a temporary view using the DataFrame
          wordsDataFrame.createOrReplaceTempView("words")

          # Do word count on table using SQL and print it
          wordCountsDataFrame = spark.sql("select word, count(*) as total from words group by word")
          wordCountsDataFrame.show()
      except:
          pass

  words.foreachRDD(process)

这里有完整代码：source code。


你也可以在其他线程里执行SQL查询（异步查询，即：执行SQL查询的线程和运行StreamingContext的线程不同）。不过这种情况下，你需要确保查询的时候 StreamingContext 没有把所需的数据丢弃掉，否则StreamingContext有可能已将老的RDD数据丢弃掉了，那么异步查询的SQL语句也可能无法得到查询结果。举个栗子，如果你需要查询上一个批次的数据，但是你的SQL查询可能要执行5分钟，那么你就需要StreamingContext至少保留最近5分钟的数据：streamingContext.remember(Minutes(5)) （这是Scala为例，其他语言差不多）

更多DataFrame和SQL的文档见这里： DataFrames and SQL


MLlib 算子
=============================

MLlib 提供了很多机器学习算法。首先，你需要关注的是流式计算相关的机器学习算法（如：Streaming Linear Regression, Streaming KMeans），这些流式算法可以在流式数据上一边学习训练模型，一边用最新的模型处理数据。除此以外，对更多的机器学习算法而言，你需要离线训练这些模型，然后将训练好的模型用于在线的流式数据。详见MLlib。


缓存/持久化
=============================

和RDD类似，DStream也支持将数据持久化到内存中。只需要调用 DStream的persist() 方法，该方法内部会自动调用DStream中每个RDD的persist方法进而将数据持久化到内存中。这对于可能需要计算很多次的DStream非常有用（例如：对于同一个批数据调用多个算子）。对于基于滑动窗口的算子，如：reduceByWindow和reduceByKeyAndWindow，或者有状态的算子，如：updateStateByKey，数据持久化就更重要了。因此，滑动窗口算子产生的DStream对象默认会自动持久化到内存中（不需要开发者调用persist）。

对于从网络接收数据的输入数据流（如：Kafka、Flume、socket等），默认的持久化级别会将数据持久化到两个不同的节点上互为备份副本，以便支持容错。

注意，与RDD不同的是，DStream的默认持久化级别是将数据序列化到内存中。进一步的讨论见性能调优这一小节。关于持久化级别（或者存储级别）的更详细说明见Spark编程指南（Spark Programming Guide）。


检查点
=============================

一般来说Streaming 应用都需要7*24小时长期运行，所以必须对一些与业务逻辑无关的故障有很好的容错（如：系统故障、JVM崩溃等）。对于这些可能性，Spark Streaming 必须在检查点保存足够的信息到一些可容错的外部存储系统中，以便能够随时从故障中恢复回来。所以，检查点需要保存以下两种数据：

* 元数据检查点（Metadata checkpointing） – 保存流式计算逻辑的定义信息到外部可容错存储系统（如：HDFS）。主要用途是用于在故障后回复应用程序本身（后续详谈）。元数包括：
    * Configuration – 创建Streaming应用程序的配置信息。
    * DStream operations – 定义流式处理逻辑的DStream操作信息。
    * Incomplete batches – 已经排队但未处理完的批次信息。
* 数据检查点（Data checkpointing） – 将生成的RDD保存到可靠的存储中。这对一些需要跨批次组合数据或者有状态的算子来说很有必要。在这种转换算子中，往往新生成的RDD是依赖于前几个批次的RDD，因此随着时间的推移，有可能产生很长的依赖链条。为了避免在恢复数据的时候需要恢复整个依赖链条上所有的数据，检查点需要周期性地保存一些中间RDD状态信息，以斩断无限制增长的依赖链条和恢复时间。

总之，元数据检查点主要是为了恢复驱动器节点上的故障，而数据或RDD检查点是为了支持对有状态转换操作的恢复。

何时启用检查点
----------------

如果有以下情况出现，你就必须启用检查点了：

* 使用了有状态的转换算子（Usage of stateful transformations） – 不管是用了 updateStateByKey 还是用了 reduceByKeyAndWindow（有”反归约”函数的那个版本），你都必须配置检查点目录来周期性地保存RDD检查点。
* 支持驱动器故障中恢复（Recovering from failures of the driver running the application） – 这时候需要元数据检查点以便恢复流式处理的进度信息。

注意，一些简单的流式应用，如果没有用到前面所说的有状态转换算子，则完全可以不开启检查点。不过这样的话，驱动器（driver）故障恢复后，有可能会丢失部分数据（有些已经接收但还未处理的数据可能会丢失）。不过通常这点丢失时可接受的，很多Spark Streaming应用也是这样运行的。对非Hadoop环境的支持未来还会继续改进。

如何配置检查点
----------------

检查点的启用，只需要设置好保存检查点信息的检查点目录即可，一般会会将这个目录设为一些可容错的、可靠性较高的文件系统（如：HDFS、S3等）。开发者只需要调用 streamingContext.checkpoint(checkpointDirectory)。设置好检查点，你就可以使用前面提到的有状态转换算子了。另外，如果你需要你的应用能够支持从驱动器故障中恢复，你可能需要重写部分代码，实现以下行为：

* 如果程序是首次启动，就需要new一个新的StreamingContext，并定义好所有的数据流处理，然后调用StreamingContext.start()。
* 如果程序是故障后重启，就需要从检查点目录中的数据中重新构建StreamingContext对象。


不过这个行为可以用StreamingContext.getOrCreate来实现，示例如下：

**Scala**

.. code-block:: Scala

  // 首次创建StreamingContext并定义好数据流处理逻辑
  def functionToCreateContext(): StreamingContext = {
      val ssc = new StreamingContext(...)  // 新建一个StreamingContext对象
      val lines = ssc.socketTextStream(...) // 创建DStreams
      ...
      ssc.checkpoint(checkpointDirectory)  // 设置好检查点目录
      ssc
  }

  // 创建新的StreamingContext对象，或者从检查点构造一个
  val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

  // Do additional setup on context that needs to be done,
  // irrespective of whether it is being started or restarted
  context. ...

  // 启动StreamingContext对象
  context.start()
  context.awaitTermination()

如果 checkpointDirectory 目录存在，则context对象会从检查点数据重新构建出来。如果该目录不存在（如：首次运行），则 functionToCreateContext 函数会被调用，创建一个新的StreamingContext对象并定义好DStream数据流。完整的示例请参见RecoverableNetworkWordCount，这个例子会将网络数据中的单词计数统计结果添加到一个文件中。

除了使用getOrCreate之外，开发者还需要确保驱动器进程能在故障后重启。这一点只能由应用的部署环境基础设施来保证。进一步的讨论见部署（Deployment）这一节。

另外需要注意的是，RDD检查点会增加额外的保存数据的开销。这可能会导致数据流的处理时间变长。因此，你必须仔细的调整检查点间隔时间。如果批次间隔太小（比如：1秒），那么对每个批次保存检查点数据将大大减小吞吐量。另一方面，检查点保存过于频繁又会导致血统信息和任务个数的增加，这同样会影响系统性能。对于需要RDD检查点的有状态转换算子，默认的间隔是批次间隔的整数倍，且最小10秒。开发人员可以这样来自定义这个间隔：dstream.checkpoint(checkpointInterval)。一般推荐设为批次间隔时间的5~10倍。


累加器, 广播变量以及检查点
====================================================
Accumulators and Broadcast variables cannot be recovered from checkpoint in Spark Streaming. If you enable checkpointing and use Accumulators or Broadcast variables as well, you’ll have to create lazily instantiated singleton instances for Accumulators and Broadcast variables so that they can be re-instantiated after the driver restarts on failure. This is shown in the following example.

**Scala**

.. code-block:: Scala

  object WordBlacklist {

    @volatile private var instance: Broadcast[Seq[String]] = null

    def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
      if (instance == null) {
        synchronized {
          if (instance == null) {
            val wordBlacklist = Seq("a", "b", "c")
            instance = sc.broadcast(wordBlacklist)
          }
        }
      }
      instance
    }
  }

  object DroppedWordsCounter {

    @volatile private var instance: LongAccumulator = null

    def getInstance(sc: SparkContext): LongAccumulator = {
      if (instance == null) {
        synchronized {
          if (instance == null) {
            instance = sc.longAccumulator("WordsInBlacklistCounter")
          }
        }
      }
      instance
    }
  }

  wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) =>
    // Get or register the blacklist Broadcast
    val blacklist = WordBlacklist.getInstance(rdd.sparkContext)
    // Get or register the droppedWordsCounter Accumulator
    val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
    // Use blacklist to drop words and use droppedWordsCounter to count them
    val counts = rdd.filter { case (word, count) =>
      if (blacklist.value.contains(word)) {
        droppedWordsCounter.add(count)
        false
      } else {
        true
      }
    }.collect().mkString("[", ", ", "]")
    val output = "Counts at time " + time + " " + counts
  })

See the full source code.

**Java**

.. code-block:: Java

  class JavaWordBlacklist {

    private static volatile Broadcast<List<String>> instance = null;

    public static Broadcast<List<String>> getInstance(JavaSparkContext jsc) {
      if (instance == null) {
        synchronized (JavaWordBlacklist.class) {
          if (instance == null) {
            List<String> wordBlacklist = Arrays.asList("a", "b", "c");
            instance = jsc.broadcast(wordBlacklist);
          }
        }
      }
      return instance;
    }
  }

  class JavaDroppedWordsCounter {

    private static volatile LongAccumulator instance = null;

    public static LongAccumulator getInstance(JavaSparkContext jsc) {
      if (instance == null) {
        synchronized (JavaDroppedWordsCounter.class) {
          if (instance == null) {
            instance = jsc.sc().longAccumulator("WordsInBlacklistCounter");
          }
        }
      }
      return instance;
    }
  }

  wordCounts.foreachRDD((rdd, time) -> {
    // Get or register the blacklist Broadcast
    Broadcast<List<String>> blacklist = JavaWordBlacklist.getInstance(new JavaSparkContext(rdd.context()));
    // Get or register the droppedWordsCounter Accumulator
    LongAccumulator droppedWordsCounter = JavaDroppedWordsCounter.getInstance(new JavaSparkContext(rdd.context()));
    // Use blacklist to drop words and use droppedWordsCounter to count them
    String counts = rdd.filter(wordCount -> {
      if (blacklist.value().contains(wordCount._1())) {
        droppedWordsCounter.add(wordCount._2());
        return false;
      } else {
        return true;
      }
    }).collect().toString();
    String output = "Counts at time " + time + " " + counts;
  }

See the full source code.

**Python**

.. code-block:: Python

  def getWordBlacklist(sparkContext):
      if ("wordBlacklist" not in globals()):
          globals()["wordBlacklist"] = sparkContext.broadcast(["a", "b", "c"])
      return globals()["wordBlacklist"]

  def getDroppedWordsCounter(sparkContext):
      if ("droppedWordsCounter" not in globals()):
          globals()["droppedWordsCounter"] = sparkContext.accumulator(0)
      return globals()["droppedWordsCounter"]

  def echo(time, rdd):
      # Get or register the blacklist Broadcast
      blacklist = getWordBlacklist(rdd.context)
      # Get or register the droppedWordsCounter Accumulator
      droppedWordsCounter = getDroppedWordsCounter(rdd.context)

      # Use blacklist to drop words and use droppedWordsCounter to count them
      def filterFunc(wordCount):
          if wordCount[0] in blacklist.value:
              droppedWordsCounter.add(wordCount[1])
              False
          else:
              True

      counts = "Counts at time %s %s" % (time, rdd.filter(filterFunc).collect())

  wordCounts.foreachRDD(echo)

See the full source code.


部署应用程序
=============================

本节中将主要讨论一下如何部署Spark Streaming应用。

前提条件
-----------------------------

要运行一个Spark Streaming 应用，你首先需要具备以下条件：

* 集群以及集群管理器 – 这是一般Spark应用的基本要求，详见 deployment guide。
* 给Spark应用打个JAR包 – 你需要将你的应用打成一个JAR包。如果使用spark-submit 提交应用，那么你不需要提供Spark和Spark Streaming的相关JAR包。但是，如果你使用了高级数据源（advanced sources – 如：Kafka、Flume、Twitter等），那么你需要将这些高级数据源相关的JAR包及其依赖一起打包并部署。例如，如果你使用了TwitterUtils，那么就必须将spark-streaming-twitter_2.10及其相关依赖都打到应用的JAR包中。
* 为执行器（executor）预留足够的内存 – 执行器必须配置预留好足够的内存，因为接受到的数据都得存在内存里。注意，如果某些窗口长度达到10分钟，那也就是说你的系统必须知道保留10分钟的数据在内存里。可见，到底预留多少内存是取决于你的应用处理逻辑的。
* 配置检查点 – 如果你的流式应用需要检查点，那么你需要配置一个Hadoop API兼容的可容错存储目录作为检查点目录，流式应用的信息会写入这个目录，故障恢复时会用到这个目录下的数据。详见前面的检查点小节。
* 配置驱动程序自动重启 – 流式应用自动恢复的前提就是，部署基础设施能够监控驱动器进程，并且能够在其故障时，自动重启之。不同的集群管理器有不同的工具来实现这一功能：
    * Spark独立部署 – Spark独立部署集群可以支持将Spark应用的驱动器提交到集群的某个worker节点上运行。同时，Spark的集群管理器可以对该驱动器进程进行监控，一旦驱动器退出且返回非0值，或者因worker节点原始失败，Spark集群管理器将自动重启这个驱动器。详见Spark独立部署指南（Spark Standalone guide）。
* YARN – YARN支持和独立部署类似的重启机制。详细请参考YARN的文档。
    * Mesos – Mesos上需要用Marathon来实现这一功能。
* 配置WAL（write ahead log）- 从Spark 1.2起，我们引入了write ahead log来提高容错性。如果启用这个功能，则所有接收到的数据都会以write ahead log形式写入配置好的检查点目录中。这样就能确保数据零丢失（容错语义有详细的讨论）。用户只需将 spark.streaming.receiver.writeAheadLog 设为true。不过，这同样可能会导致接收器的吞吐量下降。不过你可以启动多个接收器并行接收数据，从而提升整体的吞吐量（more receivers in parallel）。另外，建议在启用WAL后禁用掉接收数据多副本功能，因为WAL其实已经是存储在一个多副本存储系统中了。你只需要把存储级别设为 StorageLevel.MEMORY_AND_DISK_SER。如果是使用S3（或者其他不支持flushing的文件系统）存储WAL，一定要记得启用这两个标识：spark.streaming.driver.writeAheadLog.closeFileAfterWrite 和 spark.streaming.receiver.writeAheadLog.closeFileAfterWrite。更详细请参考： Spark Streaming Configuration。
* 设置好最大接收速率 – 如果集群可用资源不足以跟上接收数据的速度，那么可以在接收器设置一下最大接收速率，即：每秒接收记录的条数。相关的主要配置有：spark.streaming.receiver.maxRate，如果使用Kafka Direct API 还需要设置 spark.streaming.kafka.maxRatePerPartition。从Spark 1.5起，我们引入了backpressure的概念来动态地根据集群处理速度，评估并调整该接收速率。用户只需将 spark.streaming.backpressure.enabled设为true即可启用该功能。

升级应用程序代码
---------------------

升级Spark Streaming应用程序代码，可以使用以下两种方式：

* 新的Streaming程序和老的并行跑一段时间，新程序完成初始化以后，再关闭老的。注意，这种方式适用于能同时发送数据到多个目标的数据源（即：数据源同时将数据发给新老两个Streaming应用程序）。
* 老程序能够优雅地退出（参考  StreamingContext.stop(...) or JavaStreamingContext.stop(...) ），即：确保所收到的数据都已经处理完毕后再退出。然后再启动新的Streaming程序，而新程序将接着在老程序退出点上继续拉取数据。注意，这种方式需要数据源支持数据缓存（或者叫数据堆积，如：Kafka、Flume），因为在新旧程序交接的这个空档时间，数据需要在数据源处缓存。目前还不能支持从检查点重启，因为检查点存储的信息包含老程序中的序列化对象信息，在新程序中将其反序列化可能会出错。这种情况下，只能要么指定一个新的检查点目录，要么删除老的检查点目录。


应用程序监控
=============================

除了Spark自身的监控能力（monitoring capabilities）之外，对Spark Streaming还有一些额外的监控功能可用。如果实例化了StreamingContext，那么你可以在Spark web UI上看到多出了一个Streaming tab页，上面显示了正在运行的接收器（是否活跃，接收记录的条数，失败信息等）和处理完的批次信息（批次处理时间，查询延时等）。这些信息都可以用来监控streaming应用。

web UI上有两个度量特别重要：

* 批次处理耗时（Processing Time） – 处理单个批次耗时
* 批次调度延时（Scheduling Delay） -各批次在队列中等待时间（等待上一个批次处理完）

如果批次处理耗时一直比批次间隔时间大，或者批次调度延时持续上升，就意味着系统处理速度跟不上数据接收速度。这时候你就得考虑一下怎么把批次处理时间降下来（reducing）。

Spark Streaming 程序的处理进度可以用StreamingListener接口来监听，这个接口可以监听到接收器的状态和处理时间。不过需要注意的是，这是一个developer API接口，换句话说这个接口未来很可能会变动（可能会增加更多度量信息）。


*****************************
性能调优
*****************************

要获得Spark Streaming应用的最佳性能需要一点点调优工作。本节将深入解释一些能够改进Streaming应用性能的配置和参数。总体上来说，你需要考虑这两方面的事情：

1. 提高集群资源利用率，减少单批次处理耗时。
2. 设置合适的批次大小，以便使数据处理速度能跟上数据接收速度。


减少批次处理时间
=============================

有不少优化手段都可以减少Spark对每个批次的处理时间。细节将在优化指南（Tuning Guide）中详谈。这里仅列举一些最重要的。

数据接收并发度
-------------------

跨网络接收数据（如：从Kafka、Flume、socket等接收数据）需要在Spark中序列化并存储数据。

如果接收数据的过程是系统瓶颈，那么可以考虑增加数据接收的并行度。注意，每个输入DStream只包含一个单独的接收器（receiver，运行约worker节点），每个接收器单独接收一路数据流。所以，配置多个输入DStream就能从数据源的不同分区分别接收多个数据流。例如，可以将从Kafka拉取两个topic的数据流分成两个Kafka输入数据流，每个数据流拉取其中一个topic的数据，这样一来会同时有两个接收器并行地接收数据，因而增加了总体的吞吐量。同时，另一方面我们又可以把这些DStream数据流合并成一个，然后可以在合并后的DStream上使用任何可用的transformation算子。示例代码如下：

**Scala**

.. code-block:: Scala

  val numStreams = 5
  val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
  val unifiedStream = streamingContext.union(kafkaStreams)
  unifiedStream.print()

**Java**

.. code-block:: Java

  int numStreams = 5;
  List<JavaPairDStream<String, String>> kafkaStreams = new ArrayList<>(numStreams);
  for (int i = 0; i < numStreams; i++) {
    kafkaStreams.add(KafkaUtils.createStream(...));
  }
  JavaPairDStream<String, String> unifiedStream = streamingContext.union(kafkaStreams.get(0), kafkaStreams.subList(1, kafkaStreams.size()));
  unifiedStream.print();

**Python**

.. code-block:: Python

  numStreams = 5
  kafkaStreams = [KafkaUtils.createStream(...) for _ in range (numStreams)]
  unifiedStream = streamingContext.union(*kafkaStreams)
  unifiedStream.pprint()

另一个可以考虑优化的参数就是接收器的阻塞间隔，该参数由配置参数（configuration parameter）spark.streaming.blockInterval决定。大多数接收器都会将数据合并成一个个数据块，然后再保存到spark内存中。对于map类算子来说，每个批次中数据块的个数将会决定处理这批数据并行任务的个数，每个接收器每批次数据处理任务数约等于 （批次间隔 / 数据块间隔）。例如，对于2秒的批次间隔，如果数据块间隔为200ms，则创建的并发任务数为10。如果任务数太少（少于单机cpu core个数），则资源利用不够充分。如需增加这个任务数，对于给定的批次间隔来说，只需要减少数据块间隔即可。不过，我们还是建议数据块间隔至少要50ms，否则任务的启动开销占比就太高了。

另一个切分接收数据流的方法是，显示地将输入数据流划分为多个分区（使用 inputStream.repartition(<number of partitions>)）。该操作会在处理前，将数据散开重新分发到集群中多个节点上。

数据处理并发度
---------------------

在计算各个阶段（stage）中，任何一个阶段的并发任务数不足都有可能造成集群资源利用率低。例如，对于reduce类的算子，如：reduceByKey 和 reduceByKeyAndWindow，其默认的并发任务数是由 spark.default.parallelism 决定的。你既可以修改这个默认值（spark.default.parallelism），也可以通过参数指定这个并发数量（见PairDStreamFunctions）。

数据序列化
--------------------

调整数据的序列化格式可以大大减少数据序列化的开销。在 Spark Streaming 中主要有两种类型的数据需要序列化：

* 输入数据: 默认地，接收器收到的数据是以 StorageLevel.MEMORY_AND_DISK_SER_2 的存储级别存储到执行器（executor）内存中的。也就是说，收到的数据会被序列化以减少GC开销，同时保存两个副本以容错。同时，数据会优先保存在内存里，当内存不足时才吐出到磁盘上。很明显，这个过程中会有数据序列化的开销 – 接收器首先将收到的数据反序列化，然后再以spark所配置指定的格式来序列化数据。
* Streaming 算子所生产的持久化的RDDs: Streaming计算所生成的RDD可能会持久化到内存中。例如，基于窗口的算子会将数据持久化到内存，因为窗口数据可能会多次处理。所不同的是，spark core默认用 StorageLevel.MEMORY_ONLY 级别持久化RDD数据，而spark streaming默认使用StorageLevel.MEMORY_ONLY_SER 级别持久化接收到的数据，以便尽量减少GC开销。

不管是上面哪一种数据，都可以使用Kryo序列化来减少CPU和内存开销，详见Spark Tuning Guide。另，对于Kryo，你可以考虑这些优化：注册自定义类型，禁用对象引用跟踪（详见Configuration Guide）。

在一些特定的场景下，如果数据量不是很大，那么你可以考虑不用序列化格式，不过你需要注意的是取消序列化是否会导致大量的GC开销。例如，如果你的批次间隔比较短（几秒）并且没有使用基于窗口的算子，这种情况下你可以考虑禁用序列化格式。这样可以减少序列化的CPU开销以优化性能，同时GC的增长也不多。

任务启动开销
--------------------

如果每秒启动的任务数过多（比如每秒50个以上），那么将任务发送给slave节点的开销会明显增加，那么你也就很难达到亚秒级（sub-second）的延迟。不过以下两个方法可以减少任务的启动开销：

* 任务序列化（Task Serialization）: 使用Kryo来序列化任务，以减少任务本身的大小，从而提高发送任务的速度。任务的序列化格式是由 spark.closure.serializer 属性决定的。不过，目前还不支持闭包序列化，未来的版本可能会增加对此的支持。
* 执行模式（Execution mode）: Spark独立部署或者Mesos粗粒度模式下任务的启动时间比Mesos细粒度模式下的任务启动时间要短。详见Running on Mesos guide。

这些调整有可能能够减少100ms的批次处理时间，这也使得亚秒级的批次间隔成为可能。


设置合适的批次间隔
=========================

要想streaming应用在集群上稳定运行，那么系统处理数据的速度必须能跟上其接收数据的速度。换句话说，批次数据的处理速度应该和其生成速度一样快。对于特定的应用来说，可以从其对应的监控（monitoring）页面上观察验证，页面上显示的处理耗时应该要小于批次间隔时间。

根据 Spark Streaming 计算的性质，在一定的集群资源限制下，批次间隔的值会极大地影响系统的数据处理能力。例如，在 WordCountNetwork 示例中，对于特定的数据速率，一个系统可能能够在批次间隔为2秒时跟上数据接收速度，但如果把批次间隔改为500毫秒系统可能就处理不过来了。所以，批次间隔需要谨慎设置，以确保生产系统能够处理得过来。

要找出适合的批次间隔，你可以从一个比较保守的批次间隔值（如5~10秒）开始测试。要验证系统是否能跟上当前的数据接收速率，你可能需要检查一下端到端的批次处理延迟（可以看看Spark驱动器log4j日志中的Total delay，也可以用StreamingListener接口来检测）。如果这个延迟能保持和批次间隔差不多，那么系统基本就是稳定的。否则，如果这个延迟持久在增长，也就是说系统跟不上数据接收速度，那也就意味着系统不稳定。一旦系统文档下来后，你就可以尝试提高数据接收速度，或者减少批次间隔值。不过需要注意，瞬间的延迟增长可以只是暂时的，只要这个延迟后续会自动降下来就没有问题（如：降到小于批次间隔值）


内存调优
============================

Spark应用内存占用和GC调优已经在调优指南（Tuning Guide）中有详细的讨论。墙裂建议你读一读那篇文档。本节中，我们只是讨论一下几个专门用于Spark Streaming的调优参数。

Spark Streaming 应用在集群中占用的内存量严重依赖于具体所使用的tranformation算子。例如，如果想要用一个窗口算子操纵最近10分钟的数据，那么你的集群至少需要在内存里保留10分钟的数据；另一个例子是updateStateByKey，如果key很多的话，相对应的保存的key的state也会很多，而这些都需要占用内存。而如果你的应用只是做一个简单的 “映射-过滤-存储”（map-filter-store）操作的话，那需要的内存就很少了。

一般情况下，streaming 接收器接收到的数据会以 StorageLevel.MEMORY_AND_DISK_SER_2 这个存储级别存到spark中，也就是说，如果内存装不下，数据将被吐到磁盘上。数据吐到磁盘上会大大降低streaming应用的性能，因此还是建议根据你的应用处理的数据量，提供充足的内存。最好就是，一边小规模地放大内存，再观察评估，然后再放大，再评估。

另一个内存调优的方向就是垃圾回收。因为streaming应用往往都需要低延迟，所以肯定不希望出现大量的或耗时较长的JVM垃圾回收暂停。

以下是一些能够帮助你减少内存占用和GC开销的参数或手段：

* DStream持久化级别（Persistence Level of DStreams）: 前面数据序列化（Data Serialization）这小节已经提到过，默认streaming的输入RDD会被持久化成序列化的字节流。相对于非序列化数据，这样可以减少内存占用和GC开销。如果启用Kryo序列化，还能进一步减少序列化数据大小和内存占用量。如果你还需要进一步减少内存占用的话，可以开启数据压缩（通过spark.rdd.compress这个配置设定），只不过数据压缩会增加CPU消耗。
* 清除老数据（Clearing old data）: 默认情况下，所有的输入数据以及DStream的transformation算子产生的持久化RDD都是自动清理的。Spark Streaming会根据所使用的transformation算子来清理老数据。例如，你用了一个窗口操作处理最近10分钟的数据，那么Spark Streaming会保留至少10分钟的数据，并且会主动把更早的数据都删掉。当然，你可以设置 streamingContext.remember 以保留更长时间段的数据（比如：你可能会需要交互式地查询更老的数据）。
* CMS垃圾回收器（CMS Garbage Collector）: 为了尽量减少GC暂停的时间，我们强烈建议使用CMS垃圾回收器（concurrent mark-and-sweep GC）。虽然CMS GC会稍微降低系统的总体吞吐量，但我们仍建议使用它，因为CMS GC能使批次处理的时间保持在一个比较恒定的水平上。最后，你需要确保在驱动器（通过spark-submit中的–driver-java-options设置）和执行器（使用spark.executor.extraJavaOptions配置参数）上都设置了CMS GC。
* 其他提示: 如果还想进一步减少GC开销，以下是更进一步的可以尝试的手段：
    * 配合Tachyon使用堆外内存来持久化RDD。详见Spark编程指南（Spark Programming Guide）
    * 使用更多但是更小的执行器进程。这样GC压力就会分散到更多的JVM堆中。

**Important points to remember:**

A DStream is associated with a single receiver. For attaining read parallelism multiple receivers i.e. multiple DStreams need to be created. A receiver is run within an executor. It occupies one core. Ensure that there are enough cores for processing after receiver slots are booked i.e. spark.cores.max should take the receiver slots into account. The receivers are allocated to executors in a round robin fashion.

When data is received from a stream source, receiver creates blocks of data. A new block of data is generated every blockInterval milliseconds. N blocks of data are created during the batchInterval where N = batchInterval/blockInterval. These blocks are distributed by the BlockManager of the current executor to the block managers of other executors. After that, the Network Input Tracker running on the driver is informed about the block locations for further processing.

An RDD is created on the driver for the blocks created during the batchInterval. The blocks generated during the batchInterval are partitions of the RDD. Each partition is a task in spark. blockInterval== batchinterval would mean that a single partition is created and probably it is processed locally.

The map tasks on the blocks are processed in the executors (one that received the block, and another where the block was replicated) that has the blocks irrespective of block interval, unless non-local scheduling kicks in. Having bigger blockinterval means bigger blocks. A high value of spark.locality.wait increases the chance of processing a block on the local node. A balance needs to be found out between these two parameters to ensure that the bigger blocks are processed locally.

Instead of relying on batchInterval and blockInterval, you can define the number of partitions by calling inputDstream.repartition(n). This reshuffles the data in RDD randomly to create n number of partitions. Yes, for greater parallelism. Though comes at the cost of a shuffle. An RDD’s processing is scheduled by driver’s jobscheduler as a job. At a given point of time only one job is active. So, if one job is executing the other jobs are queued.

If you have two dstreams there will be two RDDs formed and there will be two jobs created which will be scheduled one after the another. To avoid this, you can union two dstreams. This will ensure that a single unionRDD is formed for the two RDDs of the dstreams. This unionRDD is then considered as a single job. However the partitioning of the RDDs is not impacted.

If the batch processing time is more than batchinterval then obviously the receiver’s memory will start filling up and will end up in throwing exceptions (most probably BlockNotFoundException). Currently there is no way to pause the receiver. Using SparkConf configuration spark.streaming.receiver.maxRate, rate of receiver can be limited.

*****************************
容错语义
*****************************

本节中，我们将讨论Spark Streaming应用在出现失败时的具体行为。

背景
===================

要理解Spark Streaming所提供的容错语义，我们首先需要回忆一下Spark RDD所提供的基本容错语义。

1. RDD是不可变的，可重算的，分布式数据集。每个RDD都记录了其创建算子的血统信息，其中每个算子都以可容错的数据集作为输入数据。
2. 如果RDD的某个分区因为节点失效而丢失，则该分区可以根据RDD的血统信息以及相应的原始输入数据集重新计算出来。
3. 假定所有RDD transformation算子计算过程都是确定性的，那么通过这些算子得到的最终RDD总是包含相同的数据，而与Spark集群的是否故障无关。

Spark主要操作一些可容错文件系统的数据，如：HDFS或S3。因此，所有从这些可容错数据源产生的RDD也是可容错的。然而，对于Spark Streaming并非如此，因为多数情况下Streaming需要从网络远端接收数据，这回导致Streaming的数据源并不可靠（尤其是对于使用了fileStream的应用）。要实现RDD相同的容错属性，数据接收就必须用多个不同worker节点上的Spark执行器来实现（默认副本因子是2）。因此一旦出现故障，系统需要恢复两种数据：

1. 接收并保存了副本的数据 – 数据不会因为单个worker节点故障而丢失，因为有副本！
2. 接收但尚未保存副本数据 – 因为数据并没有副本，所以一旦故障，只能从数据源重新获取。

此外，还有两种可能的故障类型需要考虑：

1. Worker节点故障 – 任何运行执行器的worker节点一旦故障，节点上内存中的数据都会丢失。如果这些节点上有接收器在运行，那么其包含的缓存数据也会丢失。
2. Driver节点故障 – 如果Spark Streaming的驱动节点故障，那么很显然SparkContext对象就没了，所有执行器及其内存数据也会丢失。

有了以上这些基本知识，下面我们就进一步了解一下Spark Streaming的容错语义。

定义
===================

流式系统的可靠度语义可以据此来分类：单条记录在系统中被处理的次数保证。一个流式系统可能提供保证必定是以下三种之一（不管系统是否出现故障）：

1. 至多一次（At most once）: 每条记录要么被处理一次，要么就没有处理。
2. 至少一次（At least once）: 每条记录至少被处理过一次（一次或多次）。这种保证能确保没有数据丢失，比“至多一次”要强。但有可能出现数据重复。
3. 精确一次（Exactly once）: 每条记录都精确地只被处理一次 – 也就是说，既没有数据丢失，也不会出现数据重复。这是三种保证中最强的一种。

基础语义
===================

任何流式处理系统一般都会包含以下三个数据处理步骤：

1. 数据接收（Receiving the data）: 从数据源拉取数据。
2. 数据转换（Transforming the data）: 将接收到的数据进行转换（使用DStream和RDD transformation算子）。
3. 数据推送（Pushing out the data）: 将转换后最终数据推送到外部文件系统，数据库或其他展示系统。

如果Streaming应用需要做到端到端的“精确一次”的保证，那么就必须在以上三个步骤中各自都保证精确一次：即，每条记录必须，只接收一次、处理一次、推送一次。下面让我们在Spark Streaming的上下文环境中来理解一下这三个步骤的语义：

1. 数据接收: 不同数据源提供的保证不同，下一节再详细讨论。
2. 数据转换: 所有的数据都会被“精确一次”处理，这要归功于RDD提供的保障。即使出现故障，只要数据源还能访问，最终所转换得到的RDD总是包含相同的内容。
3. 数据推送: 输出操作默认保证“至少一次”的语义，是否能“精确一次”还要看所使用的输出算子（是否幂等）以及下游系统（是否支持事务）。不过用户也可以开发自己的事务机制来实现“精确一次”语义。这个后续会有详细讨论。

接收数据语义
===================

不同的输入源提供不同的数据可靠性级别，从“至少一次”到“精确一次”。

从文件接收数据
-----------------------

如果所有的输入数据都来源于可容错的文件系统，如HDFS，那么Spark Streaming就能在任何故障中恢复并处理所有的数据。这种情况下就能保证精确一次语义，也就是说不管出现什么故障，所有的数据总是精确地只处理一次，不多也不少。

基于接收器接收数据
-----------------------

对于基于接收器的输入源，容错语义将同时依赖于故障场景和接收器类型。前面也已经提到过，spark Streaming主要有两种类型的接收器：

1. 可靠接收器 – 这类接收器会在数据接收并保存好副本后，向可靠数据源发送确认信息。这类接收器故障时，是不会给缓存的（已接收但尚未保存副本）数据发送确认信息。因此，一旦接收器重启，没有收到确认的数据，会重新从数据源再获取一遍，所以即使有故障也不会丢数据。
2. 不可靠接收器 – 这类接收器不会发送确认信息，因此一旦worker和driver出现故障，就有可能会丢失数据。

对于不同的接收器，我们可以获得如下不同的语义。如果一个worker节点故障了，对于可靠接收器来书，不会有数据丢失。而对于不可靠接收器，缓存的（接收但尚未保存副本）数据可能会丢失。如果driver节点故障了，除了接收到的数据之外，其他的已经接收且已经保存了内存副本的数据都会丢失，这将会影响有状态算子的计算结果。

为了避免丢失已经收到且保存副本的数，从 spark 1.2 开始引入了WAL（write ahead logs），以便将这些数据写入到可容错的存储中。只要你使用可靠接收器，同时启用WAL（write ahead logs enabled），那么久再也不用为数据丢失而担心了。并且这时候，还能提供“至少一次”的语义保证。

下表总结了故障情况下的各种语义：
====================================================          ==================================================        ======================
部署场景                                                        Worker 故障                                               Driver 故障
====================================================          ==================================================        ======================
Spark 1.1及以前版本或者Spark 1.2及以后版本，且未开启WAL             若使用不可靠接收器，则可能丢失缓存（已接收但尚未保存副本）数据；      若使用不可靠接收器，则缓存数据和已保存数据都可能丢失；
                                                              若使用可靠接收器，则没有数据丢失，且提供至少一次处理语义            若使用可靠接收器，则没有缓存数据丢失，但已保存数据可能丢失，且不提供语义保证
Spark 1.2及以后版本，并启用WAL                                   若使用可靠接收器，则没有数据丢失，且提供至少一次语义保证             若使用可靠接收器和文件，则无数据丢失，且提供至少一次语义保证
====================================================          ==================================================        ======================

从Kafka Direct API接收数据
---------------------------------

从Spark 1.3开始，我们引入 Kafka Direct API，该API能为Kafka数据源提供“精确一次”语义保证。有了这个输入API，再加上输出算子的“精确一次”保证，你就能真正实现端到端的“精确一次”语义保证。（改功能截止Spark 1.6.1还是实验性的）更详细的说明见：Kafka Integration Guide。

输出算子的语义
---------------------------


输出算子（如 foreachRDD）提供“至少一次”语义保证，也就是说，如果worker故障，单条输出数据可能会被多次写入外部实体中。不过这对于文件系统来说是可以接受的（使用saveAs***Files 多次保存文件会覆盖之前的），所以我们需要一些额外的工作来实现“精确一次”语义。主要有两种实现方式：
* 幂等更新（Idempotent updates）: 就是说多次操作，产生的结果相同。例如，多次调用saveAs***Files保存的文件总是包含相同的数据。
* 事务更新（Transactional updates）: 所有的更新都是事务性的，这样一来就能保证更新的原子性。以下是一种实现方式：
    * 用批次时间（在foreachRDD中可用）和分区索引创建一个唯一标识，该标识代表流式应用中唯一的一个数据块。
    * 基于这个标识建立更新事务，并使用数据块数据更新外部系统。也就是说，如果该标识未被提交，则原子地将标识代表的数据更新到外部系统。否则，就认为该标识已经被提交，直接忽略之。

.. code-block:: Scala

  dstream.foreachRDD { (rdd, time) =>
    rdd.foreachPartition { partitionIterator =>
      val partitionId = TaskContext.get.partitionId()
      val uniqueId = generateUniqueId(time.milliseconds, partitionId)
      // 使用uniqueId作为事务的唯一标识，基于uniqueId实现partitionIterator所指向数据的原子事务提交
    }
  }


*****************************
下一步
*****************************

* 其他相关参考文档
    * Kafka Integration Guide
    * Flume Integration Guide
    * Kinesis Integration Guide
    * Custom Receiver Guide
* Third-party DStream data sources can be found in Third Party Projects
* API 文档
    * Scala 文档
        * StreamingContext 和 DStream
        * KafkaUtils, FlumeUtils, KinesisUtils, TwitterUtils, ZeroMQUtils, 以及 MQTTUtils
    * Java 文档
        * JavaStreamingContext, JavaDStream 以及 JavaPairDStream
        * KafkaUtils, FlumeUtils, KinesisUtils TwitterUtils, ZeroMQUtils, 以及 MQTTUtils
    * Python 文档
        * StreamingContext 和 DStream
        * KafkaUtils
* 其他示例：Scala ，Java 以及 Python
* Spark Streaming 相关的 Paper 和 video。
