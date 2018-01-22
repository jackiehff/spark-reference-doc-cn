#############
快速入门
#############

本教程是对使用 Spark 的一个简单介绍。首先我们会通过 Spark 的交互式 shell 简单介绍一下 (Python 或 Scala) API，然后展示如何使用 Java、Scala 以及 Python 编写一个 Spark 应用程序。

为了方便参照该指南进行学习，请先到 `Spark 网站 <http://spark.apache.org/downloads.html>`_ 下载一个 Spark 发布包。由于我们暂时还不会用到 HDFS，所以你可以下载对应任意 Hadoop 版本的 Spark 发布包。

.. attention:: Spark 2.0 版本之前, Spark 的核心编程接口是弹性分布式数据集(RDD)。Spark 2.0 版本之后, RDD 被 Dataset 所取代, Dataset 跟 RDD 一样也是强类型的, 但是底层做了更多的优化。Spark 目前仍然支持 RDD 接口, 你可以在 :ref:`rdd_programming_guide` 页面获得更完整的参考，但是我们强烈建议你转而使用比 RDD 有着更好性能的 Dataset。想了解关于 Dataset 的更多信息请参考 :ref:`sql_programming_guide`。


********************************
使用 Spark Shell 进行交互式分析
********************************

基础知识
===================

Spark Shell 提供了一种简单的方式来学习 Spark API，同时它也是一个强大的交互式数据分析工具。Spark Shell 既支持 Scala(Scala 运行在 Java 虚拟机上，所以可以很方便的引用现有的 Java 库)也支持 Python。

**Scala**

在 Spark 目录下运行以下命令可以启动 Spark Shell：

.. code-block:: Shell

  ./bin/spark-shell


Spark 最主要的抽象概念就是一个叫做 Dataset 的分布式数据集。Dataset 可以从 Hadoop InputFormats(例如 HDFS 文件)创建或者由其他 Dataset 转换而来。下面我们利用 Spark 源码目录下 README 文件中的文本来新建一个 Dataset：

.. code-block:: Shell

  scala> val textFile = spark.read.textFile("README.md")
  textFile: org.apache.spark.sql.Dataset[String] = [value: string]

你可以调用 action 算子直接从 Dataset 获取值，或者转换该 Dataset 以获取一个新的 Dataset。更多细节请参阅 `API 文档 <http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset>`_ 。

.. code-block:: Shell

  scala> textFile.count() // Number of items in this Dataset
  res0: Long = 126 // May be different from yours as README.md will change over time, similar to other outputs

  scala> textFile.first() // First item in this Dataset
  res1: String = # Apache Spark

现在我们将该 Dataset 转换成一个新的 Dataset。我们调用 filter 这个 transformation 算子返回一个只包含原始文件数据项子集的新 Dataset。

.. code-block:: Shell

  scala> val linesWithSpark = textFile.filter(line => line.contains("Spark"))
  linesWithSpark: spark.RDD[String] = spark.FilteredRDD@7dd4af09

我们可以将 transformation 算子和 action 算子连在一起:

.. code-block:: Shell

  scala> textFile.filter(line => line.contains("Spark")).count() // How many lines contain "Spark"?
  res3: Long = 15


**Python**

在 Spark 目录下运行以下命令可以启动 Spark Shell：

.. code-block:: Shell

  ./bin/pyspark

或者如果在你当前环境已经使用 pip 安装了 PySpark，你也可以直接使用以下命令:

.. code-block:: Shell

  pyspark

Spark 最主要的抽象概念就是一个叫做 Dataset 的分布式数据集。Dataset 可以从 Hadoop InputFormats(例如 HDFS 文件)创建或者由其他 Dataset 转换而来。由于 Python 语言的动态性, 我们不需要 Dataset 是强类型的。因此 Python 中所有的 Dataset 都是 Dataset[Row], 并且为了和 Pandas 以及 R 中的 data frame 概念保持一致, 我们称其为 DataFrame。下面我们利用 Spark 源码目录下 README 文件中的文本来新建一个 DataFrame:

.. code-block:: Shell

  >>> textFile = spark.read.text("README.md")

你可以调用 action 算子直接从 DataFrame 获取值，或者转换该 DataFrame 以获取一个新的 DataFrame。更多细节请参阅 `API 文档 <http://spark.apache.org/docs/latest/api/python/index.html#pyspark.sql.DataFrame>`_ 。

.. code-block:: Shell

  >>> textFile.count()  # Number of rows in this DataFrame
  126

  >>> textFile.first()  # First row in this DataFrame
  Row(value=u'# Apache Spark')

现在我们将该 DataFrame 转换成一个新的 DataFrame。我们调用 filter 这个 transformation 算子返回一个只包含原始文件数据项子集的新 DataFrame。

.. code-block:: Shell

  >>> linesWithSpark = textFile.filter(textFile.value.contains("Spark"))

我们可以将 transformation 算子和 action 算子连在一起:

.. code-block:: Shell

  >>> textFile.filter(textFile.value.contains("Spark")).count()  # How many lines contain "Spark"?
  15


更多 Dataset 算子
===================

Dataset action 和 transformation 算子可以用于更加复杂的计算。比方说我们想要找到文件中包含单词数最多的行。

**Scala**

.. code-block:: Shell

  scala> textFile.map(line => line.split(" ").size).reduce((a, b) => if (a > b) a else b)
  res4: Long = 15

首先，使用 map 算子将每一行映射为一个整数值，创建了一个新的 Dataset。然后在该 Dataset 上调用 reduce 算子找出最大的单词计数。map 和 reduce 算子的参数都是 cala 函数字面量(闭包)，并且可以使用任意语言特性或 Scala/Java 库。例如，我们可以很容易地调用其他地方声明的函数。为了使代码更容易理解，下面我们使用Math.max():

.. code-block:: Shell

  scala> import java.lang.Math
  import java.lang.Math

  scala> textFile.map(line => line.split(" ").size).reduce((a, b) => Math.max(a, b))
  res5: Int = 15

因 Hadoop 而广为流行的 MapReduce 是一种通用的数据流模式。Spark 可以很容易地实现 MapReduce 流程：

.. code-block:: Shell

  scala> val wordCounts = textFile.flatMap(line => line.split(" ")).groupByKey(identity).count()
  wordCounts: org.apache.spark.sql.Dataset[(String, Long)] = [value: string, count(1): bigint]

这里我们调用 flatMap 这个 transformation 算子将一个行的 Dataset 转换成了一个单词的 Dataset, 然后组合 groupByKey 和 count 算子来计算文件中每个单词出现的次数，生成一个包含(String, Long)键值对的 Dataset。为了在 shell 中收集到单词计数, 我们可以调用 collect 算子:

.. code-block:: Shell

  scala> wordCounts.collect()
  res6: Array[(String, Int)] = Array((means,1), (under,2), (this,3), (Because,1), (Python,2), (agree,1), (cluster.,1), ...)


**Python**

.. code-block:: Shell

  >>> from pyspark.sql.functions import *
  >>> textFile.select(size(split(textFile.value, "\s+")).name("numWords")).agg(max(col("numWords"))).collect()
  [Row(max(numWords)=15)]

首先，使用 map 算子将每一行映射为一个整数值并给其取别名 “numWords”, 创建了一个新的 DataFrame。然后在该 DataFrame 上调用 agg 算子找出最大的单词计数。select 和 agg 的参数都是 `Column <http://spark.apache.org/docs/latest/api/python/index.html#pyspark.sql.Column>`_ , 我们可以使用 df.colName 从 DataFrame 上获取一列，也可以引入 pyspark.sql.functions, 它提供了很多方便的函数用来从旧的 Column 构建新的 Column。

因 Hadoop 而广为流行的 MapReduce 是一种通用的数据流模式。Spark 可以很容易地实现 MapReduce 流程：

.. code-block:: Shell

  >>> wordCounts = textFile.select(explode(split(textFile.value, "\s+")).alias("word")).groupBy("word").count()

这里我们在 select 函数中使用 explode 函数将一个行的 Dataset 转换成了一个单词的 Dataset, 然后组合 groupBy 和 count 算子来计算文件中每个单词出现的次数，生成一个包含 “word” 和 “count” 这 2 列的 DataFrame。为了在 shell 中收集到单词计数, 我们可以调用 collect 算子:

.. code-block:: Shell

  >>> wordCounts.collect()
  [Row(word=u'online', count=1), Row(word=u'graphs', count=1), ...]

缓存
===================

Spark 还支持把数据集拉到集群范围的内存缓存中。当数据需要反复访问时非常有用，比如查询一个小的热门数据集或者运行一个像 PageRank 这样的迭代算法。作为一个简单的示例，我们把 linesWithSpark 这个数据集缓存起来。

**Scala**

.. code-block:: Shell

  scala> linesWithSpark.cache()
  res7: linesWithSpark.type = [value: string]

  scala> linesWithSpark.count()
  res8: Long = 15

  scala> linesWithSpark.count()
  res9: Long = 15

用 Spark 浏览和缓存一个 100 行左右的文本文件看起来确实有点傻。但有趣的部分是这些相同的函数可以用于非常大的数据集，即使这些数据集分布在数十或数百个节点上。如 :ref:`rdd_programming_guide` 中描述的那样, 你也可以通过 bin/spark-shell 连接到一个集群，交互式地执行上面那些操作。

**Python**

.. code-block:: Shell

  >>> linesWithSpark.cache()

  >>> linesWithSpark.count()
  15

  >>> linesWithSpark.count()
  15

用 Spark 浏览和缓存一个 100 行左右的文本文件看起来确实有点傻。但有趣的部分是这些相同的函数可以用于非常大的数据集，即使这些数据集分布在数十或数百个节点上。如 :ref:`rdd_programming_guide` 中描述的那样, 你也可以通过 bin/pyspark 连接到一个集群，交互式地执行上面那些操作。


********************************
自包含的(self-contained)应用程序
********************************

假设我们想使用 Spark API 编写一个自包含(self-contained)的 Spark 应用程序。下面我们将快速过一下一个简单的应用程序，分别使用 Scala(sbt编译)，Java(maven编译)和 Python(pip) 编写。

**Scala**

首先创建一个非常简单的 Spark 应用程序 – 简单到连名字都叫 SimpleApp.scala:

.. code-block:: Scala

  /* SimpleApp.scala */
  import org.apache.spark.sql.SparkSession

  object SimpleApp {
    def main(args: Array[String]) {
      val logFile = "YOUR_SPARK_HOME/README.md" // Should be some file on your system
      val spark = SparkSession.builder.appName("Simple Application").getOrCreate()
      val logData = spark.read.textFile(logFile).cache()
      val numAs = logData.filter(line => line.contains("a")).count()
      val numBs = logData.filter(line => line.contains("b")).count()
      println(s"Lines with a: $numAs, Lines with b: $numBs")
      spark.stop()
    }
  }

.. attention:: 应用程序需要定义一个 main 方法，而不是继承 scala.App。scala.App 的子类可能不能正常工作。

这个程序只是统计 Spark README 文件中包含‘a’和包含’b’的行数。注意，你需要把 YOUR_SPARK_HOME 替换成 Spark 的安装目录。与之前使用 Spark Shell 的示例不同，Spark Shell 会初始化自己的 SparkSession 对象, 而我们需要初始化 SparkSession 对象作为程序的一部分。

我们调用 SparkSession.builder 来构造一个 [[SparkSession]] 对象, 然后设置应用程序名称, 最后调用 getOrCreate 方法获取 [[SparkSession]] 实例。

我们的应用程序依赖于 Spark API，所以我们需要包含一个 sbt 配置文件，build.sbt，用于配置 Spark 依赖项。这个文件同时也添加了 Spark 本身的依赖库：

.. code-block:: text

  name := "Simple Project"
  version := "1.0"
  scalaVersion := "2.11.8"
  libraryDependencies += "org.apache.spark" %% "spark-sql" % "2.2.1"

为了让 sbt 能够正常工作，我们需要根据一个标准规范的 Scala 项目目录结构来放置 SimpleApp.scala 和 build.sbt 文件。一切准备就绪后，我们就可以创建一个包含应用程序代码的 JAR 包，然后使用 spark-submit 脚本运行我们的程序。

.. code-block:: Shell

  # Your directory layout should look like this
  $ find .
  .
  ./simple.sbt
  ./src
  ./src/main
  ./src/main/scala
  ./src/main/scala/SimpleApp.scala

  # Package a jar containing your application
  $ sbt package
  ...
  [info] Packaging {..}/{..}/target/scala-2.11/simple-project_2.11-1.0.jar

  # Use spark-submit to run your application
  $ YOUR_SPARK_HOME/bin/spark-submit \
    --class "SimpleApp" \
    --master local[4] \
    target/scala-2.11/simple-project_2.11-1.0.jar
  ...
  Lines with a: 46, Lines with b: 23


**Java**

下面这个示例程序将使用 Maven 来编译一个应用程序 JAR, 但是适用任何类似的构建系统。

我们创建一个非常简单的 Spark 应用程序, SimpleApp.java:

.. code-block:: Java

  /* SimpleApp.java */
  import org.apache.spark.sql.SparkSession;
  import org.apache.spark.sql.Dataset;

  public class SimpleApp {
    public static void main(String[] args) {
      String logFile = "YOUR_SPARK_HOME/README.md"; // Should be some file on your system
      SparkSession spark = SparkSession.builder().appName("Simple Application").getOrCreate();
      Dataset<String> logData = spark.read().textFile(logFile).cache();

      long numAs = logData.filter(s -> s.contains("a")).count();
      long numBs = logData.filter(s -> s.contains("b")).count();

      System.out.println("Lines with a: " + numAs + ", lines with b: " + numBs);

      spark.stop();
    }
  }

这个程序只是统计 Spark README 文件中包含‘a’和包含’b’的行数。注意，你需要把 YOUR_SPARK_HOME 替换成 Spark 的安装目录。与之前使用 Spark Shell 的示例不同，Spark Shell 会初始化自己的 SparkSession 对象, 而我们需要初始化 SparkSession 对象作为程序的一部分。

为了构建程序, 我们还需要编写一个 Maven pom.xml 文件将 Spark 列为依赖项。注意，Spark 构件都附加了 Scala 版本号。

.. code-block:: XML

  <project>
    <groupId>edu.berkeley</groupId>
    <artifactId>simple-project</artifactId>
    <modelVersion>4.0.0</modelVersion>
    <name>Simple Project</name>
    <packaging>jar</packaging>
    <version>1.0</version>
    <dependencies>
      <dependency> <!-- Spark dependency -->
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql_2.11</artifactId>
        <version>2.2.1</version>
      </dependency>
    </dependencies>
  </project>

接着，我们根据标准规范的 Maven 项目目录结构放置这些文件:

.. code-block:: Shell

  $ find .
  ./pom.xml
  ./src
  ./src/main
  ./src/main/java
  ./src/main/java/SimpleApp.java

现在我们可以使用 Maven 打包应用程序并使用 ./bin/spark-submit 命令执行它。

.. code-block:: Shell

  # Package a JAR containing your application
  $ mvn package
  ...
  [INFO] Building jar: {..}/{..}/target/simple-project-1.0.jar

  # Use spark-submit to run your application
  $ YOUR_SPARK_HOME/bin/spark-submit \
    --class "SimpleApp" \
    --master local[4] \
    target/simple-project-1.0.jar
  ...
  Lines with a: 46, Lines with b: 23


**Python**

现在我们将展示如何使用 Python API (PySpark) 来编写一个 Spark 应用程序。

如果你在构建一个打包好的 PySpark 应用程序或者库, 你可以像下面这样将其添加到 setup.py 文件中:

.. code-block:: Python

    install_requires=[
        'pyspark=={site.SPARK_VERSION}'
    ]


我们将创建一个简单的 Spark 应用程序 SimpleApp.py 作为示例程序:

.. code-block:: Python

  """SimpleApp.py"""
  from pyspark.sql import SparkSession

  logFile = "YOUR_SPARK_HOME/README.md"  # Should be some file on your system
  spark = SparkSession.builder().appName(appName).master(master).getOrCreate()
  logData = spark.read.text(logFile).cache()

  numAs = logData.filter(logData.value.contains('a')).count()
  numBs = logData.filter(logData.value.contains('b')).count()

  print("Lines with a: %i, lines with b: %i" % (numAs, numBs))

  spark.stop()


这个程序只是统计 Spark README 文件中包含‘a’和包含’b’的行数。注意，你需要把 YOUR_SPARK_HOME 替换成 Spark 的安装目录。在 Scala 和 Java 编写的示例程序中, 我们使用 SparkSession 来创建 Dataset。对于使用自定义类或第三方库的应用程序，我们还可以将代码依赖打包成 .zip 文件, 然后通过 spark-submit 脚本提供的 --py-files 参数添加到 spark-submit (更多细节参见 spark-submit --help)。SimpleApp 已经足够简单，我们不需要指定任何代码依赖。

我们可以使用 bin/spark-submit 脚本运行这个应用程序:

.. code-block:: Shell

  # Use spark-submit to run your application
  $ YOUR_SPARK_HOME/bin/spark-submit \
    --master local[4] \
    SimpleApp.py
  ...
  Lines with a: 46, Lines with b: 23

如果你已经使用 pip 安装了 PySpark (例如 pip install pyspark), 你可以使用普通的 Python 解释器运行应用程序，或着根据你自己的喜好使用 Spark 提供的 spark-submit 脚本。

.. code-block:: Shell

  # Use python to run your application
  $ python SimpleApp.py
  ...
  Lines with a: 46, Lines with b: 23


********************************
下一步
********************************

恭喜您成功运行您的第一个 Spark 应用程序！

* 如果想深入了解 Spark API, 可以从 :ref:`rdd_programming_guide` 和 :ref:`sql_programming_guide`，或者在 "Programming Guides" 菜单下查找其它组件。
* 如果想了解如何在集群上运行 Spark 应用程序，请前往：:ref:`cluster_overview`。
* 最后，Spark examples 目录下包含了多个编程语言(Scala, Java, Python, R)版本的示例程序，你可以像下面这样运行它们：

.. code-block:: Shell

  # For Scala and Java, use run-example:
  ./bin/run-example SparkPi

  # For Python examples, use spark-submit directly:
  ./bin/spark-submit examples/src/main/python/pi.py

  # For R examples, use spark-submit directly:
  ./bin/spark-submit examples/src/main/r/dataframe.R
