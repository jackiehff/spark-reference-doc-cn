Spark 独立模式
============================

除了可以在 Mesos 和 YARN 集群管理器上运行之外，Spark 还提供一种简单的独立部署模式。你既可以通过手工启动一个master和多个worker来手动地启动一个独立集群，也可以使用我们提供的启动脚本来启动一个独立集群。为了方便测试，你也可以在单机上运行这些后台程序。

Spark集群独立安装
-------------------------

要独立安装Spark，你只需要将编译好的 Spark 版本复制到集群中每一个节点上即可。你可以下载Spark 每个 release 的预编译版本，也可以自己构建Spark。

手动启动集群
-------------------------

执行以下命令，就可以启动一个独立的 master 服务器：

.. code-block:: shell

  ./sbin/start-master.sh


启动完成之后，master 自己会打印出一个 spark://HOST:PORT URL，你可以使用这个URL将worker 连接到 master，或者作为master参数传递给。你还可以在 master 的 web UI（默认地址是：http://localhost:8080）上查看 master URL。

类似地，你可以通过以下命令，启动一个或多个worker节点，并将其连接到 master：

.. code-block:: shell

  ./sbin/start-slave.sh <master-spark-URL>


启动一个 worker 以后，查看 master 的 web UI（默认地址是：http://localhost:8080），你应该可以看到一个新的节点，以及节点的CPU个数和内存(为操作系统预留1GB)。

最后，下面的配置选项将可以传给 master 和 worker：

========================    =======================
参数                         含义
========================    =======================
-h HOST, --host HOST        监听的主机名
-i HOST, --ip HOST          监听的主机名（已经废弃，请使用-h 或者--host）
-p PORT, --port PORT        服务监听的端口（master节点默认7077，worker节点随机）
--webui-port PORT           web UI端口（master节点默认8080，worker节点默认8081）
-c CORES, --cores CORES     Spark应用程序能够使用的CPU core数上限（默认值是：所有可用的CPU core个数）; 仅worker节点有效
-m MEM, --memory MEM        Spark应用程序能够使用的内存上限，格式为 1000M 或者 2G（默认值是：机器总内存减去1G）；仅worker节点有效
-d DIR, --work-dir DIR      工作目录，同时 job 的日志也输出到该目录（默认值是：${SPAKR_HOME}/work）; 仅worker节点有效
--properties-file FILE      自定义 Spark 属性文件的加载路径（默认值是：conf/spark-defaults.conf）
========================    =======================


集群启动脚本
----------------------

要使用启动脚本来启动一个Spark独立集群，你应该在 Spark 安装目录下创建一个文件conf/slaves，它包括你打算作为worker节点启动的每一台机器的主机名（或IP），每行一台机器。如果 conf/slaves 文件不存在，启动脚本会默认会用单机方式启动，这种方式非常方便测试。注意，master 节点访问各个 worker 时使用 ssh。默认情况下，你需要配置 ssh免密码登陆（使用秘钥文件）。如果你没有设置免密码登陆，那么你也可以通过环境变量SPARK_SSH_FOREGROUND来一个一个地设置每个 worker 的密码。

设置好 conf/slaves 文件以后，你就可以使用以下基于 Hadoop 部署脚本的 shell 脚本来启动或停止集群，这些脚本都在 ${SPARK_HOME}/sbin 目录下：

* sbin/start-master.sh – 在执行该脚本的机器上启动一个 master 实例
* sbin/start-slaves.sh – conf/slaves 文件中指定的每一台机器上都启动一个 slave 实例
* sbin/start-slave.sh – 在执行该脚本的机器上启动一个 slave 实例
* sbin/start-all.sh – 启动一个 master 和多个 slave 实例，详细见上面的描述。
* sbin/stop-master.sh – 停止 start-master.sh 所启动的 master 实例
* sbin/stop-slaves.sh – 停止所有在 conf/slaves 中指定的 slave 实例
* sbin/stop-all.sh – 停止 master 节点和所有的 slave 节点，详细见上面的描述

注意，这些脚本都需要在你启动Spark master的机器上运行，而不是你的本地机器。

通过在 conf/spark-env.sh 文件中设置环境变量, 你可以进一步的配置集群。可以通过复制 conf/spark-env.sh.template 来创建这个文件，并且为了使这些环境变量设置生效你还需要将其拷贝到所有worker节点上。以下是可用的设置：

========================    ======================
环境变量                      含义
========================    ======================
SPARK_MASTER_IP             master实例绑定的IP地址，例如，绑定到一个公网IP
SPARK_MASTER_PORT           mater实例绑定的端口（默认7077）
SPARK_MASTER_WEBUI_PORT     master web UI的端口（默认8080）
SPARK_MASTER_OPTS           master专用配置属性，格式如”-Dx=y” （默认空），可能的选项请参考下面的列表。
SPARK_LOCAL_DIRS            Spark的本地工作目录，包括：映射输出的临时文件和RDD保存到磁盘上的临时数据。这个目录需要快速访问，最好设成本地磁盘上的目录。也可以通过使用逗号分隔列表，将其设成多个磁盘上的不同路径。
SPARK_WORKER_CORES          本机上Spark应用程序可以使用的CPU core上限（默认所有CPU core）
SPARK_WORKER_MEMORY         本机上Spark应用程序可以使用的内存上限，如：1000m，2g（默认为本机总内存减去1GB）；注意每个应用单独使用的内存大小要用 spark.executor.memory 属性配置的。
SPARK_WORKER_PORT           Spark worker绑定的端口（默认随机）
SPARK_WORKER_WEBUI_PORT     worker web UI端口（默认8081）
SPARK_WORKER_INSTANCES      每个slave机器上启动的worker实例个数（默认：1）。如果你的slave机器非常强劲，可以把这个值设为大于1；相应的，你需要设置SPARK_WORKER_CORES参数来显式地限制每个worker实例使用的CPU个数，否则每个worker实例都会使用所有的CPU。
SPARK_WORKER_DIR            Spark worker的工作目录，包括worker的日志以及临时存储空间（默认：${SPARK_HOME}/work）
SPARK_WORKER_OPTS           worker的专用配置属性，格式为："-Dx=y"，可能的选项请参考下面的列表。
SPARK_DAEMON_MEMORY         Spark master和worker后台进程所使用的内存（默认：1g）
SPARK_DAEMON_JAVA_OPTS      Spark master和workers后台进程所使用的JVM选项，格式为："-Dx=y"（默认空）
SPARK_PUBLIC_DNS            Spark master和workers使用的公共DNS（默认空）
========================    ======================

注意: 启动脚本目前不支持Windows。如需在Windows上运行，请手工启动 master 和 workers。

SPARK_MASTER_OPTS支持以下系统属性：

==================================    ============    ============
属性名                                 默认值           含义
==================================    ============    ============
spark.deploy.retainedApplications       200           web UI上最多展示几个已结束应用。更早的应用的数将被删除。
spark.deploy.retainedDrivers            200           web UI上最多展示几个已结束的驱动器。更早的驱动器进程数据将被删除。
spark.deploy.spreadOut                  true          独立部署集群的master是否应该尽可能将应用分布到更多的节点上；设为true，对数据本地性支持较好；设为false，计算会收缩到少数几台机器上，这对计算密集型任务比较有利。
spark.deploy.defaultCores               (无限制)       Spark独立模式下应用程序默认使用的CPU个数（没有设置spark.cores.max的情况下）。如果不设置，则为所有可用CPU个数（除非设置了spark.cores.max）。如果集群是共享的，最好将此值设小一些，以避免用户占满整个集群。
spark.worker.timeout                    60            如果master没有收到worker的心跳，那么将在这么多秒之后，master将丢弃该worker。
==================================    ============    ============

SPARK_WORKER_OPTS支持以下系统属性：

==================================    ========================    ============
属性名                                 默认值                       含义
==================================    ========================    ============
spark.worker.cleanup.enabled          false                       是否定期清理 worker 和应用的工作目录。注意，该设置仅在独立模式下有效，YARN有自己的清理方式；同时，只会清理已经结束的应用对应的目录。
spark.worker.cleanup.interval         1800 (30 minutes)           worker清理本地应用工作目录的时间间隔（秒）
spark.worker.cleanup.appDataTtl       7 * 24 * 3600 (7 days)      清理多久以前的应用的工作目录。这个选项值将取决于你的磁盘总量。spark应用会将日志和jar包都放在其对应的工作目录下。随着时间流逝，应用的工作目录很快会占满磁盘，尤其是在你的应用提交比较频繁的情况下。
==================================    ========================    ============

连接应用程序到集群
-----------------------

要在 Spark 集群上运行一个应用程序，只需把 spark://IP:PORT 这个master URL 传给SparkContext构造器。

如需要运行交互式的spark shell，运行如下命令：

.. code-block:: shell

  ./bin/spark-shell --master spark://IP:PORT


你也可以通过设置选项 --total-executor-cores <numCores> 来控制 spark-shell 在集群上使用的CPU 核心总数。

启动Spark应用
-----------------------

spark-submit脚本提供了最直接的方式来提交一个编译好的Spark应用程序到集群上。对于独立集群来说，Spark目前支持两种部署模式。在客户端（client）模式下，驱动器进程（driver）将在提交应用程序的机器上启动。而在集群（cluster）模式下，驱动器（driver）将会在集群中的某一台worker上启动，同时提交应用程序的客户端进程在完成提交应用程序之后立即退出，而不会等到Spark应用运行结束。

如果你的应用程序是通过Spark submit 来启动的，那么应用程序 jar 包会自动分发到所有的 Worker节点上。应用程序所依赖的任何额外的jar包，都必须在 --jars 参数中指明，并以逗号分隔（如：–jars jar1,jar2）。

另外，独立集群模式还支持异常退出（返回值非0）时自动重启。想要使用这个特性，你需要在启动应用程序时将 --supervise 标识传递给 spark-submit。随后如果你需要杀掉一个不断失败的应用程序，你可能需要运行如下指令：

.. code-block:: shell

  ./bin/spark-class org.apache.spark.deploy.Client kill <master url> <driver ID>


你可以在 master web UI（http://<master url>:8080）上查看驱动器ID。

资源调度
-----------------------

独立集群模式目前只支持简单的先进先出（FIFO）调度器。这个调度器可以支持多用户，你可以控制每个应用程序所使用的最大资源。默认情况下，Spark应用会申请集群中所有的CPU，这不太合理，除非你的集群同一时刻只运行一个应用程序。你可以通过在 SparkConf 中设置 spark.cores.max 来限制其使用的CPU核心总数。例如：

.. code-block:: Scala

  val conf = new SparkConf()
    .setMaster(...)
    .setAppName(...)
    .set("spark.cores.max", "10")
  val sc = new SparkContext(conf)

另外，你也可以通过 conf/spark-env.sh 中的 spark.deploy.defaultCores 设置应用默认使用的CPU个数（特别针对没有设置 spark.cores.max 的应用）。

.. code-block:: shell

  export SPARK_MASTER_OPTS="-Dspark.deploy.defaultCores=<value>"


在一些共享的集群上，用户很可能忘记单独设置一个最大CPU限制，那么这个参数将很有用。

监控和日志
-----------------------

Spark 独立安装模式提供了一个基于web的集群监控用户界面。master 和每个 worker 都有其对应的web UI，展示集群和 Spark 作业的统计数据。默认情况下，你可以在 master 机器的8080端口上访问到这个 web UI。这个端口可以通过配置文件或者命令行来设置。

另外，每个作业的详细日志，将被输出到每个 slave 节点上的工作目录下（默认为：${SPARK_HOME}/work）。每个 Spark 作业下都至少有两个日志文件，stdout 和 stderr，这里将包含所有的输出到控制台的信息。

和Hadoop同时运行
-----------------------

你可以让 Spark 和现有的Hadoop集群同时运行，只需要将Spark作为独立的服务在同样的机器上启动即可。这样Spark可以通过hdfs:// URL来访问Hadoop上的数据（通常情况下是，hdfs://<namenode>:9000/path，你可以在Hadoop Namenode的web UI上找到正确的链接）。当然，你也可以为 Spark 部署一个独立的集群，这时候 Spark 仍然可以通过网络访问 HDFS 上的数据；这会比访问本地磁盘慢一些，但如果Spark和Hadoop集群都在同一个本地局域网内的话，问题不大（例如，你可以在Hadoop集群的每个机架上新增一些部署Spark的机器）。

网络安全端口配置
-----------------------

Spark会大量使用网络资源，而有些环境会设置严密的防火墙设置，以严格限制网络访问。完整的端口列表，请参考这里：security page.

高可用性
-----------------------

默认情况下，独立调度的集群能够容忍worker节点的失败（在Spark本身来说，它能够将失败的工作移到其他worker节点上）。然而，调度器需要master做出调度决策，而这（默认行为）会造成单点失败：如果master挂了，任何应用都不能提交和调度。为了绕过这个单点问题，我们有两种高可用方案，具体如下：

基于Zookeeper的热备master
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

概要

利用Zookeeper来提供领导节点选举以及一些状态数据的存储，你可以在集群中启动多个master并连接到同一个Zookeeper。其中一个将被选举为“领导”，而另一个将处于备用（standby）状态。如果“领导”挂了，则另一个master会立即被选举，并从Zookeeper恢复已挂“领导”的状态，并继续调度。整个恢复流程（从“领导”挂开始计时）可能需要1到2分钟的时间。注意，整个延时只会影响新增应用 – 已经运行的应用不会受到影响。

更多关于Zookeeper信息请参考这里：here

配置

要启用这种恢复模式，你可以在spark-env中设置 SPARK_DAEMON_JAVA_OPTS，有关这些配置的更多信息，请参阅配置文档

可能的问题：如果你有多个master，但没有正确设置好master使用Zookeeper的配置，那么这些master彼此都不可见，并且每个master都认为自己是“领导”。这件会导致整个集群处于不稳定状态（多个master都会独立地进行调度）

详细

如果你已经有一个Zookeeper集群，那么启动高可用特性是很简单的。只需要在不同节点上启动多个master，并且配置相同的Zookeeper（包括Zookeeper URL和目录）即可。masters可以随时添加和删除。

在调度新提交的Spark应用或者新增worker节点时，需要知道当前”领导“的IP地址。你只需要将以前单个的master地址替换成多个master地址即可。例如，你可以在SparkContext中设置master URL为spark://host1:port1.host2:port2。这会导致SparkContext在两个master中都进行登记 – 那么这时候，如果host1挂了，这个应用的配置同样可以在新”领导“（host2）中找到。

”在master注册“和普通操作有一个显著的区别。在Spark应用或worker启动时，它们需要找当前的”领导“master，并在该master上注册。一旦注册成功，它们的状态将被存储到Zookeeper上。如果”老领导“挂了，”新领导“将会联系所有之前注册过的Spark应用和worker并通知它们领导权的变更，所以Spark应用和worker在启动时甚至没有必要知道”新领导“的存在。

由于这一特性，新的master可以在任何时间添加进来，你唯一需要关注的就是，新的应用或worker能够访问到这个master。总之，只要应用和worker注册成功，其他的你都不用管了。

基于本地文件系统的单点恢复
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

概要

利用Zookeeper当然是生成环境下高可用的最佳选择，但有时候你仍然希望在master挂了的时候能够重启之，FILESYSTEM模式能帮你实现这一需求。当应用和worker注册到master的时候，他们的状态都将被写入文件系统目录中，一旦master挂了，你只需要重启master，这些状态都能够恢复。

配置

要使用这种恢复模式，你需要在spark-env中设置SPARK_DAEMON_JAVA_OPTS，可用的属性如下：

================================    ================================
系统属性                              含义
================================    ================================
spark.deploy.recoveryMode           设为FILESYSTEM以启用单点恢复模式（默认空）
spark.deploy.recoveryDirectory      用于存储可恢复状态数据的目录，master进程必须有访问权限
================================    ================================


Details（详细）

* 这个解决方案可以用于和一些监控、管理系统进行串联（如：monit），或者与手动重启相结合。
* 至少，基于文件系统工单恢复总比不能恢复强；同时，这种恢复模式也会是开发或者实验场景下的不错的选择。在某些情况下，通过stop-master.sh杀死master可能不会清理其状态恢复目录下的数据，那么这时候你启动或重启master，将会进入恢复模式。这可能导致master的启动时间长达一分钟（master可能要等待之前注册的worker和客户端超时）。
* 你也可以使用NFS目录来作为数据恢复目录（虽然这不是官方声明支持的）。如果老的master挂了，你可以在另一个节点上启动master，这个master只要能访问同一个NFS目录，它就能够正确地恢复状态数据，包括之前注册的worker和应用（等价于Zookeeper模式）。后续的应用必须使用新的master来进行注册。
