在 YARN 上运行 Spark
==============================

Spark 对 YARN (Hadoop NextGen) 的支持是从 0.6.0 版本开始的，后续的版本也在持续的改进。

在YARN上启动
-------------------------

首先要确保 HADOOP_CONF_DIR 或者 YARN_CONF_DIR 指向一个包含 Hadoop 集群客户端配置文件的目录。这些配置用于向 HDFS 写数据以及连接到 YARN 的 ResourceManager（资源管理器）。这个目录下的配置将会分发到 YARN 集群上的各个节点，这样应用程序使用的所有YARN 容器都将使用同样的配置。如果这些配置引用了 Java 系统属性或其它不属于 YARN 管理的环境变量，那么这些系统属性和环境变量也应该在 Spark 应用程序的配置中设置（包括 Driver、Executors，以及运行于 client 模式时的 YARN Application Master, 简称 AM）。

有两种部署模式可用于在 YARN 上启动 Spark 应用程序。在 cluster 模式下，Spark driver 在YARN Application Master中运行（运行于集群中），因此客户端可以在Spark应用启动之后关闭退出。而client模式下，Spark驱动器在客户端进程中，这时的YARN Application Master只用于向YARN申请资源。

不同于 Spark standalone 和 Mesos 模式，YARN 的master地址不是在–master参数中指定的，而是在Hadoop配置文件中设置。因此，这种情况下，–master只需设置为yarn。

以下用cluster模式启动一个Spark应用：

.. code-block:: shell

  $ ./bin/spark-submit --class path.to.your.Class --master yarn --deploy-mode cluster [options] <app jar> [app options]

例如：

.. code-block:: shell

  $ ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
      --master yarn \
      --deploy-mode cluster \
      --driver-memory 4g \
      --executor-memory 2g \
      --executor-cores 1 \
      --queue thequeue \
      lib/spark-examples*.jar 10


以上例子中，启动了一个YARN客户端程序，使用默认的Application Master。而后SparkPi在Application Master中的子线程中运行。客户端会周期性的把Application Master的状态信息拉取下来，并更新到控制台。客户端会在你的应用程序结束后退出。参考“调试你的应用”，这一节说明了如何查看驱动器和执行器的日志。

要以client模式启动一个spark应用，只需在上面的例子中把cluster换成client。下面这个例子就是以client模式启动spark-shell：

.. code-block:: shell

  $ ./bin/spark-shell --master yarn --deploy-mode client


增加其他JAR包
----------------------

在cluster模式下，驱动器不在客户端机器上运行，所以SparkContext.addJar添加客户端本地文件就不好使了。要使客户端上本地文件能够用SparkContext.addJar来添加，可以用–jars选项：

.. code-block:: shell

  $ ./bin/spark-submit --class my.main.Class \
      --master yarn \
      --deploy-mode cluster \
      --jars my-other-jar.jar,my-other-other-jar.jar \
      my-main-jar.jar \
      app_arg1 app_arg2


准备
----------------------

在YARN上运行Spark需要其二进制发布包构建的时候增加YARN支持。二进制发布包可以在这里下载：downloads page 。

想要自己编译，参考这里： Building Spark

配置
-----------------------

大多数配置，对于YARN或其他集群模式下，都是一样的。详细请参考这里： configuration page。

以下是YARN上专有的配置项。

调试应用程序
-----------------------

在YARN术语集中，执行器和Application Master在容器（container）中运行。YARN在一个应用程序结束后，有两种处理容器日志的模式。如果开启了日志聚合（yarn.log-aggregation-enable），那么容器日志将被复制到HDFS，并删除本地日志。而后这些日志可以在集群任何节点上用yarn logs命令查看：

.. code-block:: shell

  yarn logs -applicationId <app ID>


以上命令，将会打印出指定应用的所有日志文件的内容。你也可以直接在HDFS上查看这些日志（HDFS shell或者HDFS API）。这些目录可以在你的YARN配置中指定（yarn.nodemanager.remote-app-log-dir和yarn.nodemanager-remote-app-log-dir-suffix）。这些日志同样还可以在Spark Web UI上Executors tab页查看。当然，你需要启动Spark history server和 MapReduce history server，再在 yarn-site.xml 中配置好 yarn.log.server.url。Spark history server UI 将把你重定向到MapReduce history server 以查看这些聚合日志。

如果日志聚合没有开启，那么日志文件将在每台机器上的 YARN_APP_LOGS_DIR 目录保留，通常这个目录指向 /tmp/logs 或者 $HADOOP_HOME/log/userlogs（这取决于Hadoop版本和安全方式）。查看日志的话，需要到每台机器上查看这些目录。子目录是按 application ID 和 container ID来组织的。这些日志同样可以在 Spark Web UI 上 Executors tab 页查看，而且这时你不需要运行MapReduce history server。

如果需要检查各个容器的启动环境，可以先把 yarn.nodemanager.delete.debug-delay-sec 增大（如：36000），然后访问应用缓存目录yarn.nodemanager.local-dirs，这时容器的启动目录。这里包含了启动脚本、jar包以及容器启动所用的所有环境变量。这对调试 classpath 相关问题尤其有用。（注意，启用这个需要管理员权限，并重启所有的node managers，因此，对托管集群不适用）

要自定义Application Master或执行器的 log4j 配置，有如下方法：
* 通过spark-submit –files 上传一个自定义的 log4j.properties 文件。
* 在 spark.driver.extraJavaOptions（对Spark驱动器）或者 spark.executor.extraJavaOptions（对Spark执行器）增加 -Dlog4j.configuration=<location of configuration file>。注意，如果使用文件，那么 file: 协议头必须显式写上，且文件必须在所节点上都存在。
* 更新 ${SPARK_CONF_DIR}/log4j.properties 文件以及其他配置。注意，如果在多个地方都配置了log4j，那么上面其他两种方法的配置优先级比本方法要高。
注意，第一种方法中，执行器和Application Master共享同一个log4j配置，在有些环境下（AM和执行器在同一个节点上运行）可能会有问题（例如，AM和执行器日志都写入到同一个日志文件）

如果你需要引用YARN放置日志文件的路径，以便YARN可以正确地展示和聚合日志，请在log4j.properties文件中使用spark.yarn.app.container.log.dir。例如，log4j.appender.file_appender.File=${spark.yarn.app.container.log.dir}/spark.log 。对于流式应用，可以配置RollingFileAppender，并将文件路径设置为YARN日志目录，以避免磁盘打满，而且这些日志还可以利用YARN的日志工具访问和查看。

====================================================      ===============================================      =======================================
Spark属性                                                  默认值                                                含义
====================================================      ===============================================      =======================================
spark.yarn.am.memory                                      512m                                                  YARN Application Master在client模式下，使用内存总量，与JVM内存设置格式相同（如：512m，2g）。如果是cluster模式下，请设置 spark.driver.memory。注意使用小写的后缀，如：k、m、g、t、p，分别代表 kibi-, mebi, gibi-, tebi- 以及pebibytes。
spark.yarn.am.cores                                       1                                                     client模式下，用来控制YARN AM的CPU core个数。cluster模式下，请使用 spark.driver.cores。
spark.yarn.am.waitTime                                    100s                                                  在cluster模式下，该属性表示YARN AM等待SparkContext初始化的时间。在client模式下，该属性表示YARN AM等待驱动器连接的时间。
spark.yarn.submit.file.replication                        默认的HDFS副本数(通常是3)                                HDFS文件副本数。包括Spark jar，app jar以及其他分布式缓存文件和存档。
spark.yarn.preserve.staging.files                         false                                                 设为true以保存stage相关文件（stage相关的jar包和缓存）到作业结束，而不是立即删除。
spark.yarn.scheduler.heartbeat.interval-ms                3000                                                  Spark AM发送给YARN资源管理器心跳的间隔（ms）。这个值最多不能超过YARN配置的超时间隔的一半。（yarn.am.liveness-monitor.expiry-interval-ms）
spark.yarn.scheduler.initial-allocation.interval          200ms                                                 Spark AM的初始带外心跳间隔（有待定的资源申请时）。其值不应该大于 spark.yarn.scheduler.heartbeat.interval-ms。该资源分配间隔会在每次带外心跳成功后但仍有待定资源申请时倍增，直至达到 spark.yarn.scheduler.heartbeat.interval-ms 所设定的值。
spark.yarn.max.executor.failures                          执行器个数*2且不小于3                                    Spark应用最大容忍执行器失败次数。
spark.yarn.historyServer.address                          (none)                                                Spark history server地址，如：host.com:18080 。这个地址不要包含协议头（http://）。默认不设置，因为history server是可选的。应用程序结束以后，YARN资源管理器web UI通过这个地址链接到Spark history server UI。对于这属性，可以使用YARN属性变量，且这些变量是Spark在运行时组装的。例如，如果Spark history server和YARN资源管理器（ResourceManager）部署在同一台机器上运行，那么这个属性可以设置为 ${hadoopconf-yarn.resourcemanager.hostname}:18080
spark.yarn.dist.archives                                  (none)                                                逗号分隔的文档列表，其指向的文档将被提取到每个执行器的工作目录下。
spark.yarn.dist.files                                     (none)                                                逗号分隔的文件列表，其指向的文件将被复制到每个执行器的工作目录下。
spark.executor.instances                                  2                                                     执行器个数。注意，这个属性和 spark.dynamicAllocation.enabled是不兼容的。如果同时设置了 spark.dynamicAllocation.enabled，那么动态分配将被关闭，并使用 spark.executor.instances 所设置的值。
spark.yarn.executor.memoryOverhead                        执行器内存 * 0.10或者 384MB中较大者                       每个执行器所分配的堆外内存（MB）总量。这些内存将被用于存储VM开销、字符串常量，以及其他原生开销等。这会使执行器所需内存增加（典型情况，增加6%~10%）
spark.yarn.driver.memoryOverhead                          驱动器内存 * 0.10或者 384MB中较大者                       每个驱动器所分配的堆外内存（MB）总量。这些内存将被用于存储VM开销、字符串常量，以及其他原生开销等。这会使执行器所需内存增加（典型情况，增加6%~10%）
spark.yarn.am.memoryOverhead                              Application Master 内存 * 0.10 或者 384MB中较大者        与 spark.yarn.driver.memoryOverhead 相同，只是仅用于YARN AM client模式下。
spark.yarn.am.port                                        (random)                                              YARN AM所监听的端口。在YARN client模式下，用于Spark驱动器(driver)和YARN AM通信。而在YARN cluster模式下，这个端口将被用于动态执行器特性，这个特性会处理调度器后台杀死执行器的请求。
spark.yarn.queue                                          default                                               Spark应用提交到哪个yarn队列。
spark.yarn.jars                                           (none)                                                Spark jar文件位置，如果需要覆盖默认位置，请设定这个值。默认的，Spark on YARN会使用本地的Spark jar包，但Spark jar包同样可以使用整个集群可读的HDFS文件位置。这使YARN可以在各节点上缓存Spark jar包，而不需要每次运行一个应用的时候都要分发。使用 hdfs:///some/path 来指定HDFS上jar包文件路径。
spark.yarn.access.namenodes                               (none)                                                逗号分隔的HDFS namenodes。例如 spark.yarn.access.namenodes=hdfs://nn1.com:8032,hdfs://nn2.com:8032。Spark应用必须有这些机器的访问权限，并且需要配置好 kerberos(可以在同一个域或者信任的域)。Spark需要每个namenode的安全token，以便访问集群中HDFS。
spark.yarn.appMasterEnv.[EnvironmentVariableName]         (none)                                                增加EnvironmentVariableName所指定的环境变量到YARN AM的进程中。用户可以指定多个环境变量。在cluster模式下，这个可以控制Spark驱动器的环境变量；而在client模式下，只控制执行器启动器的环境变量。
spark.yarn.containerLauncherMaxThreads                    25                                                    YARN AM 启动执行器的容器最多包含多少线程数。
spark.yarn.am.extraJavaOptions                            (none)                                                在client模式下，传给YARN AM 的JVM参数。在cluster模式下，请使用spark.driver.extraJavaOptions
spark.yarn.am.extraLibraryPath                            (none)                                                client模式下传给YARN AM 额外依赖库。
spark.yarn.maxAppAttempts                                 yarn.resourcemanager.am.max-attempts in YARN          提交应用最大尝试次数。不应大于YARN全局配置的最大尝试次数。
spark.yarn.am.attemptFailuresValidityInterval             (none)                                                定义AM失败跟踪校验间隔。AM运行了至少要运行这么多时间后，其失败计数才被重置。这个特性只有配置其值后才会生效，且只支持Hadoop-2.6+
spark.yarn.submit.waitAppCompletion                       true                                                  在YARN cluster模式下，控制是否客户端等到Spark应用结束后再退出。如果设为true，客户端进程将一直等待，并持续报告应用状态。否则，客户端会在提交完成后退出。
spark.yarn.am.nodeLabelExpression                         (none)                                                一个YARN节点标签表达式（node label expression），以此来限制AM可以被调度到哪些节点上执行。只有Hadoop 2.6+才能支持节点标签表达式，所以如果用其他版本运行，这个属性将被忽略。
spark.yarn.executor.nodeLabelExpression                   (none)                                                一个YARN节点标签表达式（node label expression），以此来限制执行器可以被调度到哪些节点上启动。只有Hadoop 2.6+才能支持节点标签表达式，所以如果在其他版本上运行时，这个属性将被忽略。
spark.yarn.tags                                           (none)                                                逗号分隔的字符串，传递YARN应用tags。其值将出现在YARN Application Reports中，可以用来过滤和查询YARN 应用。
spark.yarn.keytab                                         (none)                                                认证文件keytab的全路径。这个文件将被复制到访问Secure Distributed Cache的YARN 应用节点上，并且周期性的刷新登陆的ticket和代理token（本地模式下也能work）
spark.yarn.principal                                      (none)                                                登陆KDC的认证，secure HDFS需要（local模式下也能用）
spark.yarn.config.gatewayPath                             (none)                                                某些路径，可能在网关主机上能正常访问（Spark应用启动的地方），而在其他节点上的访问方式（路径）可能不同。对于这样的路径，需要本属性配合 spark.yarn.config.replacementPath组合使用，对于支持异构配置的集群，必须配置好这两个值，Spark才能正确地启动远程进程。replacement path 通常包含一些YARN导出的环境变量（因此，对Spark containers可见）。例如，如果网关节点上Hadoop库安装在 /disk1/hadoop，并且其导出环境变量为 HADOOP_HOME，就需要将 spark.yarn.config.gatewayPath 设置为 /disk1/hadoop 并将 replacement path设为 $HADOOP_HOME，这样才能在远程节点上以正确的环境变量启动进程。
spark.yarn.config.replacementPath                         (none)                                                见 spark.yarn.config.getewayPath
spark.yarn.security.tokens.${service}.enabled             true                                                  在启用安全设置的情况下，控制是否对non-HDFS服务，获取代理token。默认地，所有支持的服务，都启用；但你也可以在某些有冲突的情况下，对某些服务禁用。目前支持的服务有：hive，hbase
spark.yarn.rolledLog.includePattern                       (none)                                                Java正则表达式筛选与定义的包含模式和这些日志文件相匹配的日志文件将以滚动方式聚合。 这将与YARN的滚动日志聚合一起使用，以便在YARN端yarn.nodemanager.log-aggregation.roll-monitoring-interval-seconds中启用此功能，应在yarn-site.xml中配置。 此功能只能用于Hadoop 2.6.4+。 Spark log4j appender需要更改为使用FileAppender或另一个可以处理正在运行的文件的appender。 根据在log4j配置（如spark.log）中配置的文件名，用户应该设置正则表达式（spark *）以包含所有需要聚合的日志文件。
spark.yarn.rolledLog.excludePattern                       (none)                                                Java正则表达式来过滤与定义的排除模式相匹配的日志文件，这些日志文件不会以滚动方式聚合。 如果日志文件名称与包含和排除模式都匹配，则最终将排除该文件。
====================================================      ===============================================      =======================================

重要提示
------------------------------

* 对CPU资源的请求是否满足，取决于调度器如何配置和使用。
* cluster模式下，Spark执行器（executor）和驱动器（driver）的local目录都由YARN配置决定（yarn.nodemanager.local-dirs）；如果用户指定了spark.local.dir，这时候将被忽略。在client模式下，Spark执行器（executor）的local目录由YARN决定，而驱动器（driver）的local目录由spark.local.dir决定，因为这时候，驱动器不在YARN上运行。
* 选项参数 –files和 –archives中井号（#）用法类似于Hadoop。例如，你可以指定 –files localtest.txt#appSees.txt，这将会把localtest.txt文件上传到HDFS上，并重命名为 appSees.txt，而你的程序应用用 appSees.txt来引用这个文件。
* 当你在cluster模式下使用本地文件时，使用选项–jar 才能让SparkContext.addJar正常工作，而不必使用 HDFS，HTTP，HTTPS或者FTP上的文件。

在安全的集群中运行
------------------------------
在安全性方面，Kerberos用于安全的Hadoop集群，以对与服务和客户端关联的主体进行身份验证。这允许客户提出这些认证服务的请求;向经认证的校长授予权利的服务。

Hadoop服务发布hadoop令牌来授予对服务和数据的访问权限。客户必须首先获取他们将访问的服务的标记，并将其与在YARN集群中启动的应用程序一起传递。

对于Spark应用程序来与任何Hadoop文件系统（例如hdfs，webhdfs等），HBase和Hive进行交互，它必须使用启动应用程序的用户的Kerberos凭据来获取相关的令牌，也就是说，将成为Spark应用程序的启动。

这通常在启动时完成：在一个安全的集群中，Spark将自动获取集群的默认Hadoop文件系统的标记，并可能为HBase和Hive获取标记。

如果HBase位于类路径中，HBase配置声明应用程序是安全的（即，hbase-site.xml将hbase.security.authentication设置为kerberos），则将获得HBase令牌，并且spark.yarn.security.credentials.hbase.enabled没有设置为false。

同样，如果Hive位于类路径上，则会获得Hive标记，其配置包含“hive.metastore.uris”中的元数据存储的URI，并且spark.yarn.security.credentials.hive.enabled未设置为false。

如果应用程序需要与其他安全的Hadoop文件系统进行交互，则在启动时必须明确请求访问这些群集所需的令牌。这是通过将它们列在spark.yarn.access.hadoopFileSystems属性中完成的。

spark.yarn.access.hadoopFileSystems hdfs://ireland.example.org:8020/,webhdfs://frankfurt.example.org:50070/

Spark通过Java服务机制支持与其他安全感知服务的集成（请参阅java.util.ServiceLoader）。 要做到这一点，org.apache.spark.deploy.yarn.security.ServiceCredentialProvider的实现应该可用于Spark，方法是将它们的名称列在jar的META-INF / services目录下的相应文件中。 可以通过将spark.yarn.security.credentials {service} .enabled设置为false来禁用这些插件，其中{service}是凭证提供程序的名称。


配置外部的 Shuffle 服务
------------------------------

要在您的YARN集群中的每个NodeManager上启动Spark Shuffle服务，请按照以下说明进行操作：

1、使用YARN配置文件构建Spark。 如果您使用的是预打包发行版，请跳过此步骤。
2、找到spark- <version> -yarn-shuffle.jar。 这应该在$ SPARK_HOME / common / network-yarn / target / scala- <version>之下，如果你自己创建Spark，并且在使用分发的情况下使用yarn。
3、将此jar添加到群集中所有NodeManagers的类路径中。
4、在每个节点的yarn-site.xml中，将spark_shuffle添加到yarn.nodemanager.aux-services，然后将yarn.nodemanager.aux-services.spark_shuffle.class设置为org.apache.spark.network.yarn.YarnShuffleService。
5、通过在etc / hadoop / yarn-env.sh中设置YARN_HEAPSIZE（默认为1000）来增加NodeManager的堆大小，以避免在shuffle期间垃圾收集问题。
6、重新启动群集中的所有节点管理器。

在YARN上运行shuffle服务时，可以使用以下额外的配置选项：

==================================    =======   ==========================
属性名称                           	    默认值     	含义
==================================    =======   ==========================
spark.yarn.shuffle.stopOnFailure      false	    是否在Spark Shuffle Service初始化失败时停止NodeManager。 这可以防止在Spark Shuffle服务未运行的NodeManager上运行容器导致应用程序失败。
==================================    =======   ==========================


使用 Apache Oozie启动应用程序
------------------------------

Apache Oozie can launch Spark applications as part of a workflow. In a secure cluster, the launched application will need the relevant tokens to access the cluster’s services. If Spark is launched with a keytab, this is automatic. However, if Spark is to be launched without a keytab, the responsibility for setting up security must be handed over to Oozie.
The details of configuring Oozie for secure clusters and obtaining credentials for a job can be found on the Oozie web site in the “Authentication” section of the specific release’s documentation.
For Spark applications, the Oozie workflow must be set up for Oozie to request all tokens which the application needs, including:
* The YARN resource manager.
* The local HDFS filesystem.
* Any remote HDFS filesystems used as a source or destination of I/O.
* Hive —if used.
* HBase —if used.
* The YARN timeline server, if the application interacts with this.
To avoid Spark attempting —and then failing— to obtain Hive, HBase and remote HDFS tokens, the Spark configuration must be set to disable token collection for the services.

The Spark configuration must include the lines:
spark.yarn.security.tokens.hive.enabled false
spark.yarn.security.tokens.hbase.enabled false

The configuration option spark.yarn.access.namenodes must be unset.

Troubleshooting Kerberos
--------------------------------

Debugging Hadoop/Kerberos problems can be “difficult”. One useful technique is to enable extra logging of Kerberos operations in Hadoop by setting the HADOOP_JAAS_DEBUG environment variable.
bash export HADOOP_JAAS_DEBUG=true
The JDK classes can be configured to enable extra logging of their Kerberos and SPNEGO/REST authentication via the system propertiessun.security.krb5.debug and sun.security.spnego.debug=true
-Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true
All these options can be enabled in the Application Master:
spark.yarn.appMasterEnv.HADOOP_JAAS_DEBUG true spark.yarn.am.extraJavaOptions -Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true
Finally, if the log level for org.apache.spark.deploy.yarn.Client is set to DEBUG, the log will include a list of all tokens obtained, and their expiry details


使用 Spark History Server 替代 Spark Web UI
---------------------------------------------------------------------

It is possible to use the Spark History Server application page as the tracking URL for running applications when the application UI is disabled. This may be desirable on secure clusters, or to reduce the memory usage of the Spark driver. To set up tracking through the Spark History Server, do the following:

* On the application side, set spark.yarn.historyServer.allowTracking=true in Spark’s configuration. This will tell Spark to use the history server’s URL as the tracking URL if the application’s UI is disabled.
* On the Spark History Server, add org.apache.spark.deploy.yarn.YarnProxyRedirectFilter to the list of filters in the spark.ui.filters configuration.

Be aware that the history server information may not be up-to-date with the application’s state.