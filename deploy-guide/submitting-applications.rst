.. _submitting-applications:

#######################
提交 Spark 应用程序
#######################

Spark bin 目录下的 spark-submit 脚本用于在集群中启动 Spark 应用程序。通过一个统一的接口它可以使用 Spark 支持的所有类型的集群管理器, 因此不需要为每个集群管理器专门配置你的应用程序。

应用程序依赖打包
---------------------------


如果你的代码依赖于其它工程，为了将代码发布到 Spark 集群, 你需要将应用程序的依赖项一起打包进来。这需要创建一个包含你自己代码和其依赖程序集 jar 包（或者 "uber" jar）。sbt 和 Maven 都有 assembly 插件。创建程序集 jar 包时，需要把 Spark 和 Hadoop 的 jar 包的依赖范围声明为 provided；因为这些 jar 包会由集群管理器在运行时提供，所以不需要再打包进来。一旦打完 jar 包之后，你就可以调用 bin/spark-submit 脚本来提交你的 jar 包了。

对于 Python，你可以使用 spark-submit 的 --py-files 参数，将你的程序以 .py、.zip 或 .egg 文件格式提交给集群。如果你需要依赖很多 Python 文件，我们推荐你将它们打成一个 .zip 或者 .egg 包。

使用 spark-submit 启动应用程序
-------------------------------------

打包好一个应用程序之后，就可以使用 bin/spark-submit 脚本来提交它。这个脚本会负责设置 Spark 及其依赖的 classpath，同时它可以支持 Spark 所支持的所有不同类型的集群管理器和部署模式：

.. code-block:: Shell

  ./bin/spark-submit \
    --class <main-class>
    --master <master-url> \
    --deploy-mode <deploy-mode> \
    --conf <key>=<value> \
    ... # 其他选项
    <application-jar> \
    [application-arguments]


一些常用选项如下：

* --class: 应用程序的入口（例如：org.apache.spark.examples.SparkPi）
* --master: 集群的 master URL（如：spark://23.195.26.187:7077）
* --deploy-mode: Driver 进程是在集群的 Worker 节点上运行（cluster模式），还是在本地作为一个外部客户端运行（client模式）（默认值是：client）
* --conf: 可以设置任意的Spark配置属性，键值对（key=value）格式。如果值中包含空白字符，可以用双引号括起来（"key=value"）。
* application-jar: 应用程序jar包路径，该jar包必须包括你自己的代码及其所有的依赖项。如果是URL，那么该路径URL必须是对整个集群可见且一致的，如：hdfs://path 或者 file://path （要求对所有节点都一致）
* application-arguments: 传给应用程序入口类 main 函数的启动参数，可选。

一种常见的部署策略是，从一台距离 Worker 节点的物理距离比较近的网关机器上提交你的应用程序(例如 Standalone EC2 集群中的 Master 节点)。这种情况下比较适合使用 client 模式。client 模式下，Driver 直接运行在 spark-submit 的进程中，对于集群来说它就像是一个客户端。应用程序的输入输出也被绑定到控制台上。因此，这种模式尤其适合于交互式执行（REPL）的应用程序，(例如 spark-shell)。

当然, 如果从距离 Worker 节点很远的机器(例如你的笔记本)上提交应用程序，通常使用 cluster 模式来尽量减少 Driver 和 Executor 之间的网络延迟。目前，只有 YARN 支持 Python 应用程序的 cluster 模式部署。

对于 Python 应用，只要把 <application-jar> 替换成一个 .py 文件，再把 Python 的 .zip、.egg 或者 .py 文件传给 --py-files 参数即可。

只有很少几个参数是专门用于所使用的集群管理器。例如，对于一个使用 cluster 模式部署的 Spark Standalone集群，你可以指定 --supervise 参数来确保 Driver 在异常退出码非0的情况下能够自动重启。运行 spark-submit --help 命令可查看所有这样的选项列表。下面是常用选项的几个示例：

.. code-block:: shell

  # 本地运行应用程序，使用8个core
  ./bin/spark-submit \
    --class org.apache.spark.examples.SparkPi \
    --master local[8] \
    /path/to/examples.jar \
    100

  # 在 client 部署模式中的一个 Spark 独立集群上运行应用程序，
  ./bin/spark-submit \
    --class org.apache.spark.examples.SparkPi \
    --master spark://207.184.161.138:7077 \
    --executor-memory 20G \
    --total-executor-cores 100 \
    /path/to/examples.jar \
    1000

  # 在 cluster 部署模式中的一个 Spark Standalone集群上运行应用程序，异常退出时自动重启
  ./bin/spark-submit \
    --class org.apache.spark.examples.SparkPi \
    --master spark://207.184.161.138:7077 \
    --deploy-mode cluster
    --supervise
    --executor-memory 20G \
    --total-executor-cores 100 \
    /path/to/examples.jar \
    1000

  # 在 YARN 集群上运行应用程序
  export HADOOP_CONF_DIR=XXX
  ./bin/spark-submit \
    --class org.apache.spark.examples.SparkPi \
    --master yarn \
    --deploy-mode cluster \  # 对于 client 模式其值为 client
    --executor-memory 20G \
    --num-executors 50 \
    /path/to/examples.jar \
    1000

  # 在一个 Spark Standalone集群上运行 python 应用程序
  ./bin/spark-submit \
    --master spark://207.184.161.138:7077 \
    examples/src/main/python/pi.py \
    1000

  # 在 cluster 部署模式中的一个 Mesos 集群上运行应用程序，异常时自动重启
  ./bin/spark-submit \
    --class org.apache.spark.examples.SparkPi \
    --master mesos://207.184.161.138:7077 \
    --deploy-mode cluster
    --supervise
    --executor-memory 20G \
    --total-executor-cores 100 \
    http://path/to/examples.jar \
    1000


Master URLs
-------------------------------------

传给 Spark 的 master URL 可以是以下几种格式：

==================   =============
Master URL           含义
==================   =============
local                本地运行 Spark，只使用 1 个 Worker 线程（即没有并行计算）
local[K]             本地运行 Spark，使用 K 个 Worker 线程（理想情况是将这个值设为你机器上 CPU core的个数）
local[K,F]           本地运行 Spark，使用 K 个 Worker 线程并且允许 Task 最大失败次数为 F (see spark.task.maxFailures for an explanation of this variable)
local[*]             本地运行 Spark，使用 Worker 线程数和机器上逻辑 CPU core个数一样。
local[*,F]           本地运行 Spark，使用 Worker 线程数和机器上逻辑 CPU core个数一样并且允许 Task 最大失败次数为 F。
spark://HOST:PORT    连接到指定的 Spark Standalone 集群的 master。端口是可以配置的，默认是7077。
mesos://HOST:PORT    连接到指定的 Mesos 集群。端口号是可以配置的，默认是 5050。如果 Mesos 集群依赖于 ZooKeeper，可以使用 mesos://zk://… 来提交，注意 --deploy-mode需要设置为 cluster，同时，HOST:PORT 应指向 MesosClusterDispatcher.
yarn                 连接到指定的 YARN  集群，使用--deploy-mode 来指定 client 模式或是 cluster 模式。YARN 集群位置需要通过 $HADOOP_CONF_DIR 或者 $YARN_CONF_DIR 变量来查找。
==================   =============



从文件中加载配置
---------------------------------------

spark-submit 脚本可以从一个属性文件加载默认的 Spark配置值，并将这些属性值传给你的应用程序。Spark 默认会从 Spark 安装目录中的 conf/spark-defaults.conf 文件读取这些属性配置。更详细信息，请参考 加载默认配置 这篇文章。

用这种方式加载默认Spark属性配置，可以在调用 spark-submit 脚本时省略一些参数标志。例如：如果属性文件中设置了 spark.master 属性，那么你就可以忽略 spark-submit 的 --master参数。通常，在代码里显示地在 SparkConf 对象上设置的参数具有最高的优先级，其次是 spark-submit 中传的参数，再次才是spark-defaults.conf文件中的配置值。

如果你总是搞不清楚最终生效的配置值是从哪里来的，你可以通过 spark-submit 的 —verbose 选项来打印细粒度的调试信息。

高级依赖管理
---------------------------------------

使用 spark-submit 时，application jar 和 --jars 选项指定的 jar 包都会自动传到集群上。—jars后面的 URL 必须以逗号分隔。这个列表已经包含在驱动器和执行器的类路径上扩展目录不支持 —jars。

Spark 使用下面的URL协议以允许不同的 jar 包分发策略：

* file: – 文件绝对路径，并且file:/URI是通过驱动器的HTTP文件服务器来下载的，每个执行器都从驱动器的HTTP server拉取这些文件。
* hdfs:, http:, https:, ftp: – 设置这些参数后，Spark将会从指定的URI位置下载所需的文件和jar包。
* local: –  local:/ 打头的URI用于指定在每个工作节点上都能访问到的本地或共享文件。这意味着，不会占用网络IO，特别是对一些大文件或jar包，最好使用这种方式，当然，你需要把文件推送到每个工作节点上，或者通过NFS和GlusterFS共享文件。

注意，每个 SparkContext 对应的 jar 包和文件都需要拷贝到所对应 Executor 的工作目录下。一段时间之后，这些文件可能会占用相当多的磁盘。在 YARN 上，这些清理工作是自动完成的；而在Spark独立部署时，这种自动清理需要配置 spark.worker.cleanup.appDataTtl 属性。

用户还可以用 --packages 参数，通过给定一个逗号分隔的 maven 坐标，来指定其它依赖项。这个命令会自动处理依赖树。额外的 maven库（或者SBT resolver）可以通过 --repositories 参数来指定。Spark 命令（pyspark，spark-shell，spark-submit）都支持这些参数。

对于 Python，也可以使用等价的 --py-files 选项来分发 .egg、.zip 以及 .py 库到执行器上。

更多信息
---------------------------------------

部署完了你的应用程序后，集群模式概览 一文中描述了分布式执行中所涉及到的各个组件，以及如何监控和调试应用程序。
