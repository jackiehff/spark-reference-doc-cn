监控和工具
============================

监控Spark应用有很多种方式：web UI，metrics 以及外部工具。

Web界面
----------------------

每个SparkContext都会启动一个web UI，其默认端口为4040，并且这个web UI能展示很多有用的Spark应用相关信息。包括：
* 一个stage和task的调度列表
* 一个关于RDD大小以及内存占用的概览
* 运行环境相关信息
* 运行中的执行器相关信息

你只需打开浏览器，输入 http://<driver-node>:4040 即可访问该web界面。如果有多个SparkContext在同时运行中，那么它们会从4040开始，按顺序依次绑定端口（4041,4042，等）。

注意，默认情况下，这些信息只有在Spark应用运行期内才可用。如果需要在Spark应用退出后仍然能在web UI上查看这些信息，则需要在应用启动前，将 spark.eventLog.enabled 设为 true。这项配置将会把Spark事件日志都记录到持久化存储中。

事后查看
---------------------

Spark独立部署时，其对应的集群管理器也有其对应的 web UI。如果Spark应用将其运行期事件日志保留下来了，那么独立部署集群管理器对应的web UI将会根据这些日志自动展示已经结束的Spark应用。

如果Spark是运行于Mesos或者YARN上的话，那么你需要开启Spark的history server，开启event log。开启history server需要如下指令：

./sbin/start-history-server.sh

如果使用file-system provider class（参考下面的 spark.history.provider），那么日志目录将会基于 spark.history.fs.logDirectory 配置项，并且在表达Spark应用的事件日志路径时，应该带上子目录。history server对应的web界面默认在这里 http://<server-url>:18080。同时，history server有一些可用的配置如下：


环境变量
---------------------

========================      ========================
环境变量                        含义
========================      ========================
SPARK_DAEMON_MEMORY            history server分配多少内存（默认: 1g）
SPARK_DAEMON_JAVA_OPTS         history server的 JVM参数（默认：none）
SPARK_DAEMON_CLASSPATH	       Classpath for the history server (default: none).
SPARK_PUBLIC_DNS               history server的外部访问地址，如果不配置，那么history server有可能会绑定server的内部地址，这可能会导致外部不能访问（默认：none）
SPARK_HISTORY_OPTS             history server配置项（默认：none）：spark.history.*
========================      ========================

Spark 配置选项
---------------------

===================================      ===================================================      ===================================================
属性名称                                   默认值                                                    含义
===================================      ===================================================      ===================================================
spark.history.provider                    org.apache.spark.deploy.history.FsHistoryProvider        Spark应用历史后台实现的类名。目前可用的只有spark自带的一个实现，支持在本地文件系统中查询应用日志。
spark.history.fs.logDirectory             file:/tmp/spark-events                                   history server加载应用日志的目录
spark.history.fs.update.interval          10s                                                      history server更新信息的时间间隔。每次更新将会检查磁盘上的日志是否有更新。
spark.history.retainedApplications        50                                                       UI上保留的spark应用历史个数。超出的将按时间排序，删除最老的。
spark.history.ui.maxApplications	        Int.MaxValue	                                           The number of applications to display on the history summary page. Application UIs are still available by accessing their URLs directly even if they are not displayed on the history summary page.
spark.history.ui.port                     18080                                                    history server绑定的端口
spark.history.kerberos.enabled            false                                                    history server是否启用kerberos验证登陆。如果history server需要访问一个需要安全保证的hadoop集群，则需要开启这个功能。该配置设为true以后，需要同时配置 spark.history.kerberos.principal 和 spark.history.kerberos.keytab
spark.history.kerberos.principal          (none)                                                   登陆history server的kerberos 主体名称
spark.history.kerberos.keytab             (none)                                                   history server对应的kerberos keytab文件路径
spark.history.ui.acls.enable              false                                                    指定是否启用ACL以控制用户访问验证。如果启用，那么不管某个应用是否设置了 spark.ui.acls.enabled，访问控制都将检查用户是否有权限。Spark应用的owner始终有查看的权限，而其他用户则需要通过 spark.ui.view.acls 配置其访问权限。如果禁用，则不会检查访问权限。
spark.history.ui.admin.acls	              empty	                                                   Comma separated list of users/administrators that have view access to all the Spark applications in history server. By default only the users permitted to view the application at run-time could access the related application history, with this, configured users/administrators could also have the permission to access it. Putting a "*" in the list means any user can have the privilege of admin.
spark.history.ui.admin.acls.groups	      empty	                                                   Comma separated list of groups that have view access to all the Spark applications in history server. By default only the groups permitted to view the application at run-time could access the related application history, with this, configured groups could also have the permission to access it. Putting a "*" in the list means any group can have the privilege of admin.
spark.history.fs.cleaner.enabled          false                                                    指定history server是否周期性清理磁盘上的event log
spark.history.fs.cleaner.interval         1d                                                       history server清理磁盘文件的时间间隔。只会清理比 spark.history.fs.cleaner.maxAge 时间长的磁盘文件。
spark.history.fs.cleaner.maxAge           7d                                                       如果启用了history server周期性清理，比这个时间长的Spark作业历史文件将会被清理掉
spark.history.fs.numReplayThreads	        25% of available cores	                                 Number of threads that will be used by history server to process event logs.
===================================      ===================================================      ===================================================


注意，所有web界面上的 table 都可以点击其表头来排序，这样可以帮助用户做一些简单分析，如：发现跑的最慢的任务、数据倾斜等。

注意history server 只展示已经结束的Spark作业。一种通知Spark作业结束的方法是，显式地关闭SparkContext（通过调用 sc.stop()，或者在 使用 SparkContext() 处理其 setup 和 tear down 事件（适用于python），然后作业历史就会出现在web UI上了。


REST API
-------------------------

度量信息除了可以在UI上查看之外，还可以以JSON格式访问。这能使开发人员很容易构建新的Spark可视化和监控工具。JSON格式的度量信息对运行中的Spark应用和history server中的历史作业均有效。其访问端点挂载在 /api/v1 路径下。例如，对于history server，一般你可以通过 http://<server-url>:18080/api/v1 来访问，而对于运行中的应用，可以通过 http://localhost:4040/api/v1 来访问。

=========================================================================      =====================================================
端点                                                                                      含义
=========================================================================      =====================================================
/applications                                                                  所有应用的列表
/applications/[app-id]/jobs                                                    给定应用的全部作业列表
/applications/[app-id]/jobs/[job-id]                                           给定作业的细节
/applications/[app-id]/stages                                                  给定应用的stage列表
/applications/[app-id]/stages/[stage-id]                                       给定stage的所有attempt列表
/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]                    给定attempt的详细信息
/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskSummary        指定attempt对应的所有task的概要度量信息
/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskList           指定的attempt的所有task的列表
/applications/[app-id]/executors                                               给定应用的所有执行器
/applications/[app-id]/storage/rdd                                             给定应用的已保存的RDD列表
/applications/[app-id]/storage/rdd/[rdd-id]                                    给定的RDD的存储详细信息
/applications/[app-id]/logs                                                    将给定应用的所有attempt对应的event log以zip格式打包下载
/applications/[app-id]/[attempt-id]/logs                                       将给定attempt的所有attempt对应的event log以zip格式打包下载
=========================================================================      =====================================================

如果在YARN上运行，每个应用都由多个attempts，所以 [app-id] 实际上是 [app-id]/[attempt-id]。


API Versioning Policy
-----------------------------

这些API端点都有版本号，所以基于这些API开发程序就比较容易。Spark将保证：

* 端点一旦添加进来，就不会删除
* 某个端点支持的字段永不删除
* 未来可能会增加新的端点
* 已有端点可能会增加新的字段
* 未来可能会增加新的API版本，但会使用不同的端点（如：api/v2 ）。但新版本不保证向后兼容。
* API版本可能会整个丢弃掉，但在丢弃前，一定会和新版本API共存至少一个小版本。

注意，在UI上检查运行中的应用时，虽然每次只能查看一个应用， 但applicatoins/[app-id] 这部分路径仍然是必须的。例如，你需要查看运行中应用的作业列表时，你需要输入 http://localhost:4040/api/v1/applications/[app-id]/jobs。虽然麻烦点，但这能保证两种模式下访问路径的一致性。


度量
------------------------

Spark的度量子系统是可配置的，其功能是基于Coda Hale Metrics Library开发的。这套度量子系统允许用户以多种形式的汇报槽（sink）汇报Spark度量信息，包括：HTTP、JMX和CSV文件等。其对应的配置文件路径为：${SPARK_HOME}/conf/metrics.properties。当然，你可以通过spark.metrics.conf 这个Spark属性来自定义配置文件路径（详见configuration property）。Spark的各个组件都有其对应的度量实例，且这些度量实例之间是解耦的。这些度量实例中，你都可以配置一系列不同的汇报槽来汇报度量信息。以下是目前支持的度量实例：

* master: 对应Spark独立部署时的master进程。
* applications: master进程中的一个组件，专门汇报各个Spark应用的度量信息。
* worker: 对应Spark独立部署时的worker进程。
* executor: 对应Spark执行器。
* driver: 对应Spark驱动器进程（即创建SparkContext对象的进程）。
每个度量实例可以汇报给0~n个槽。以下是目前 org.apache.spark.metrics.sink 包中包含的几种汇报槽（sink）：
* ConsoleSink:将度量信息打印到控制台。
* CSVSink: 以特定的间隔，将度量信息输出到CSV文件。
* JmxSink: 将度量信息注册到JMX控制台。
* MetricsServlet: 在已有的Spark UI中增加一个servlet，对外提供JSON格式的度量数据。
* GraphiteSink: 将度量数据发到Graphite 节点。
* Slf4jSink: 将度量数据发送给slf4j 打成日志。
Spark同样也支持Ganglia，但因为license限制的原因没有包含在默认的发布包中：
* GangliaSink: 将度量信息发送给一个Ganglia节点或者多播组。

如果需要支持GangliaSink的话，你需要自定义Spark构建包。注意，如果你包含了GangliaSink代码包的话，就必须同时将 LGPL-licensed 协议包含进你的Spark包中。对于sbt用户，只需要在编译打包前设置好环境变量：SPARK_GANGLIA_LGPL即可。对于maven用户，启用 -Pspark-ganglia-lgpl 即可。另外，除了修改集群的Spark之外，用户程序还需要链接 spark-ganglia-lgpl 工件。

度量系统配置文件语法可以参考这个配置文件示例：${SPARK_HOME}/conf/metrics.properties.template


高级工具
------------------------

以下是几个可以用以分析Spark性能的外部工具：

* 集群整体监控工具，如：Ganglia，可以提供集群整体的使用率和资源瓶颈视图。比如，Ganglia的仪表盘可以迅速揭示出整个集群的工作负载是否达到磁盘、网络或CPU限制。
* 操作系统分析工具，如：dstat, iostat, 以及 iotop ，可以提供单个节点上细粒度的分析剖面。
* JVM工具可以帮助你分析JVM虚拟机，如：jstack可以提供调用栈信息，jmap可以转储堆内存数据，jstat可以汇报时序统计信息，jconsole可以直观的探索各种JVM属性，这对于熟悉JVM内部机制非常有用。
