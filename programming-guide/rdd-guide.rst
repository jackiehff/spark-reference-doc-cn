.. _rdd_programming_guide:

###############
RDD 编程指南
###############


***************
概述
***************

总体上来说，每个 Spark 应用程序都包含一个驱动器（driver）程序，驱动器程序运行用户的 main 函数，并在集群上执行各种并行操作。Spark 最重要的一个抽象概念就是弹性分布式数据集（resilient distributed dataset – RDD）, RDD是一个可分区的元素集合，这些元素分布在集群的各个节点上，并且可以在这些元素上执行并行操作。RDD通常是通过HDFS（或者Hadoop支持的其它文件系统）上的文件，或者驱动器中的Scala集合对象来创建或转换得到；其次，用户也可以请求Spark将RDD持久化到内存里，以便在不同的并行操作里复用之；最后，RDD具备容错性，可以从节点失败中自动恢复。

Spark 第二个重要抽象概念是共享变量，共享变量是一种可以在并行操作之间共享使用的变量。默认情况下，当Spark把一系列任务调度到不同节点上运行时，Spark会同时把每个变量的副本和任务代码一起发送给各个节点。但有时候，我们需要在任务之间，或者任务和驱动器之间共享一些变量。Spark 支持两种类型的共享变量：广播变量 和 累加器，广播变量可以用于在各个节点上缓存数据，而累加器则是用来执行跨节点的 "累加" 操作，例如：计数和求和。

本文将会使用 Spark 所支持的所有语言来展示 Spark 的这些特性。如果你能启动 Spark 的交互式shell动手实验一下，效果会更好（对于 Scala shell请使用bin/spark-shell，而对于python，请使用bin/pyspark）。


***************
链接 Spark
***************

**Scala**

Spark 2.2.1 默认使用 Scala 2.11 版本进行构建和分发的。(Spark 也可以使用其它版本的 Scala 进行构建)如果想用 Scala 写应用程序，你需要使用兼容的 Scala 版本(如：2.11.X)

要编写 Spark 应用程序，你需要添加 Spark 的 Maven 依赖。Spark 依赖可以通过以下 Maven 坐标从 Maven 中央仓库中获得：

.. code-block:: TEXT

  groupId = org.apache.spark
  artifactId = spark-core_2.11
  version = 2.2.1

另外，如果你想要访问 HDFS 集群，那么需要添加对应 HDFS 版本的 hadoop-client 依赖。

.. code-block:: TEXT

  groupId = org.apache.hadoop
  artifactId = hadoop-client
  version = <your-hdfs-version>

最后，你需要在程序中添加下面几行来引入一些 Spark 类：

.. code-block:: Scala

  import org.apache.spark.SparkContext
  import org.apache.spark.SparkConf

(在 Spark 1.3.0 版本之前，你需要显示地 import org.apache.spark.SparkContext._ 来启用必要的隐式转换)

**Java**

Spark 2.2.1 对 `Lambda 表达式 <https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html>`_ 的支持可以让我们很简洁地编写函数, 否则的话你可以使用 `org.apache.spark.api.java.function <http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/function/package-summary.html>`_ 包中的类.

.. attention:: Spark 2.2.0 版本中已经移除对 Java 7 的支持。

要使用 Java 来编写 Spark 应用程序，你需要添加 Spark 的 Maven 依赖。Spark 依赖可以通过以下 Maven 坐标从 Maven 中央仓库中获得:

.. code-block:: TEXT

  groupId = org.apache.spark
  artifactId = spark-core_2.11
  version = 2.2.1

另外，如果你想要访问 HDFS 集群，那么需要添加对应 HDFS 版本的 hadoop-client 依赖。

.. code-block:: TEXT

  groupId = org.apache.hadoop
  artifactId = hadoop-client
  version = <your-hdfs-version>

最后，你需要在程序中添加下面几行来引入一些 Spark 类：

.. code-block:: Java

  import org.apache.spark.api.java.JavaSparkContext
  import org.apache.spark.api.java.JavaRDD
  import org.apache.spark.SparkConf


**Python**

Spark 2.2.1 适用于 Python 2.7 及以上版本 或 Python 3.4 及以上版本。它可以使用标准的 CPython 解释器, 因此我们可以使用像 NumPy 这样的 C 语言库。它也适用 PyPy 2.3 及以上版本。

Spark 2.2.0 版本中移除了对 Python 2.6 的支持。

使用 Python 编写的 Spark 应用程序既可以使用在运行时包含 Spark 的 bin/spark-submit 脚本运行, 也可以像下面这样通过在 setup.py 文件中包含它:

.. code-block:: Python

    install_requires=[
        'pyspark=={site.SPARK_VERSION}'
    ]

To run Spark applications in Python without pip installing PySpark, use the bin/spark-submit script located in the Spark directory. This script will load Spark’s Java/Scala libraries and allow you to submit applications to a cluster. You can also use bin/pyspark to launch an interactive Python shell.

如果你想要访问 HDFS 数据, you need to use a build of PySpark linking to your version of HDFS. Prebuilt packages are also available on the Spark homepage for common HDFS versions.

最后, 你需要添加下面这行来在程序中引入一些 Spark 类:

.. code-block:: Python

  from pyspark import SparkContext, SparkConf

PySpark requires the same minor version of Python in both driver and workers. 它使用 PATH 中默认的 Python 版本, 你也可以通过 PYSPARK_PYTHON 指定你想要使用的 Python 版本, 例如:

.. code-block:: Shell

  $ PYSPARK_PYTHON=python3.4 bin/pyspark
  $ PYSPARK_PYTHON=/opt/pypy-2.5/bin/pypy bin/spark-submit examples/src/main/python/pi.py


***************
初始化 Spark
***************

**Scala**

Spark 程序需要做的第一件事就是创建一个 SparkContext 对象，SparkContext 对象决定了 Spark 如何访问集群。而要新建一个 SparkContext 对象，你还得需要构造一个 SparkConf 对象，SparkConf对象包含了你的应用程序的配置信息。

每个JVM进程中，只能有一个活跃（active）的 SparkContext 对象。如果你非要再新建一个，那首先必须将之前那个活跃的 SparkContext 对象stop()掉。

.. code-block:: Scala

  val conf = new SparkConf().setAppName(appName).setMaster(master)
  new SparkContext(conf)

**Java**

Spark 程序需要做的第一件事就是创建一个 JavaSparkContext 对象, which tells Spark how to access a cluster. To create a SparkContext you first need to build a SparkConf object that contains information about your application.

.. code-block:: Java

  SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
  JavaSparkContext sc = new JavaSparkContext(conf);

**Python**

Spark 程序需要做的第一件事就是创建一个 SparkContext 对象, which tells Spark how to access a cluster. To create a SparkContext you first need to build a SparkConf object that contains information about your application.

.. code-block:: Python

  conf = SparkConf().setAppName(appName).setMaster(master)
  sc = SparkContext(conf=conf)


appName 参数值是你的应用展示在集群UI上的应用名称。master参数值是Spark, Mesos or YARN cluster URL 或者特殊的“local”（本地模式）。实际上，一般不应该将master参数值硬编码到代码中，而是应该用spark-submit脚本的参数来设置。然而，如果是本地测试或单元测试中，你可以直接在代码里给master参数写死一个”local”值。


使用 Shell
====================

**Scala**

在 Spark Shell 中，默认已经为你新建了一个 SparkContext 对象，变量名为sc。所以 spark-shell 里不能自建SparkContext对象。你可以通过–master参数设置要连接到哪个集群，而且可以给–jars参数传一个逗号分隔的jar包列表，以便将这些jar包加到classpath中。你还可以通过–packages设置逗号分隔的maven工件列表，以便增加额外的依赖项。同样，还可以通过–repositories参数增加maven repository地址。下面是一个示例，在本地4个CPU core上运行的实例：

.. code-block:: Shell

  $ ./bin/spark-shell –master local[4]

或者，将 code.jar 添加到 classpath 下：

.. code-block:: Shell

  $ ./bin/spark-shell --master local[4] --jars code.jar

通过 maven标识添加依赖：

.. code-block:: Shell

  $ ./bin/spark-shell --master local[4] --packages "org.example:example:0.1"

spark-shell –help 可以查看完整的选项列表。实际上，spark-shell 是在后台调用 spark-submit 来实现其功能的（spark-submit script.）


**Python**

In the PySpark shell, a special interpreter-aware SparkContext is already created for you, in the variable called sc. Making your own SparkContext will not work. You can set which master the context connects to using the --master argument, and you can add Python .zip, .egg or .py files to the runtime path by passing a comma-separated list to --py-files. You can also add dependencies (e.g. Spark Packages) to your shell session by supplying a comma-separated list of Maven coordinates to the --packages argument. Any additional repositories where dependencies might exist (e.g. Sonatype) can be passed to the --repositories argument. Any Python dependencies a Spark package has (listed in the requirements.txt of that package) must be manually installed using pip when necessary. For example, to run bin/pyspark on exactly four cores, use:

.. code-block:: Shell

  $ ./bin/pyspark --master local[4]

Or, to also add code.py to the search path (in order to later be able to import code), use:

.. code-block:: Shell

  $ ./bin/pyspark --master local[4] --py-files code.py

For a complete list of options, run pyspark --help. Behind the scenes, pyspark invokes the more general spark-submit script.

It is also possible to launch the PySpark shell in IPython, the enhanced Python interpreter. PySpark works with IPython 1.0.0 and later. To use IPython, set the PYSPARK_DRIVER_PYTHON variable to ipython when running bin/pyspark:

.. code-block:: Shell

  $ PYSPARK_DRIVER_PYTHON=ipython ./bin/pyspark

To use the Jupyter notebook (previously known as the IPython notebook),

.. code-block:: Shell

  $ PYSPARK_DRIVER_PYTHON=jupyter PYSPARK_DRIVER_PYTHON_OPTS=notebook ./bin/pyspark

You can customize the ipython or jupyter commands by setting PYSPARK_DRIVER_PYTHON_OPTS.

After the Jupyter Notebook server is launched, you can create a new “Python 2” notebook from the “Files” tab. Inside the notebook, you can input the command %pylab inline as part of your notebook before you start to try Spark from the Jupyter notebook.


***********************
弹性分布式数据集(RDD)
***********************

Spark的核心概念是弹性分布式数据集(RDD)，RDD是一个可容错、可并行操作的分布式元素集合。总体上有两种方法可以创建 RDD 对象：由驱动程序中的集合对象通过并行化操作创建，或者从外部存储系统中数据集加载（如：共享文件系统、HDFS、HBase或者其他Hadoop支持的数据源）。


并行集合
=======================

**Scala**

并行集合是以一个已有的集合对象（例如：Scala Seq）为参数，调用 SparkContext.parallelize() 方法创建得到的 RDD。集合对象中所有的元素都将被复制到一个可并行操作的分布式数据集中。例如，以下代码将一个1到5组成的数组并行化成一个RDD：

.. code-block:: Scala

  val data = Array(1, 2, 3, 4, 5)
  val distData = sc.parallelize(data)

一旦创建成功，该分布式数据集（上例中的distData）就可以执行一些并行操作。如，distData.reduce((a, b) => a + b)，这段代码会将集合中所有元素加和。后面我们还会继续讨论分布式数据集上的各种操作。

**Java**

Parallelized collections are created by calling JavaSparkContext’s parallelize method on an existing Collection in your driver program. The elements of the collection are copied to form a distributed dataset that can be operated on in parallel. For example, here is how to create a parallelized collection holding the numbers 1 to 5:

.. code-block:: Java

  List<Integer> data = Arrays.asList(1, 2, 3, 4, 5);
  JavaRDD<Integer> distData = sc.parallelize(data);

Once created, the distributed dataset (distData) can be operated on in parallel. For example, we might call distData.reduce((a, b) -> a + b) to add up the elements of the list. We describe operations on distributed datasets later on.


**Python**

Parallelized collections are created by calling SparkContext’s parallelize method on an existing iterable or collection in your driver program. The elements of the collection are copied to form a distributed dataset that can be operated on in parallel. For example, here is how to create a parallelized collection holding the numbers 1 to 5:

.. code-block:: Python

  data = [1, 2, 3, 4, 5]
  distData = sc.parallelize(data)

Once created, the distributed dataset (distData) can be operated on in parallel. For example, we can call distData.reduce(lambda a, b: a + b) to add up the elements of the list. We describe operations on distributed datasets later on.

并行集合的一个重要参数是分区（partition），即这个分布式数据集可以分割为多少片。Spark中每个任务（task）都是基于分区的，每个分区一个对应的任务（task）。典型场景下，一般每个CPU对应2~4个分区。并且一般而言，Spark会基于集群的情况，自动设置这个分区数。当然，你还是可以手动控制这个分区数，只需给parallelize方法再传一个参数即可（如：sc.parallelize(data, 10) ）。注意：Spark代码里有些地方仍然使用分片（slice）这个术语，这只不过是分区的一个别名，主要为了保持向后兼容。


外部数据集
=======================


Spark 可以通过Hadoop所支持的任何数据源来创建分布式数据集，包括：本地文件系统、HDFS、Cassandra、HBase、Amazon S3 等。Spark 支持的文件格式包括：文本文件（text files）、SequenceFiles，以及其他 Hadoop 支持的输入格式（InputFormat）。

文本文件创建RDD可以用 SparkContext.textFile 方法。这个方法输入参数是一个文件的URI（本地路径，或者 hdfs://，s3n:// 等），其输出RDD是一个文本行集合。以下是一个简单示例：

scala> val distFile = sc.textFile("data.txt")
distFile: RDD[String] = MappedRDD@1d4cee08

创建后，distFile 就可以执行数据集的一些操作。比如，我们可以把所有文本行的长度加和：distFile.map(s => s.length).reduce((a, b) => a + b)

以下是一些 Spark 读取文件的要点：

* 如果是本地文件系统，那么这个文件必须在所有的 worker 节点上能够以相同的路径访问到。所以要么把文件复制到所有worker节点上同一路径下，要么挂载一个共享文件系统。
* 所有 Spark 基于文件输入的方法（包括textFile）都支持输入参数为：目录，压缩文件，以及通配符。例如：textFile(“/my/directory”), textFile(“/my/directory/*.txt”), 以及 textFile(“/my/directory/*.gz”)
* textFile 方法同时还支持一个可选参数，用以控制数据的分区个数。默认地，Spark会为文件的每一个block创建一个分区（HDFS上默认block大小为64MB），你可以通过调整这个参数来控制数据的分区数。注意，分区数不能少于block个数。除了文本文件之外，Spark的Scala API还支持其他几种数据格式：
* SparkContext.wholeTextFiles 可以读取一个包含很多小文本文件的目录，并且以 (filename, content) 键值对的形式返回结果。这与textFile 不同，textFile只返回文件的内容，每行作为一个元素。
* 对于 SequenceFiles，可以调用 SparkContext.sequenceFile[K, V]，其中 K 和 V 分别是文件中 key 和 value 的类型。这些类型都应该是 Writable 接口的子类, 如：IntWritable and Text 等。另外，Spark 允许你为一些常用Writable指定原生类型，例如：sequenceFile[Int, String] 将自动读取 IntWritable 和 Text。
* 对于其他的 Hadoop InputFormat，你可以用 SparkContext.hadoopRDD 方法，并传入任意的 JobConf 对象和 InputFormat，以及 key class、value class。这和设置 Hadoop job 的输入源是同样的方法。你还可以使用 SparkContext.newAPIHadoopRDD，该方法接收一个基于新版Hadoop MapReduce API （org.apache.hadoop.mapreduce）的InputFormat作为参数。
* RDD.saveAsObjectFile 和 SparkContext.objectFile 支持将 RDD 中元素以 Java 对象序列化的格式保存成文件。虽然这种序列化方式不如 Avro 效率高，却为保存 RDD 提供了一种简便方式。


RDD 算子
=======================

RDD 支持两种类型的算子：transformation 和 action。transformation算子可以将已有RDD转换得到一个新的RDD，而action算子则是基于RDD的计算，并将结果返回给驱动器（driver）。例如，map 是一个 transformation 算子，它将数据集中每个元素传给一个指定的函数，并将该函数返回结果构建为一个新的RDD；而 reduce 是一个 action 算子，它可以将 RDD 中所有元素传给指定的聚合函数，并将最终的聚合结果返回给驱动器（还有一个 reduceByKey 算子，其返回的聚合结果是一个 RDD）。

Spark 中所有 transformation 算子都是懒惰的，也就是说，transformation 算子并不立即计算结果，而是记录下对基础数据集（如：一个数据文件）的转换操作。只有等到某个 action 算子需要计算一个结果返回给驱动器的时候，transformation 算子所记录的操作才会被计算。这种设计使Spark可以运行得更加高效 – 例如，map算子创建了一个数据集，同时该数据集下一步会调用reduce算子，那么Spark将只会返回reduce的最终聚合结果（单独的一个数据）给驱动器，而不是将map所产生的数据集整个返回给驱动器。

默认情况下，每次调用 action 算子的时候，每个由 transformation 转换得到的RDD都会被重新计算。然而，你也可以通过调用 persist（或者cache）操作来持久化一个 RDD，这意味着 Spark 将会把 RDD 的元素都保存在集群中，因此下一次访问这些元素的速度将大大提高。同时，Spark 还支持将RDD元素持久化到内存或者磁盘上，甚至可以支持跨节点多副本。

基础
------------------


以下简要说明一下RDD的基本操作，参考如下代码：

.. code-block:: Scala

  val lines = sc.textFile("data.txt")
  val lineLengths = lines.map(s => s.length)
  val totalLength = lineLengths.reduce((a, b) => a + b)

其中，第一行是从外部文件加载数据，并创建一个基础RDD。这时候，数据集并没有加载进内存除非有其他操作施加于lines，这时候的lines RDD其实可以说只是一个指向 data.txt 文件的指针。第二行，用lines通过map转换得到一个lineLengths RDD，同样，lineLengths也是懒惰计算的。最后，我们使用 reduce算子计算长度之和，reduce是一个action算子。此时，Spark将会把计算分割为一些小的任务，分别在不同的机器上运行，每台机器上都运行相关的一部分map任务，并在本地进行reduce，并将这些reduce结果都返回给驱动器。

如果我们后续需要重复用到 lineLengths RDD，我们可以增加一行：

lineLengths.persist()

这一行加在调用 reduce 之前，则 lineLengths RDD 首次计算后，Spark会将其数据保存到内存中。

将函数传给Spark
------------------

**Scala**

Spark 的 API 很多都依赖于在驱动程序中向集群传递操作函数。以下是两种建议的实现方式：

* 匿名函数（Anonymous function syntax），这种方式代码量比较少。
* 全局单件中的静态方法。例如，你可以按如下方式定义一个 object MyFunctions 并传递其静态成员函数 MyFunctions.func1：

.. code-block:: Scala

  object MyFunctions {
    def func1(s: String): String = { ... }
  }

  myRdd.map(MyFunctions.func1)


注意，技术上来说，你也可以传递一个类对象实例上的方法（不是单例对象），不过这回导致传递函数的同时，需要把相应的对象也发送到集群中各节点上。例如，我们定义一个MyClass如下：

.. code-block:: Scala

  class MyClass {
    def func1(s: String): String = { ... }
    def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(func1) }
  }

如果我们 new MyClass 创建一个实例，并调用其 doStuff 方法，同时doStuff中的 map算子引用了该MyClass实例上的 func1 方法，那么接下来，这个MyClass对象将被发送到集群中所有节点上。rdd.map(x => this.func1(x)) 也会有类似的效果。

类似地，如果应用外部对象的成员变量，也会导致对整个对象实例的引用：

.. code-block:: Scala

  class MyClass {
    val field = "Hello"
    def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(x => field + x) }
  }


上面的代码对 field 的引用等价于 rdd.map(x => this.field + x)，这将导致应用整个this对象。为了避免类似问题，最简单的方式就是，将field固执到一个本地临时变量中，而不是从外部直接访问之，如下：

.. code-block:: Scala

  def doStuff(rdd: RDD[String]): RDD[String] = {
    val field_ = this.field
    rdd.map(x => field_ + x)
  }


**Java**

Spark’s API relies heavily on passing functions in the driver program to run on the cluster. In Java, functions are represented by classes implementing the interfaces in the org.apache.spark.api.java.function package. There are two ways to create such functions:

Implement the Function interfaces in your own class, either as an anonymous inner class or a named one, and pass an instance of it to Spark.
Use lambda expressions to concisely define an implementation.
While much of this guide uses lambda syntax for conciseness, it is easy to use all the same APIs in long-form. For example, we could have written our code above as follows:

.. code-block:: Java

  JavaRDD<String> lines = sc.textFile("data.txt");
  JavaRDD<Integer> lineLengths = lines.map(new Function<String, Integer>() {
    public Integer call(String s) { return s.length(); }
  });
  int totalLength = lineLengths.reduce(new Function2<Integer, Integer, Integer>() {
    public Integer call(Integer a, Integer b) { return a + b; }
  });

Or, if writing the functions inline is unwieldy:

.. code-block:: Java

  class GetLength implements Function<String, Integer> {
    public Integer call(String s) { return s.length(); }
  }
  class Sum implements Function2<Integer, Integer, Integer> {
    public Integer call(Integer a, Integer b) { return a + b; }
  }

  JavaRDD<String> lines = sc.textFile("data.txt");
  JavaRDD<Integer> lineLengths = lines.map(new GetLength());
  int totalLength = lineLengths.reduce(new Sum());

.. attention:: anonymous inner classes in Java can also access variables in the enclosing scope as long as they are marked final. Spark will ship copies of these variables to each worker node as it does for other languages.

**Python**

Spark’s API relies heavily on passing functions in the driver program to run on the cluster. There are three recommended ways to do this:

Lambda 表达式, for simple functions that can be written as an expression. (Lambdas do not support multi-statement functions or statements that do not return a value.)
Local defs inside the function calling into Spark, for longer code.
Top-level functions in a module.

For example, to pass a longer function than can be supported using a lambda, consider the code below:

.. code-block:: Python

  """MyScript.py"""
  if __name__ == "__main__":
      def myFunc(s):
          words = s.split(" ")
          return len(words)

      sc = SparkContext(...)
      sc.textFile("file.txt").map(myFunc)

Note that while it is also possible to pass a reference to a method in a class instance (as opposed to a singleton object), this requires sending the object that contains that class along with the method. For example, consider:

.. code-block:: Python

  class MyClass(object):
      def func(self, s):
          return s
      def doStuff(self, rdd):
          return rdd.map(self.func)

Here, if we create a new MyClass and call doStuff on it, the map inside there references the func method of that MyClass instance, so the whole object needs to be sent to the cluster.

In a similar way, accessing fields of the outer object will reference the whole object:

.. code-block:: Python

  class MyClass(object):
      def __init__(self):
          self.field = "Hello"
      def doStuff(self, rdd):
          return rdd.map(lambda s: self.field + s)


为了避免这个问题, 最简单的方式就是将字段拷贝到一个局部变量中, 而不是外部访问:

.. code-block:: Python

  def doStuff(self, rdd):
      field = self.field
      return rdd.map(lambda s: field + s)


理解闭包
-------------------------

Spark里一个比较难的事情就是，理解在整个集群上跨节点执行的变量和方法的作用域以及生命周期。Spark里一个频繁出现的问题就是RDD算子在变量作用域之外修改了其值。下面的例子，我们将会以foreach() 算子为例，来递增一个计数器counter，不过类似的问题在其他算子上也会出现。

示例
^^^^^^^^^^^^^^^^^^^^^^^

考虑如下例子，我们将会计算RDD中原生元素的总和，如果不是在同一个 JVM 中执行，其表现将有很大不同。例如，这段代码如果使用Spark本地模式（–master=local[n]）运行，和在集群上运行（例如，用spark-submit提交到YARN上）结果完全不同。

**Scala**

.. code-block:: Scala

  var counter = 0
  var rdd = sc.parallelize(data)

  // Wrong: Don't do this!!
  rdd.foreach(x => counter += x)

  println("Counter value: " + counter)

**Java**

.. code-block:: Java

  int counter = 0;
  JavaRDD<Integer> rdd = sc.parallelize(data);

  // Wrong: Don't do this!!
  rdd.foreach(x -> counter += x);

  println("Counter value: " + counter);

**Python**

.. code-block:: Python

  counter = 0
  rdd = sc.parallelize(data)

  # Wrong: Don't do this!!
  def increment_counter(x):
      global counter
      counter += x
  rdd.foreach(increment_counter)

  print("Counter value: ", counter)

本地模式 VS 集群模式
^^^^^^^^^^^^^^^^^^^^^^

上面这段代码其行为是不确定的。在本地模式下运行，所有代码都在运行于单个JVM中，所以RDD的元素都能够被累加并保存到counter变量中，这是因为本地模式下，counter变量和驱动器节点在同一个内存空间中。

然而，在集群模式下，情况会更复杂，以上代码的运行结果就不是所预期的结果了。为了执行这个作业，Spark会将 RDD 算子的计算过程分割成多个独立的任务（task）- 每个任务分发给不同的执行器（executor）去执行。而执行之前，Spark需要计算闭包。闭包是由执行器执行RDD算子（本例中的foreach()）时所需要的变量和方法组成的。闭包将会被序列化，并发送给每个执行器。由于本地模式下，只有一个执行器，所有任务都共享同样的闭包。而在其他模式下，情况则有所不同，每个执行器都运行于不同的worker节点，并且都拥有独立的闭包副本。

在上面的例子中，闭包中的变量会跟随不同的闭包副本，发送到不同的执行器上，所以等到foreach真正在执行器上运行时，其引用的counter已经不再是驱动器上所定义的那个counter副本了，驱动器内存中仍然会有一个counter变量副本，但是这个副本对执行器是不可见的！执行器只能看到其所收到的序列化闭包中包含的counter副本。因此，最终驱动器上得到的counter将会是0。

为了确保类似这样的场景下，代码能有确定的行为，这里应该使用累加器（Accumulator）。累加器是Spark中专门用于集群跨节点分布式执行计算中，安全地更新同一变量的机制。本指南中专门有一节详细说明累加器。

通常来说，闭包（由循环或本地方法组成），不应该改写全局状态。Spark中改写闭包之外对象的行为是未定义的。这种代码，有可能在本地模式下能正常工作，但这只是偶然情况，同样的代码在分布式模式下其行为很可能不是你想要的。所以，如果需要全局聚合，请记得使用累加器（Accumulator）。

打印 RDD 中的元素
^^^^^^^^^^^^^^^^^^^^^^^

另一种常见习惯是，试图用 rdd.foreach(println) 或者 rdd.map(println) 来打印RDD中所有的元素。如果是在单机上，这种写法能够如预期一样，打印出RDD所有元素。然后，在集群模式下，这些输出将会被打印到执行器的标准输出（stdout）上，因此驱动器的标准输出（stdout）上神马也看不到！如果真要在驱动器上把所有RDD元素都打印出来，你可以先调用collect算子，把RDD元素先拉到驱动器上来，代码可能是这样：rdd.collect().foreach(println)。不过如果RDD很大的话，有可能导致驱动器内存溢出，因为collect会把整个RDD都弄到驱动器所在单机上来；如果你只是需要打印一部分元素，那么take是更安全的选择：rdd.take(100).foreach(println)

使用键值对
-----------------------

大部分Spark算子都能在包含任意类型对象的RDD上工作，但也有一部分特殊的算子要求RDD包含的元素必须是键值对（key-value pair）。这种算子常见于做分布式混洗（shuffle）操作，如：以key分组或聚合。

在Scala中，这种操作在包含 Tuple2 （内建与scala语言，可以这样创建：(a, b) ）类型对象的RDD上自动可用。键值对操作是在 PairRDDFunctions 类上可用，这个类型也会自动包装到包含tuples的RDD上。

例如，以下代码将使用 reduceByKey 算子来计算文件中每行文本出现的次数：

.. code-block:: Scala

  val lines = sc.textFile("data.txt")
  val pairs = lines.map(s => (s, 1))
  val counts = pairs.reduceByKey((a, b) => a + b)

同样，我们还可以用 counts.sortByKey() 来对这些键值对按字母排序，最后再用 counts.collect() 将数据以对象数据组的形式拉到驱动器内存中。

注意：如果使用自定义类型对象做键值对中的key的话，你需要确保自定义类型实现了 equals() 方法（通常需要同时也实现hashCode()方法）。完整的细节可以参考：Object.hashCode()文档

转换算子 – transformation
---------------------------

以下是Spark支持的一些常用transformation算子。详细请参考 RDD API doc (Scala, Java, Python, R) 以及 键值对 RDD 函数 (Scala, Java) 。

=====================================================         ======================
Transformation算子                                             含义
=====================================================         ======================
map(func)                                                     返回一个新的分布式数据集，其中每个元素都是由源RDD中一个元素经func转换得到的。
filter(func)                                                  返回一个新的数据集，其中包含的元素来自源RDD中元素经func过滤后（func返回true时才选中）的结果
flatMap(func)                                                 类似于map，但每个输入元素可以映射到0到n个输出元素（所以要求func必须返回一个Seq而不是单个元素）
mapPartitions(func)                                           类似于map，但基于每个RDD分区（或者数据block）独立运行，所以如果RDD包含元素类型为T，则 func 必须是 Iterator<T> => Iterator<U> 的映射函数。
mapPartitionsWithIndex(func)                                  类似于 mapPartitions，只是func 多了一个整型的分区索引值，因此如果RDD包含元素类型为T，则 func 必须是 Iterator<T> => Iterator<U> 的映射函数。
sample(withReplacement, fraction, seed)                       采样部分（比例取决于 fraction ）数据，同时可以指定是否使用回置采样（withReplacement），以及随机数种子(seed)
union(otherDataset)                                           返回源数据集和参数数据集（otherDataset）的并集
intersection(otherDataset)                                    返回源数据集和参数数据集（otherDataset）的交集
distinct([numTasks]))                                         返回对源数据集做元素去重后的新数据集
groupByKey([numTasks])                                        只对包含键值对的RDD有效，如源RDD包含 (K, V) 对，则该算子返回一个新的数据集包含 (K, Iterable<V>) 对。注意：如果你需要按key分组聚合的话（如sum或average），推荐使用 reduceByKey或者 aggregateByKey 以获得更好的性能。注意：默认情况下，输出计算的并行度取决于源RDD的分区个数。当然，你也可以通过设置可选参数 numTasks 来指定并行任务的个数。
reduceByKey(func, [numTasks])                                 如果源RDD包含元素类型 (K, V) 对，则该算子也返回包含(K, V) 对的RDD，只不过每个key对应的value是经过func聚合后的结果，而func本身是一个 (V, V) => V 的映射函数。另外，和 groupByKey 类似，可以通过可选参数 numTasks 指定reduce任务的个数。
aggregateByKey(zeroValue)(seqOp, combOp, [numTasks])          如果源RDD包含 (K, V) 对，则返回新RDD包含 (K, U) 对，其中每个key对应的value都是由 combOp 函数 和 一个“0”值zeroValue 聚合得到。允许聚合后value类型和输入value类型不同，避免了不必要的开销。和 groupByKey 类似，可以通过可选参数 numTasks 指定reduce任务的个数。
sortByKey([ascending], [numTasks])                            如果源RDD包含元素类型 (K, V) 对，其中K可排序，则返回新的RDD包含 (K, V) 对，并按照 K 排序（升序还是降序取决于 ascending 参数）
join(otherDataset, [numTasks])                                如果源RDD包含元素类型 (K, V) 且参数RDD（otherDataset）包含元素类型(K, W)，则返回的新RDD中将包含内关联后key对应的 (K, (V, W)) 对。外关联(Outer joins)操作请参考 leftOuterJoin、rightOuterJoin 以及 fullOuterJoin 算子。
cogroup(otherDataset, [numTasks])                             如果源RDD包含元素类型 (K, V) 且参数RDD（otherDataset）包含元素类型(K, W)，则返回的新RDD中包含 (K, (Iterable<V>, Iterable<W>))。该算子还有个别名：groupWith
cartesian(otherDataset)                                       如果源RDD包含元素类型 T 且参数RDD（otherDataset）包含元素类型 U，则返回的新RDD包含前二者的笛卡尔积，其元素类型为 (T, U) 对。
pipe(command, [envVars])                                      以shell命令行管道处理RDD的每个分区，如：Perl 或者 bash 脚本。RDD中每个元素都将依次写入进程的标准输入（stdin），然后按行输出到标准输出（stdout），每一行输出字符串即成为一个新的RDD元素。
coalesce(numPartitions)                                       将RDD的分区数减少到numPartitions。当以后大数据集被过滤成小数据集后，减少分区数，可以提升效率。
repartition(numPartitions)                                    将RDD数据重新混洗（reshuffle）并随机分布到新的分区中，使数据分布更均衡，新的分区个数取决于numPartitions。该算子总是需要通过网络混洗所有数据。
repartitionAndSortWithinPartitions(partitioner)               根据partitioner（spark自带有HashPartitioner和RangePartitioner等）重新分区RDD，并且在每个结果分区中按key做排序。这是一个组合算子，功能上等价于先 repartition 再在每个分区内排序，但这个算子内部做了优化（将排序过程下推到混洗同时进行），因此性能更好。
=====================================================         ======================


动作算子 – action
------------------------------

以下是Spark支持的一些常用action算子。详细请参考 RDD API doc (Scala, Java, Python, R) 以及 键值对 RDD 函数 (Scala, Java) 。

===========================================       =================
Action算子                                          作用
===========================================       =================
reduce(func)                                       将RDD中元素按func进行聚合（func是一个 (T,T) => T 的映射函数，其中T为源RDD元素类型，并且func需要满足 交换律 和 结合律 以便支持并行计算）
collect()                                          将数据集中所有元素以数组形式返回驱动器（driver）程序。通常用于，在RDD进行了filter或其他过滤操作后，将一个足够小的数据子集返回到驱动器内存中。
count()                                            返回数据集中元素个数
first()                                            返回数据集中首个元素（类似于 take(1) ）
take(n)                                            返回数据集中前 n 个元素
takeSample(withReplacement,num, [seed])            返回数据集的随机采样子集，最多包含 num 个元素，withReplacement 表示是否使用回置采样，最后一个参数为可选参数seed，随机数生成器的种子。
takeOrdered(n, [ordering])                         按元素排序（可以通过 ordering 自定义排序规则）后，返回前 n 个元素
saveAsTextFile(path)                               将数据集中元素保存到指定目录下的文本文件中（或者多个文本文件），支持本地文件系统、HDFS 或者其他任何Hadoop支持的文件系统。保存过程中，Spark会调用每个元素的toString方法，并将结果保存成文件中的一行。
saveAsSequenceFile(path)(Java and Scala)           将数据集中元素保存到指定目录下的Hadoop Sequence文件中，支持本地文件系统、HDFS 或者其他任何Hadoop支持的文件系统。适用于实现了Writable接口的键值对RDD。在Scala中，同样也适用于能够被隐式转换为Writable的类型（Spark实现了所有基本类型的隐式转换，如：Int，Double，String 等）
saveAsObjectFile(path)(Java and Scala)             将RDD元素以Java序列化的格式保存成文件，保存结果文件可以使用 SparkContext.objectFile 来读取。
countByKey()                                       只适用于包含键值对(K, V)的RDD，并返回一个哈希表，包含 (K, Int) 对，表示每个key的个数。
foreach(func)                                      在RDD的每个元素上运行 func 函数。通常被用于累加操作，如：更新一个累加器（Accumulator ） 或者 和外部存储系统互操作。
===========================================       =================

注意：用 foreach 操作出累加器之外的变量可能导致未定义的行为。更详细请参考前面的“理解闭包”（Understanding closures ）这一小节。

混洗(Shuffle)算子
-------------------------

有一些Spark算子会触发众所周知的混洗（Shuffle）事件。Spark中的混洗机制是用于将数据重新分布，其结果是所有数据将在各个分区间重新分组。一般情况下，混洗需要跨执行器（Executor）或跨机器复制数据，这也是混洗操作一般都比较复杂而且开销大的原因。

背景
^^^^^^^^^^^^^

为了理解混洗阶段都发生了哪些事，我首先以 reduceByKey 算子为例来看一下。reduceByKey算子会生成一个新的RDD，将源RDD中一个key对应的多个value组合进一个tuple - 然后将这些values输入给reduce函数，得到的result再和key关联放入新的RDD中。这个算子的难点在于对于某一个key来说，并非其对应的所有values都在同一个分区（partition）中，甚至有可能都不在同一台机器上，但是这些values又必须放到一起计算reduce结果。

在Spark中，通常是由于为了进行某种计算操作，而将数据分布到所需要的各个分区当中。而在计算阶段，单个任务（task）只会操作单个分区中的数据 – 因此，为了组织好每个 reduceByKey 中 reduce 任务执行时所需的数据，Spark需要执行一个多对多操作。即，Spark需要读取RDD的所有分区，并找到所有key对应的所有values，然后跨分区传输这些values，并将每个key对应的所有values放到同一分区，以便后续计算各个key对应values的reduce结果 – 这个过程就叫做混洗（Shuffle）。

虽然混洗好后，各个分区中的元素和分区自身的顺序都是确定的，但是分区中元素的顺序并非确定的。如果需要混洗后分区内的元素有序，可以参考使用以下混洗操作：

* mapPartitions 使用 .sorted 对每个分区排序
* repartitionAndSortWithinPartitions 重分区的同时，对分区进行排序，比自行组合repartition和sort更高效
* sortBy 创建一个全局有序的RDD

会导致混洗的算子有：重分区（repartition）类算子，如： repartition 和 coalesce；ByKey 类算子(除了计数类的，如 countByKey) 如：groupByKey 和 reduceByKey；以及Join类算子，如：cogroup 和 join.

性能影响
^^^^^^^^^^^^^

混洗（Shuffle）之所以开销大，是因为混洗操作需要引入磁盘I/O，数据序列化以及网络I/O等操作。为了组织好混洗数据，Spark需要生成对应的任务集 – 一系列map任务用于组织数据，再用一系列reduce任务来聚合数据。注意这里的map、reduce是来自MapReduce的术语，和Spark的map、reduce算子并没有直接关系。

在Spark内部，单个map任务的输出会尽量保存在内存中，直至放不下为止。然后，这些输出会基于目标分区重新排序，并写到一个文件里。在reduce端，reduce任务只读取与之相关的并已经排序好的blocks。

某些混洗算子会导致非常明显的内存开销增长，因为这些算子需要在数据传输前后，在内存中维护组织数据记录的各种数据结构。特别地，reduceByKey和aggregateByKey都会在map端创建这些数据结构，而ByKey系列算子都会在reduce端创建这些数据结构。如果数据在内存中存不下，Spark会把数据吐到磁盘上，当然这回导致额外的磁盘I/O以及垃圾回收的开销。

混洗还会再磁盘上生成很多临时文件。以Spark-1.3来说，这些临时文件会一直保留到其对应的RDD被垃圾回收才删除。之所以这样做，是因为如果血统信息需要重新计算的时候，这些混洗文件可以不必重新生成。如果程序持续引用这些RDD或者垃圾回收启动频率较低，那么这些垃圾回收可能需要等较长的一段时间。这就意味着，长时间运行的Spark作业可能会消耗大量的磁盘。Spark的临时存储目录，是由spark.local.dir 配置参数指定的。

混洗行为可以由一系列配置参数来调优。参考Spark配置指南中"混洗行为"这一小节。


RDD持久化
=====================

Spark的一项关键能力就是它可以持久化（或者缓存）数据集在内存中，从而跨操作复用这些数据集。如果你持久化了一个RDD，那么每个节点上都会存储该RDD的一些分区，这些分区是由对应的节点计算出来并保持在内存中，后续可以在其他施加在该RDD上的action算子中复用（或者从这些数据集派生新的RDD）。这使得后续动作的速度提高很多（通常高于10倍）。因此，缓存对于迭代算法和快速交互式分析是一个很关键的工具。

你可以用persist() 或者 cache() 来标记一下需要持久化的RDD。等到该RDD首次被施加action算子的时候，其对应的数据分区就会被保留在内存里。同时，Spark的缓存具备一定的容错性 – 如果RDD的任何一个分区丢失了，Spark将自动根据其原来的血统信息重新计算这个分区。

另外，每个持久化的RDD可以使用不同的存储级别，比如，你可以把RDD保存在磁盘上，或者以java序列化对象保存到内存里（为了省空间），或者跨节点多副本，或者使用 Tachyon 存到虚拟机以外的内存里。这些存储级别都可以由persist()的参数StorageLevel对象来控制。cache() 方法本身就是一个使用默认存储级别做持久化的快捷方式，默认存储级别是 StorageLevel.MEMORY_ONLY（以Java序列化方式存到内存里）。完整的存储级别列表如下：

===================================    =======================
存储级别                                  含义
===================================    =======================
MEMORY_ONLY                            以未序列化的 Java 对象形式将 RDD 存储在 JVM 内存中。如果RDD不能全部装进内存，那么将一部分分区缓存，而另一部分分区将每次用到时重新计算。这个是Spark的RDD的默认存储级别。
MEMORY_AND_DISK                        以未序列化的Java对象形式存储RDD在JVM中。如果RDD不能全部装进内存，则将不能装进内存的分区放到磁盘上，然后每次用到的时候从磁盘上读取。
MEMORY_ONLY_SER                        以序列化形式存储 RDD（每个分区一个字节数组）。通常这种方式比未序列化存储方式要更省空间，尤其是如果你选用了一个比较好的序列化协议（fast serializer），但是这种方式也相应的会消耗更多的CPU来读取数据。
MEMORY_AND_DISK_SER                    和 MEMORY_ONLY_SER 类似，只是当内存装不下的时候，会将分区的数据吐到磁盘上，而不是每次用到都重新计算。
DISK_ONLY                              RDD 数据只存储于磁盘上。
MEMORY_ONLY_2, MEMORY_AND_DISK_2等      和上面没有”_2″的级别相对应，只不过每个分区数据会在两个节点上保存两份副本。
OFF_HEAP(实验性的)                       将RDD以序列化格式保存到Tachyon。与MEMORY_ONLY_SER相比，OFF_HEAP减少了垃圾回收开销，并且使执行器（executor）进程更小且可以共用同一个内存池，这一特性在需要大量消耗内存和多Spark应用并发的场景下比较吸引人。而且，因为RDD存储于Tachyon中，所以一个执行器挂了并不会导致数据缓存的丢失。这种模式下Tachyon 的内存是可丢弃的。因此，Tachyon并不会重建一个它逐出内存的block。如果你打算用Tachyon做为堆外存储，Spark和Tachyon具有开箱即用的兼容性。请参考这里，有建议使用的Spark和Tachyon的匹配版本对：page。
===================================    =======================

注意：在Python中存储的对象总是会使用 Pickle 做序列化，所以这时是否选择一个序列化级别已经无关紧要了。

Spark会自动持久化一些混洗操作（如：reduceByKey）的中间数据，即便用户根本没有调用persist。这么做是为了避免一旦有一个节点在混洗过程中失败，就要重算整个输入数据。当然，我们还是建议对需要重复使用的RDD调用其persist算子。

如何选择存储级别？
-------------------------

Spark的存储级别主要可于在内存使用和CPU占用之间做一些权衡。建议根据以下步骤来选择一个合适的存储级别：

* 如果RDD能使用默认存储级别（MEMORY_ONLY），那就尽量使用默认级别。这是CPU效率最高的方式，所有RDD算子都能以最快的速度运行。
* 如果步骤1的答案是否（不适用默认级别），那么可以尝试MEMORY_ONLY_SER级别，并选择一个高效的序列化协议（selecting a fast serialization library），这回大大节省数据对象的存储空间，同时速度也还不错。
* 尽量不要把数据吐到磁盘上，除非：1.你的数据集重新计算的代价很大；2.你的数据集是从一个很大的数据源中过滤得到的结果。否则的话，重算一个分区的速度很可能和从磁盘上读取差不多。
* 如果需要支持容错，可以考虑使用带副本的存储级别（例如：用Spark来服务web请求）。所有的存储级别都能够以重算丢失数据的方式来提供容错性，但是带副本的存储级别可以让你的应用持续的运行，而不必等待重算丢失的分区。
* 在一些需要大量内存或者并行多个应用的场景下，实验性的OFF_HEAP会有以下几个优势：
    * 这个级别下，可以允许多个执行器共享同一个Tachyon中内存池。
    * 可以有效地减少垃圾回收的开销。
    * 即使单个执行器挂了，缓存数据也不会丢失。

删除数据
-------------------------

Spark能够自动监控各个节点上缓存使用率，并且以LRU（最近经常使用）的方式将老数据逐出内存。如果你更喜欢手动控制的话，可以用RDD.unpersist() 方法来删除无用的缓存。


***********************
共享变量
***********************

一般而言，当我们给Spark算子（如 map 或 reduce）传递一个函数时，这些函数将会在远程的集群节点上运行，并且这些函数所引用的变量都是各个节点上的独立副本。这些变量都会以副本的形式复制到各个机器节点上，如果更新这些变量副本的话，这些更新并不会传回到驱动器（driver）程序。通常来说，支持跨任务的可读写共享变量是比较低效的。不过，Spark还是提供了两种比较通用的共享变量：广播变量和累加器。

广播变量
======================

广播变量提供了一种只读的共享变量，它是在每个机器节点上保存一个缓存，而不是每个任务保存一份副本。通常可以用来在每个节点上保存一个较大的输入数据集，这要比常规的变量副本更高效（一般的变量是每个任务一个副本，一个节点上可能有多个任务）。Spark还会尝试使用高效的广播算法来分发广播变量，以减少通信开销。

Spark的操作有时会有多个阶段（stage），不同阶段之间的分割线就是混洗操作。Spark会自动广播各个阶段用到的公共数据。这些方式广播的数据都是序列化过的，并且在运行各个任务前需要反序列化。这也意味着，显示地创建广播变量，只有在跨多个阶段（stage）的任务需要同样的数据 或者 缓存数据的序列化和反序列化格式很重要的情况下 才是必须的。

广播变量可以通过一个变量v来创建，只需调用 SparkContext.broadcast(v)即可。这个广播变量是对变量v的一个包装，要访问其值，可以调用广播变量的 value 方法。代码示例如下：

scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)

广播变量创建之后，集群中任何函数都不应该再使用原始变量v，这样才能保证v不会被多次复制到同一个节点上。另外，对象v在广播后不应该再被更新，这样才能保证所有节点上拿到同样的值（例如，更新后，广播变量又被同步到另一新节点，新节点有可能得到的值和其他节点不一样）。

累加器
=====================

累加器是一种只支持满足结合律的“累加”操作的变量，因此它可以很高效地支持并行计算。利用累加器可以实现计数（类似MapReduce中的计数器）或者求和。Spark原生支持了数字类型的累加器，开发者也可以自定义新的累加器。如果创建累加器的时候给了一个名字，那么这个名字会展示在Spark UI上，这对于了解程序运行处于哪个阶段非常有帮助（注意：Python尚不支持该功能）。

创捷累加器时需要赋一个初始值v，调用 SparkContext.accumulator(v) 可以创建一个累加器。后续集群中运行的任务可以使用 add 方法 或者 += 操作符 （仅Scala和Python支持）来进行累加操作。不过，任务本身并不能读取累加器的值，只有驱动器程序可以用 value 方法访问累加器的值。

以下代码展示了如何使用累加器对一个元素数组求和：

scala> val accum = sc.accumulator(0, "My Accumulator")
accum: spark.Accumulator[Int] = 0

scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum += x)
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
res2: Int = 10

以上代码使用了Spark内建支持的Int型累加器，开发者也可以通过子类化 AccumulatorParam 来自定义累加器。累加器接口（AccumulatorParam ）主要有两个方法：1. zero：这个方法为累加器提供一个“零值”，2.addInPlace 将收到的两个参数值进行累加。例如，假设我们需要为Vector提供一个累加机制，那么可能的实现方式如下：

.. code-block:: Scala

  object VectorAccumulatorParam extends AccumulatorParam[Vector] {
    def zero(initialValue: Vector): Vector = {
      Vector.zeros(initialValue.size)
    }
    def addInPlace(v1: Vector, v2: Vector): Vector = {
      v1 += v2
    }
  }

  // Then, create an Accumulator of this type:
  val vecAccum = sc.accumulator(new Vector(...))(VectorAccumulatorParam)


如果使用Scala，Spark还支持几种更通用的接口：1.Accumulable，这个接口可以支持所累加的数据类型与结果类型不同（如：构建一个收集元素的list）；2.SparkContext.accumulableCollection 方法可以支持常用的Scala集合类型。

对于在action算子中更新的累加器，Spark保证每个任务对累加器的更新只会被应用一次，例如，某些任务如果重启过，则不会再次更新累加器。而如果在transformation算子中更新累加器，那么用户需要注意，一旦某个任务因为失败被重新执行，那么其对累加器的更新可能会实施多次。

累加器并不会改变Spark懒惰求值的运算模型。如果在RDD算子中更新累加器，那么其值只会在RDD做action算子计算的时候被更新一次。因此，在transformation算子（如：map）中更新累加器，其值并不能保证一定被更新。以下代码片段说明了这一特性：

val accum = sc.accumulator(0)
data.map { x => accum += x; f(x) }
// 这里，accum任然是0，因为没有action算子，所以map也不会进行实际的计算


***********************
部署到集群
***********************

:ref:`submitting-applications` 中描述了如何向集群提交应用。换句话说，就是你需要把你的应用打包成 JAR文件（Java/Scala）或者一系列 .py 或 .zip 文件（Python），然后再用 bin/spark-submit 脚本将其提交给Spark所支持的集群管理器。


*****************************
从Java/Scala中启动Spark作业
*****************************

org.apache.spark.launcher 包提供了简明的Java API，可以将Spark作业作为子进程启动。


*****************************
单元测试
*****************************

Spark is friendly to unit testing with any popular unit test framework. Simply create a SparkContext in your test with the master URL set to local, run your operations, and then call SparkContext.stop() to tear it down. Make sure you stop the context within a finally block or the test framework’s tearDown method, as Spark does not support two contexts running concurrently in the same program.
Spark对所有常见的单元测试框架提供友好的支持。你只需要在测试中创建一个SparkContext对象，然后吧master URL设为local，运行测试操作，最后调用 SparkContext.stop() 来停止测试。注意，一定要在 finally 代码块或者单元测试框架的 tearDown方法里调用SparkContext.stop()，因为Spark不支持同一程序中有多个SparkContext对象同时运行。


*****************************
下一步
*****************************

你可以去 Spark 官网上看看示例程序（example Spark programs）。另外，Spark代码目录下也自带了不少例子，见 examples 目录(Scala,Java, Python, R)。你可以把示例中的类名传给 bin/run-example 脚本来运行这些例子；例如：

.. code-block:: Shell

  ./bin/run-example SparkPi

对于 Python 示例，使用 spark-submit：

.. code-block:: Shell

  ./bin/spark-submit examples/src/main/python/pi.py

对于 R 示例，使用 spark-submit：

.. code-block:: Shell

  ./bin/spark-submit examples/src/main/r/dataframe.R

配置（configuration）和调优（tuning）指南提供了不少最佳实践的信息，可以帮助你优化程序，特别是这些信息可以帮助你确保数据以一种高效的格式保存在内存里。集群模式概览这篇文章描述了分布式操作中相关的组件，以及Spark所支持的各种集群管理器。

最后，完整的API文件见：Scala, Java, Python 以及 R.
