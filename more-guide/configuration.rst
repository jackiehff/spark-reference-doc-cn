##############################
Spark 配置
##############################

Spark 提供以下三种方式修改配置：
* Spark properties （Spark属性）可以控制绝大多数应用程序参数，而且既可以通过 SparkConf 对象来设置，也可以通过Java系统属性来设置。
* Environment variables （环境变量）可以指定一些各个机器相关的设置，如IP地址，其设置方法是写在每台机器上的conf/spark-env.sh中。
* Logging （日志）可以通过log4j.properties配置日志。


******************
Spark属性
******************

Spark属性可以控制大多数的应用程序设置，并且每个应用的设定都是分开的。这些属性可以用SparkConf 对象直接设定。SparkConf为一些常用的属性定制了专用方法（如，master URL和application name），其他属性都可以用键值对做参数，调用set()方法来设置。例如，我们可以初始化一个包含2个本地线程的Spark应用，代码如下：

注意，local[2]代表2个本地线程 – 这是最小的并发方式，可以帮助我们发现一些只有在分布式上下文才能复现的bug。

.. code-block:: Scala

  val conf = new SparkConf()
              .setMaster("local[2]")
              .setAppName("CountingSheep")
  val sc = new SparkContext(conf)


注意，本地模式下，我们可以使用n个线程（n >= 1）。而且在像Spark Streaming这样的场景下，我们可能需要多个线程来防止类似线程饿死这样的问题。

配置时间段的属性应该写明时间单位，如下格式都是可接受的：

25ms (milliseconds)
5s (seconds)
10m or 10min (minutes)
3h (hours)
5d (days)
1y (years)

配置大小的属性也应该写明单位，如下格式都是可接受的：

1b (bytes)
1k or 1kb (kibibytes = 1024 bytes)
1m or 1mb (mebibytes = 1024 kibibytes)
1g or 1gb (gibibytes = 1024 mebibytes)
1t or 1tb (tebibytes = 1024 gibibytes)
1p or 1pb (pebibytes = 1024 tebibytes)


动态加载 Spark 属性
======================

在某些场景下，你可能需要避免将属性值写死在 SparkConf 中。例如，你可能希望在同一个应用上使用不同的master或不同的内存总量。Spark允许你简单地创建一个空的SparkConf对象：

.. code-block:: Scala

  val sc = new SparkContext(new SparkConf())


然后在运行时设置这些属性：

.. code-block:: Shell

  ./bin/spark-submit --name "My app" --master local[4] --conf spark.eventLog.enabled=false
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar


Spark shell和spark-submit工具支持两种动态加载配置的方法。第一种，通过命令行选项，如：上面提到的–master（设置master URL）。spark-submit可以在启动Spark应用时，通过–conf标志接受任何属性配置，同时有一些特殊配置参数同样可用（如，–master）。运行./bin/spark-submit –help可以展示这些选项的完整列表。

同时，bin/spark-submit 也支持从conf/spark-defaults.conf 中读取配置选项，在该文件中每行是一个键值对，并用空格分隔，如下：

.. code-block:: Shell

  spark.master            spark://5.6.7.8:7077
  spark.executor.memory  4g
  spark.eventLog.enabled  true
  spark.serializer        org.apache.spark.serializer.KryoSerializer


这些通过参数或者属性配置文件传递的属性，最终都会在SparkConf 中合并。其优先级是：首先是SparkConf代码中写的属性值，其次是spark-submit或spark-shell的标志参数，最后是spark-defaults.conf文件中的属性。

有一些配置项被重命名过，这种情形下，老的名字仍然是可以接受的，只是优先级比新名字优先级低。


查看Spark属性
===================

每个SparkContext都有其对应的Spark UI，所以Spark应用程序都能通过Spark UI查看其属性。默认你可以在这里看到：http://<driver>:4040，页面上的”Environment“ tab页可以查看Spark属性。如果你真的想确认一下属性设置是否正确的话，这个功能就非常有用了。注意，只有显式地通过SparkConf对象、在命令行参数、或者spark-defaults.conf设置的参数才会出现在页面上。其他属性，你可以认为都是默认值。

可用的属性
===================

绝大多数属性都有合理的默认值。这里是部分常用的选项：

应用属性
-------------------

============================      ==========      ===================
属性名称                            默认值           含义
============================      ==========      ===================
spark.app.name                    (none)           Spark应用的名字。会在SparkUI和日志中出现。
spark.driver.cores                 1               在cluster模式下，用几个core运行驱动器（driver）进程。
spark.driver.maxResultSize         1g              Spark action算子返回的结果最大多大。至少要1M，可以设为0表示无限制。如果结果超过这一大小，Spark作业（job）会直接中断退出。但是，设得过高有可能导致驱动器OOM（out-of-memory）（取决于spark.driver.memory设置，以及驱动器JVM的内存限制）。设一个合理的值，以避免驱动器OOM。
spark.driver.memory                1g              驱动器进程可以用的内存总量（如：1g，2g）。注意，在客户端模式下，这个配置不能在SparkConf中直接设置（因为驱动器JVM都启动完了呀！）。驱动器客户端模式下，必须要在命令行里用 –driver-memory 或者在默认属性配置文件里设置。
spark.executor.memory              1g              单个执行器（executor）使用的内存总量（如，2g，8g）
spark.extraListeners              (none)           逗号分隔的SparkListener子类的类名列表；初始化SparkContext时，这些类的实例会被创建出来，并且注册到Spark的监听总线上。如果这些类有一个接受SparkConf作为唯一参数的构造函数，那么这个构造函数会被优先调用；否则，就调用无参数的默认构造函数。如果没有构造函数，SparkContext创建的时候会抛异常。
spark.local.dir                   /tmp             Spark的”草稿“目录，包括map输出的临时文件，或者RDD存在磁盘上的数据。这个目录最好在本地文件系统中，这样读写速度快。这个配置可以接受一个逗号分隔的列表，通常用这种方式将文件IO分散不同的磁盘上去。注意：Spark-1.0及以后版本中，这个属性会被集群管理器所提供的环境变量覆盖：SPARK_LOCAL_DIRS（独立部署或Mesos）或者 LOCAL_DIRS（YARN）。
spark.logConf                     false            SparkContext启动时是否把生效的 SparkConf 属性以INFO日志打印到日志里
spark.master                      (none)           集群管理器URL。参考allowed master URL’s.
spark.submit.deployMode	          (none)	         The deploy mode of Spark driver program, either "client" or "cluster", Which means to launch driver program locally ("client") or remotely ("cluster") on one of the nodes inside the cluster.
spark.log.callerContext	          (none)	         Application information that will be written into Yarn RM log/HDFS audit log when running on Yarn/HDFS. Its length depends on the Hadoop configuration hadoop.caller.context.max.size. It should be concise, and typically can have up to 50 characters.
spark.driver.supervise	          false	           If true, restarts the driver automatically if it fails with a non-zero exit status. Only has effect in Spark standalone mode or Mesos cluster deploy mode.
============================      ==========      ===================

除了这些以外，以下还有很多可用的参数配置，在某些特定情形下，可能会用到：

运行时环境
-----------------

===============================================      ====================      ====================
属性名                                                默认值                     含义
===============================================      ====================      ====================
spark.driver.extraClassPath                          (none)                    额外的classpath，将插入到到驱动器的classpath开头。注意：驱动器如果运行客户端模式下，这个配置不能通过SparkConf 在程序里配置，因为这时候程序已经启动呀！而是应该用命令行参数（–driver-class-path）或者在 conf/spark-defaults.conf 配置。
spark.driver.extraJavaOptions                        (none)                    驱动器额外的JVM选项。如：GC设置或其他日志参数。注意：驱动器如果运行客户端模式下，这个配置不能通过SparkConf在程序里配置，因为这时候程序已经启动呀！而是应该用命令行参数（–driver-java-options）或者conf/spark-defaults.conf 配置。
spark.driver.extraLibraryPath                        (none)                    启动驱动器JVM时候指定的依赖库路径。注意：驱动器如果运行客户端模式下，这个配置不能通过SparkConf在程序里配置，因为这时候程序已经启动呀！而是应该用命令行参数（–driver-library-path）或者conf/spark-defaults.conf 配置。
spark.driver.userClassPathFirst                      false                     (试验性的：即未来不一定会支持该配置)驱动器是否首选使用用户指定的jars，而不是spark自身的。这个特性可以用来处理用户依赖和spark本身依赖项之间的冲突。目前还是试验性的，并且只能用在集群模式下。
spark.executor.extraClassPath                        (none)                    添加到执行器（executor）classpath开头的classpath。主要为了向后兼容老的spark版本，不推荐使用。
spark.executor.extraJavaOptions                      (none)                    传给执行器的额外JVM参数。如：GC设置或其他日志设置等。注意，不能用这个来设置Spark属性或者堆内存大小。Spark属性应该用SparkConf对象，或者spark-defaults.conf文件（会在spark-submit脚本中使用）来配置。执行器堆内存大小应该用 spark.executor.memory配置。
spark.executor.extraLibraryPath                      (none)                    启动执行器JVM时使用的额外依赖库路径。
spark.executor.logs.rolling.maxRetainedFiles         (none)                    Sets the number of latest rolling log files that are going to be retained by the system. Older log files will be deleted. Disabled by default.设置日志文件最大保留个数。老日志文件将被干掉。默认禁用的。
spark.executor.logs.rolling.enableCompression	       false	                   Enable executor log compression. If it is enabled, the rolled executor logs will be compressed. Disabled by default.
spark.executor.logs.rolling.maxSize                  (none)                    设置执行器日志文件大小上限。默认禁用的。需要自动删日志请参考 spark.executor.logs.rolling.maxRetainedFiles.
spark.executor.logs.rolling.strategy                 (none)                    执行器日志滚动策略。默认可接受的值有”time”（基于时间滚动） 或者 “size”（基于文件大小滚动）。time：将使用 spark.executor.logs.rolling.time.interval设置滚动时间间隔size：将使用 spark.executor.logs.rolling.size.maxBytes设置文件大小上限
spark.executor.logs.rolling.time.interval            daily                     设置执行器日志滚动时间间隔。日志滚动默认是禁用的。可用的值有 “daily”, “hourly”, “minutely”，也可设为数字（则单位为秒）。关于日志自动清理，请参考 spark.executor.logs.rolling.maxRetainedFiles
spark.executor.userClassPathFirst                    false                     (试验性的）与 spark.driver.userClassPathFirst类似，只不过这个参数将应用于执行器
spark.executorEnv.[EnvironmentVariableName]          (none)                    向执行器进程增加名为EnvironmentVariableName的环境变量。用户可以指定多个来设置不同的环境变量。
spark.redaction.regex                                (?i)secret|password	     Regex to decide which Spark configuration properties and environment variables in driver and executor environments contain sensitive information. When this regex matches a property key or value, the value is redacted from the environment UI and various logs like YARN and event logs.
spark.python.profile                                 false                     对Python worker启用性能分析，性能分析结果会在sc.show_profile()中，或者在驱动器退出前展示。也可以用sc.dump_profiles(path)输出到磁盘上。如果部分分析结果被手动展示过，那么驱动器退出前就不再自动展示了。默认会使用pyspark.profiler.BasicProfiler，也可以自己传一个profiler 类参数给SparkContext构造函数。
spark.python.profile.dump                            (none)                    这个目录是用来在驱动器退出前，dump性能分析结果。性能分析结果会按RDD分别dump。同时可以使用ptats.Stats()来装载。如果制定了这个，那么分析结果就不再自动展示。
spark.python.worker.memory                           512m                      聚合时每个python worker使用的内存总量，和JVM的内存字符串格式相同（如，512m，2g）。如果聚合时使用的内存超过这个量，就将数据溢出到磁盘上。
spark.python.worker.reuse                            true                      是否复用Python worker。如果是，则每个任务会启动固定数量的Python worker，并且不需要fork() python进程。如果需要广播的数据量很大，设为true能大大减少广播数据量，因为需要广播的进程数减少了。
spark.files		                                                                 Comma-separated list of files to be placed in the working directory of each executor.
spark.submit.pyFiles		                                                       Comma-separated list of .zip, .egg, or .py files to place on the PYTHONPATH for Python apps.
spark.jars		                                                                 Comma-separated list of local jars to include on the driver and executor classpaths.
spark.jars.packages		                                                         Comma-separated list of Maven coordinates of jars to include on the driver and executor classpaths. The coordinates should be groupId:artifactId:version. If spark.jars.ivySettings is given artifacts will be resolved according to the configuration in the file, otherwise artifacts will be searched for in the local maven repo, then maven central and finally any additional remote repositories given by the command-line option --repositories. For more details, see Advanced Dependency Management.
spark.jars.excludes		                                                         Comma-separated list of groupId:artifactId, to exclude while resolving the dependencies provided in spark.jars.packages to avoid dependency conflicts.
spark.jars.ivy		                                                             Path to specify the Ivy user directory, used for the local Ivy cache and package files from spark.jars.packages. This will override the Ivy property ivy.default.ivy.user.dir which defaults to ~/.ivy2.
spark.jars.ivySettings		                                                     Path to an Ivy settings file to customize resolution of jars specified using spark.jars.packages instead of the built-in defaults, such as maven central. Additional repositories given by the command-line option --repositories will also be included. Useful for allowing Spark to resolve artifacts from behind a firewall e.g. via an in-house artifact server like Artifactory. Details on the settings file format can be found at http://ant.apache.org/ivy/history/latest-milestone/settings.html
spark.pyspark.driver.python		                                                 Python binary executable to use for PySpark in driver. (default is spark.pyspark.python)
spark.pyspark.python		                                                       Python binary executable to use for PySpark in both driver and executors.
===============================================      ====================      ====================


混洗行为
---------------

==========================================      ====================      ======================
属性名                                           默认值                     含义
==========================================      ====================      ======================
spark.reducer.maxSizeInFlight                   48m                       map任务输出同时reduce任务获取的最大内存占用量。每个输出需要创建buffer来接收，对于每个reduce任务来说，有一个固定的内存开销上限，所以最好别设太大，除非你内存非常大。
spark.reducer.maxReqsInFlight	                  Int.MaxValue	            This configuration limits the number of remote requests to fetch blocks at any given point. When the number of hosts in the cluster increase, it might lead to very large number of in-bound connections to one or more nodes, causing the workers to fail under load. By allowing it to limit the number of fetch requests, this scenario can be mitigated.
spark.reducer.maxBlocksInFlightPerAddress	      Int.MaxValue	            This configuration limits the number of remote blocks being fetched per reduce task from a given host port. When a large number of blocks are being requested from a given address in a single fetch or simultaneously, this could crash the serving executor or Node Manager. This is especially useful to reduce the load on the Node Manager when external shuffle is enabled. You can mitigate this issue by setting it to a lower value.
spark.reducer.maxReqSizeShuffleToMem	          Long.MaxValue	            The blocks of a shuffle request will be fetched to disk when size of the request is above this threshold. This is to avoid a giant request takes too much memory. We can enable this config by setting a specific value(e.g. 200m). Note that this config can be enabled only when the shuffle shuffle service is newer than Spark-2.2 or the shuffle service is disabled.
spark.shuffle.compress                          true                      是否压缩map任务的输出文件。通常来说，压缩是个好主意。使用的压缩算法取决于 spark.io.compression.codec
spark.shuffle.file.buffer                       32k                       每个混洗输出流的内存buffer大小。这个buffer能减少混洗文件的创建和磁盘寻址。
spark.shuffle.io.maxRetries                     3                         (仅对netty）如果IO相关异常发生，重试次数（如果设为非0的话）。重试能是大量数据的混洗操作更加稳定，因为重试可以有效应对长GC暂停或者网络闪断。
spark.shuffle.io.numConnectionsPerPeer          1                         (仅netty）主机之间的连接是复用的，这样可以减少大集群中重复建立连接的次数。然而，有些集群是机器少，磁盘多，这种集群可以考虑增加这个参数值，以便充分利用所有磁盘并发性能。
spark.shuffle.io.preferDirectBufs               true                      (仅netty）堆外缓存可以有效减少垃圾回收和缓存复制。对于堆外内存紧张的用户来说，可以考虑禁用这个选项，以迫使netty所有内存都分配在堆上。
spark.shuffle.io.retryWait                      5s                        (仅netty）混洗重试获取数据的间隔时间。默认最大重试延迟是15秒，设置这个参数后，将变成maxRetries* retryWait。
spark.shuffle.service.enabled                   false                     启用外部混洗服务。启用外部混洗服务后，执行器生成的混洗中间文件就由该服务保留，这样执行器就可以安全的退出了。如果 spark.dynamicAllocation.enabled启用了，那么这个参数也必须启用，这样动态分配才能有外部混洗服务可用。更多请参考：dynamic allocation configuration and setup documentation
spark.shuffle.service.port                      7337                      外部混洗服务对应端口
spark.shuffle.service.index.cache.entries	      1024	                    Max number of entries to keep in the index cache of the shuffle service.
spark.shuffle.sort.bypassMergeThreshold         200                       (高级）在基于排序（sort）的混洗管理器中，如果没有map端聚合的话，就会最多存在这么多个reduce分区。
spark.shuffle.spill.compress                    true                      是否在混洗阶段压缩溢出到磁盘的数据。压缩算法取决于spark.io.compression.codec
spark.shuffle.accurateBlockThreshold	          100 * 1024 * 1024	        When we compress the size of shuffle blocks in HighlyCompressedMapStatus, we will record the size accurately if it's above this config. This helps to prevent OOM by avoiding underestimating shuffle block size when fetch shuffle blocks.
spark.io.encryption.enabled	                    false	                    Enable IO encryption. Currently supported by all modes except Mesos. It's recommended that RPC encryption be enabled when using this feature.
spark.io.encryption.keySizeBits	                128	IO                    encryption key size in bits. Supported values are 128, 192 and 256.
spark.io.encryption.keygen.algorithm	          HmacSHA1	                The algorithm to use when generating the IO encryption key. The supported algorithms are described in the KeyGenerator section of the Java Cryptography Architecture Standard Algorithm Name Documentation.
==========================================      ====================      ======================


Spark UI
------------------

====================================      =========================      ===============
属性名                                     默认值                          含义
====================================      =========================      ===============
spark.eventLog.compress                   false                          是否压缩事件日志（当然spark.eventLog.enabled必须开启）
spark.eventLog.dir                        file:///tmp/spark-events       Spark events日志的基础目录（当然spark.eventLog.enabled必须开启）。在这个目录中，spark会给每个应用创建一个单独的子目录，然后把应用的events log打到子目录里。用户可以设置一个统一的位置（比如一个HDFS目录），这样history server就可以从这里读取历史文件。
spark.eventLog.enabled                    false                          是否启用Spark事件日志。如果Spark应用结束后，仍需要在SparkUI上查看其状态，必须启用这个。
spark.ui.enabled	                        true	                         Whether to run the web UI for the Spark application.
spark.ui.killEnabled                      true                           允许从SparkUI上杀掉stage以及对应的作业（job）
spark.ui.port                             4040                           SparkUI端口，展示应用程序运行状态。
spark.ui.retainedJobs                     1000                           SparkUI和status API最多保留多少个spark作业的数据（当然是在垃圾回收之前）
spark.ui.retainedStages                   1000                           SparkUI和status API最多保留多少个spark步骤（stage）的数据（当然是在垃圾回收之前）
spark.ui.retainedTasks	                  100000	                       How many tasks the Spark UI and status APIs remember before garbage collecting. This is a target maximum, and fewer elements may be retained in some circumstances.
spark.ui.reverseProxy	                    false	                         Enable running Spark Master as reverse proxy for worker and application UIs. In this mode, Spark master will reverse proxy the worker and application UIs to enable access without requiring direct access to their hosts. Use it with caution, as worker and application UI will not be accessible directly, you will only be able to access them through spark master/proxy public URL. This setting affects all the workers and application UIs running in the cluster and must be set on all the workers, drivers and masters.
spark.ui.reverseProxyUrl		                                             This is the URL where your proxy is running. This URL is for proxy which is running in front of Spark Master. This is useful when running proxy for authentication e.g. OAuth proxy. Make sure this is a complete URL including scheme (http/https) and port to reach your proxy.
spark.ui.showConsoleProgress	            true	                         Show the progress bar in the console. The progress bar shows the progress of stages that run for longer than 500ms. If multiple stages run at the same time, multiple progress bars will be displayed on the same line.
spark.worker.ui.retainedExecutors         1000                           SparkUI和status API最多保留多少个已结束的执行器（executor）的数据（当然是在垃圾回收之前）
spark.worker.ui.retainedDrivers           1000                           SparkUI和status API最多保留多少个已结束的驱动器（driver）的数据（当然是在垃圾回收之前）
spark.sql.ui.retainedExecutions           1000                           SparkUI和status API最多保留多少个已结束的执行计划（execution）的数据（当然是在垃圾回收之前）
spark.streaming.ui.retainedBatches        1000                           SparkUI和status API最多保留多少个已结束的批量（batch）的数据（当然是在垃圾回收之前）
spark.ui.retainedDeadExecutors	          100	                           How many dead executors the Spark UI and status APIs remember before garbage collecting.
====================================      =========================      ===============


压缩和序列化
------------------

=======================================      ===========================================      ====================================
属性名                                        默认值                                            含义
=======================================      ===========================================      ====================================
spark.broadcast.compress                     true                                             是否在广播变量前使用压缩。通常是个好主意。
spark.io.compression.codec                   lz4                                              内部数据使用的压缩算法，如：RDD分区、广播变量、混洗输出。Spark提供了3中算法：lz4，lzf，snappy。你也可以使用全名来指定压缩算法：org.apache.spark.io.LZ4CompressionCodec,org.apache.spark.io.LZFCompressionCodec,org.apache.spark.io.SnappyCompressionCodec.
spark.io.compression.lz4.blockSize           32k                                              LZ4算法使用的块大小。当然你需要先使用LZ4压缩。减少块大小可以减少混洗时LZ4算法占用的内存量。
spark.io.compression.snappy.blockSize        32k                                              Snappy算法使用的块大小（先得使用Snappy算法）。减少块大小可以减少混洗时Snappy算法占用的内存量。
spark.kryo.classesToRegister                 (none)                                           如果你使用Kryo序列化，最好指定这个以提高性能（tuning guide）。本参数接受一个逗号分隔的类名列表，这些类都会注册为Kryo可序列化类型。
spark.kryo.referenceTracking                 true                                             (false when using Spark SQL Thrift Server)   是否跟踪同一对象在Kryo序列化的引用。如果你的对象图中有循环护着包含统一对象的多份拷贝，那么最好启用这个。如果没有这种情况，那就禁用以提高性能。
spark.kryo.registrationRequired              false                                            Kryo序列化时，是否必须事先注册。如果设为true，那么Kryo遇到没有注册过的类型，就会抛异常。如果设为false（默认）Kryo会序列化未注册类型的对象，但会有比较明显的性能影响，所以启用这个选项，可以强制必须在序列化前，注册可序列化类型。
spark.kryo.registrator                       (none)                                           如果你使用Kryo序列化，用这个class来注册你的自定义类型。如果你需要自定义注册方式，这个参数很有用。否则，使用 spark.kryo.classesRegister更简单。要设置这个参数，需要用KryoRegistrator的子类。详见：tuning guide 。
spark.kryo.unsafe	                           false	                                          Whether to use unsafe based Kryo serializer. Can be substantially faster by using Unsafe Based IO.
spark.kryoserializer.buffer.max              64m                                              最大允许的Kryo序列化buffer。必须必你所需要序列化的对象要大。如果你在Kryo中看到”buffer limit exceeded”这个异常，你就得增加这个值了。
spark.kryoserializer.buffer                  64k                                              Kryo序列化的初始buffer大小。注意，每台worker上对应每个core会有一个buffer。buffer最大增长到 spark.kryoserializer.buffer.max
spark.rdd.compress                           false                                            是否压缩序列化后RDD的分区（如：StorageLevel.MEMORY_ONLY_SER）。能节省大量空间，但多消耗一些CPU。
spark.serializer                             org.apache.spark.serializer.JavaSerializer       (org.apache.spark.serializer.KryoSerializer when using Spark SQL Thrift Server)用于序列化对象的类，序列化后的数据将通过网络传输，或从缓存中反序列化回来。默认的Java序列化使用java的Serializable接口，但速度较慢，所以我们建议使用usingorg.apache.spark.serializer.KryoSerializer and configuring Kryo serialization如果速度需要保证的话。当然你可以自定义一个序列化器，通过继承org.apache.spark.Serializer.
spark.serializer.objectStreamReset           100                                              如果使用org.apache.spark.serializer.JavaSerializer做序列化器，序列化器缓存这些对象，以避免输出多余数据，然而，这个会打断垃圾回收。通过调用reset来flush序列化器，从而使老对象被回收。要禁用这一周期性reset，需要把这个参数设为-1，。默认情况下，序列化器会每过100个对象，被reset一次。
=======================================      ===========================================      ====================================


内存管理
------------------

=====================================      ========      ===============================
属性名                                      默认值         含义
=====================================      ========      ===============================
spark.memory.fraction                      0.75          堆内存中用于执行、混洗和存储（缓存）的比例。这个值越低，则执行中溢出到磁盘越频繁，同时缓存被逐出内存也更频繁。这个配置的目的，是为了留出用户自定义数据结构、内部元数据使用的内存。推荐使用默认值。请参考this description.
spark.memory.storageFraction               0.5           不会被逐出内存的总量，表示一个相对于 spark.memory.fraction的比例。这个越高，那么执行混洗等操作用的内存就越少，从而溢出磁盘就越频繁。推荐使用默认值。更详细请参考 this description.
spark.memory.offHeap.enabled               true          如果true，Spark会尝试使用堆外内存。启用 后，spark.memory.offHeap.size必须为正数。
spark.memory.offHeap.size                  0             堆外内存分配的大小（绝对值）。这个设置不会影响堆内存的使用，所以你的执行器总内存必须适应JVM的堆内存大小。必须要设为正数。并且前提是 spark.memory.offHeap.enabled=true.
spark.memory.useLegacyMode                 false         是否使用老式的内存管理模式（1.5以及之前）。老模式在堆内存管理上更死板，使用固定划分的区域做不同功能，潜在的会导致过多的数据溢出到磁盘（如果不小心调整性能）。必须启用本参数，以下选项才可用：
spark.shuffle.memoryFraction               0.2           (废弃）必须先启用spark.memory.useLegacyMode这个才有用。混洗阶段用于聚合和协同分组的JVM堆内存比例。在任何指定的时间，所有用于混洗的内存总和不会超过这个上限，超出的部分会溢出到磁盘上。如果溢出台频繁，考虑增加spark.storage.memoryFraction的大小。
spark.storage.memoryFraction               0.6           (废弃）必须先启用spark.memory.useLegacyMode这个才有用。Spark用于缓存数据的对内存比例。这个值不应该比JVM 老生代（old generation）对象所占用的内存大，默认是60%的堆内存，当然你可以增加这个值，同时配置你所用的老生代对象占用内存大小。
spark.storage.unrollFraction               0.2           (废弃）必须先启用spark.memory.useLegacyMode这个才有用。Spark块展开的内存占用比例。如果没有足够的内存来完整展开新的块，那么老的块将被抛弃。
spark.storage.replication.proactive        false	       Enables proactive block replication for RDD blocks. Cached RDD block replicas lost due to executor failures are replenished if there are any existing available replicas. This tries to get the replication level of the block to the initial number.
=====================================      ========      ===============================


Executor 行为
------------------

===============================================================     =====================================================      ====================================
属性名                                                               默认值                                                      含义
===============================================================     =====================================================      ====================================
spark.broadcast.blockSize                                           4m                                                         TorrentBroadcastFactory每个分片大小。太大会减少广播时候的并发数(更慢了);如果太小，BlockManager可能会给出性能提示。
spark.executor.cores                                                YARN模式下默认1;如果是独立部署模式, 则是 Worker 节               单个执行器可用的core数。仅针对YARN和独立部署模式。独立部署时,单个Worker节点上会运行多个Executor,只要Worker上有足够的core。否则,每个应用在单个Worker上只会启动一个Executor。
                                                                    点上所有可用的core
spark.default.parallelism                                           对于 reduceByKey 和 join 这样的分布式混洗(shuffle)算子, 等      如果用户没有在参数里指定，这个属性是默认的RDD transformation算子分区数，如：join，reduceByKey，parallelize等。
                                                                    于父RDD中最大的分区。对于parallelize这样没有父RDD的算子，则取
                                                                    决于集群管理器:
                                                                    * 本地模式：机器的core数
                                                                    * Mesos细粒度模式：8
                                                                    * 其他：所有执行器节点上core的数量或者2，这两数取较大的
spark.executor.heartbeatInterval                                    10s                                                        执行器心跳间隔（报告心跳给驱动器）。心跳机制使驱动器了解哪些执行器还活着，并且可以从心跳数据中获得执行器的度量数据。
spark.files.fetchTimeout                                            60s                                                        获取文件的通讯超时，所获取的文件是通过在驱动器上调用SparkContext.addFile()添加的。
spark.files.useFetchCache                                           true                                                       如果设为true（默认），则同一个spark应用的不同执行器之间，会使用一二共享缓存来拉取文件，这样可以提升同一主机上运行多个执行器时候，任务启动的性能。如果设为false，这个优化就被禁用，各个执行器将使用自己独有的缓存，他们拉取的文件也是各自有一份拷贝。如果在NFS文件系统上使用本地文件系统，可以禁用掉这个优化（参考SPARK-6313）
spark.files.overwrite                                               false                                                      SparkContext.addFile()添加的文件已经存在，且内容不匹配的情况下，是否覆盖。
spark.files.maxPartitionBytes	                                      134217728(128 MB)	                                         The maximum number of bytes to pack into a single partition when reading files.
spark.files.openCostInBytes	                                        4194304(4 MB)	                                             The estimated cost to open a file, measured by the number of bytes could be scanned in the same time. This is used when putting multiple files into a partition. It is better to over estimate, then the partitions with small files will be faster than partitions with bigger files.
spark.hadoop.cloneConf                                              false                                                      如设为true，对每个任务复制一份Hadoop Configuration对象。启用这个可以绕过Configuration线程安全问题（SPARK-2546 ）。默认这个是禁用的，很多job并不会受这个issue的影响。
spark.hadoop.validateOutputSpecs                                    true                                                       如设为true，在saveAsHadoopFile及其变体的时候，将会验证输出（例如，检查输出目录是否存在）。对于已经验证过或确认存在输出目录的情况，可以禁用这个。我们建议不要禁用，除非你确定需要和之前的spark版本兼容。可以简单的利用Hadoop 文件系统API手动删掉已存在的输出目录。这个设置会被Spark Streaming StreamingContext生成的job忽略，因为Streaming需要在回复检查点的时候，覆盖已有的输出目录。
spark.storage.memoryMapThreshold                                    2m                                                         Spark从磁盘上读取一个块后，映射到内存块的最小大小。这阻止了spark映射过小的内存块。通常，内存映射块是有开销的，应该比接近或小于操作系统的页大小。
spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version	      1	                                                         The file output committer algorithm version, valid algorithm version number: 1 or 2. Version 2 may have better performance, but version 1 may handle failures better in certain situations, as per MAPREDUCE-4815.
===============================================================     =====================================================      ====================================


网络
------------------

===============================      =============================      ============================
属性名                                默认值                              含义
===============================      =============================      ============================
spark.rpc.message.maxSize	           128	                              Maximum message size (in MB) to allow in "control plane" communication; generally only applies to map output size information sent between executors and the driver. Increase this if you are running jobs with many thousands of map and reduce tasks and see messages about the RPC message size.
spark.blockManager.port              (random)                           块管理器（block manager）监听端口。在驱动器和执行器上都有。
spark.driver.blockManager.port	     (spark.blockManager.port的值)	      Driver-specific port for the block manager to listen on, for cases where it cannot use the same configuration as executors.
spark.driver.bindAddress	           (spark.driver.host的值)	            Hostname or IP address where to bind listening sockets. This config overrides the SPARK_LOCAL_IP environment variable (see below).It also allows a different address from the local one to be advertised to executors or external systems. This is useful, for example, when running containers with bridged networking. For this to properly work, the different ports used by the driver (RPC, block manager and UI) need to be forwarded from the container's host.
spark.driver.host                    (local hostname)                   驱动器主机名。用于和执行器以及独立部署时集群master通讯。
spark.driver.port                    (random)                           驱动器端口。用于和执行器以及独立部署时集群master通讯。
spark.network.timeout                120s                               所有网络交互的默认超时。这个配置是以下属性的默认值：spark.core.connection.ack.wait.timeout,spark.akka.timeout,spark.storage.blockManagerSlaveTimeoutMs,spark.shuffle.io.connectionTimeout,spark.rpc.askTimeout or spark.rpc.lookupTimeout
spark.port.maxRetries                16                                 绑定一个端口的最大重试次数。如果指定了一个端口（非0），每个后续重试会在之前尝试的端口基础上加1，然后再重试绑定。本质上，这确定了一个绑定端口的范围，就是 [start port, start port + maxRetries]
spark.rpc.numRetries                 3                                  RPC任务最大重试次数。RPC任务最多重试这么多次。
spark.rpc.retry.wait                 3s                                 RPC请求操作重试前等待时间。
spark.rpc.askTimeout                 spark.network.timeout              RPC请求操作超时等待时间。
spark.rpc.lookupTimeout              120s                               RPC远程端点查询超时。
===============================      =============================      ============================


调度
------------------

========================================================      ==============================================      ====================
属性名                                                         默认值                                               含义
========================================================      ==============================================      ====================
spark.cores.max                                               (not set)                                           如果运行在独立部署模式下的集群或粗粒度共享模式下的Mesos集群, 这个值决定了 Spark应用可以使用的最大CPU总数（应用在整个集群中可用CPU总数，而不是单个机器）。如果不设置，那么独立部署时默认为spark.deploy.defaultCores，Mesos集群则默认无限制（即所有可用的CPU）。
spark.locality.wait                                           3s                                                  为了数据本地性最长等待时间（spark会根据数据所在位置，尽量让任务也启动于相同的节点，然而可能因为该节点上资源不足等原因，无法满足这个任务分配，spark最多等待这么多时间，然后放弃数据本地性）。数据本地性有多个级别，每一级别都是等待这么多时间（同一进程、同一节点、同一机架、任意）。你也可以为每个级别定义不同的等待时间，需要设置spark.locality.wait.node等。如果你发现任务数据本地性不佳，可以增加这个值，但通常默认值是ok的。
spark.locality.wait.node                                      spark.locality.wait                                 单独定义同一节点数据本地性任务等待时间。你可以设为0，表示忽略节点本地性，直接跳到下一级别，即机架本地性（如果你的集群有机架信息）。
spark.locality.wait.process                                   spark.locality.wait                                 单独定义同一进程数据本地性任务等待时间。这个参数影响试图访问特定执行器上的缓存数据的任务。
spark.locality.wait.rack                                      spark.locality.wait                                 单独定义同一机架数据本地性等待时间。
spark.scheduler.maxRegisteredResourcesWaitingTime             30s                                                 调度开始前，向集群管理器注册使用资源的最大等待时间。
spark.scheduler.minRegisteredResourcesRatio                   YARN模式是0.8,standalone和Mesos粗粒度模式是0.0          调度启动前，需要注册得到资源的最小比例(注册到的资源数/需要资源总数)(YARN模式下，资源是执行器；独立部署和Mesos粗粒度模式下时资源是CPU core['spark.cores.max'是期望得到的资源总数])。可以设为0.0~1.0的一个浮点数。不管job是否得到了最小资源比例，最大等待时间都是由spark.scheduler.maxRegisteredResourcesWaitingTime控制的。
spark.scheduler.mode                                          FIFO                                                提交到同一个SparkContext上job的调度模式（scheduling mode）。另一个可接受的值是FAIR，而FIFO只是简单的把job按先来后到排队。对于多用户服务很有用。
spark.scheduler.revive.interval                               1s                                                  调度器复活worker的间隔时间。
spark.blacklist.enabled	                                      false	                                              If set to "true", prevent Spark from scheduling tasks on executors that have been blacklisted due to too many task failures. The blacklisting algorithm can be further controlled by the other "spark.blacklist" configuration options.
spark.blacklist.timeout	                                      1h	                                                (Experimental) How long a node or executor is blacklisted for the entire application, before it is unconditionally removed from the blacklist to attempt running new tasks.
spark.blacklist.task.maxTaskAttemptsPerExecutor	              1	                                                  (Experimental) For a given task, how many times it can be retried on one executor before the executor is blacklisted for that task.
spark.blacklist.task.maxTaskAttemptsPerNode	                  2	                                                  (Experimental) For a given task, how many times it can be retried on one node, before the entire node is blacklisted for that task.
spark.blacklist.stage.maxFailedTasksPerExecutor	              2	                                                  (Experimental) How many different tasks must fail on one executor, within one stage, before the executor is blacklisted for that stage.
spark.blacklist.stage.maxFailedExecutorsPerNode	              2	                                                  (Experimental) How many different executors are marked as blacklisted for a given stage, before the entire node is marked as failed for the stage.
spark.blacklist.application.maxFailedTasksPerExecutor         2	                                                  (Experimental) How many different tasks must fail on one executor, in successful task sets, before the executor is blacklisted for the entire application. Blacklisted executors will be automatically added back to the pool of available resources after the timeout specified by spark.blacklist.timeout. Note that with dynamic allocation, though, the executors may get marked as idle and be reclaimed by the cluster manager.
spark.blacklist.application.maxFailedExecutorsPerNode         2	                                                  (Experimental) How many different executors must be blacklisted for the entire application, before the node is blacklisted for the entire application. Blacklisted nodes will be automatically added back to the pool of available resources after the timeout specified by spark.blacklist.timeout. Note that with dynamic allocation, though, the executors on the node may get marked as idle and be reclaimed by the cluster manager.
spark.blacklist.killBlacklistedExecutors	                    false	                                              (Experimental) If set to "true", allow Spark to automatically kill, and attempt to re-create, executors when they are blacklisted. Note that, when an entire node is added to the blacklist, all of the executors on that node will be killed.
spark.speculation                                             false                                               如果设为true，将会启动推测执行任务。这意味着，如果stage中有任务执行较慢，他们会被重新调度到别的节点上执行。
spark.speculation.interval                                    100ms                                               Spark检查慢任务的时间间隔。
spark.speculation.multiplier                                  1.5                                                 比任务平均执行时间慢多少倍的任务会被认为是慢任务。
spark.speculation.quantile                                    0.75                                                对于一个stage来说，完成多少百分比才开始检查慢任务，并启动推测执行任务。
spark.task.cpus                                               1                                                   每个任务分配的CPU core。
spark.task.maxFailures                                        4                                                   单个任务最大失败次数。应该>=1。最大重试次数 = spark.task.maxFailures – 1
spark.task.reaper.enabled	                                    false	                                              Enables monitoring of killed / interrupted tasks. When set to true, any task which is killed will be monitored by the executor until that task actually finishes executing. See the other spark.task.reaper.* configurations for details on how to control the exact behavior of this monitoring. When set to false (the default), task killing will use an older code path which lacks such monitoring.
spark.task.reaper.pollingInterval	                            10s	                                                When spark.task.reaper.enabled = true, this setting controls the frequency at which executors will poll the status of killed tasks. If a killed task is still running when polled then a warning will be logged and, by default, a thread-dump of the task will be logged (this thread dump can be disabled via the spark.task.reaper.threadDump setting, which is documented below).
spark.task.reaper.threadDump	                                true	                                              When spark.task.reaper.enabled = true, this setting controls whether task thread dumps are logged during periodic polling of killed tasks. Set this to false to disable collection of thread dumps.
spark.task.reaper.killTimeout	                                -1	                                                When spark.task.reaper.enabled = true, this setting specifies a timeout after which the executor JVM will kill itself if a killed task has not stopped running. The default value, -1, disables this mechanism and prevents the executor from self-destructing. The purpose of this setting is to act as a safety-net to prevent runaway uncancellable tasks from rendering an executor unusable.
spark.stage.maxConsecutiveAttempts                            4	                                                  Number of consecutive stage attempts allowed before a stage is aborted.
========================================================      ==============================================      ====================


动态分配
------------------

=========================================================      =====================================      =====================================
属性名                                                          默认值                                      含义
=========================================================      =====================================      =====================================
spark.dynamicAllocation.enabled                                false                                      是否启用动态资源分配特性，启用后，执行器的个数会根据工作负载动态的调整（增加或减少）。注意，目前在YARN模式下不用。更详细信息，请参考：here该特性依赖于 spark.shuffle.service.enabled 的启用。同时还和以下配置相关：spark.dynamicAllocation.minExecutors, spark.dynamicAllocation.maxExecutors以及 spark.dynamicAllocation.initialExecutors
spark.dynamicAllocation.executorIdleTimeout                    60s                                        动态分配特性启用后，空闲时间超过该配置时间的执行器都会被移除。更详细请参考这里：description
spark.dynamicAllocation.cachedExecutorIdleTimeout              infinity                                   动态分配特性启用后，包含缓存数据的执行器如果空闲时间超过该配置设置的时间，则被移除。更详细请参考：description
spark.dynamicAllocation.initialExecutors                       spark.dynamicAllocation.minExecutors       动态分配开启后，执行器的初始个数
spark.dynamicAllocation.maxExecutors                           infinity                                   动态分配开启后，执行器个数的上限
spark.dynamicAllocation.minExecutors                           0                                          动态分配开启后，执行器个数的下限
spark.dynamicAllocation.schedulerBacklogTimeout                1s                                         动态分配启用后，如果有任务积压的持续时间长于该配置设置的时间，则申请新的执行器。更详细请参考：description
spark.dynamicAllocation.sustainedSchedulerBacklogTimeout       schedulerBacklogTimeout                    和spark.dynamicAllocation.schedulerBacklogTimeout类似，只不过该配置对应于随后持续的执行器申请。更详细请参考： description
=========================================================      =====================================      =====================================


安全
------------------

=============================================================      ===========================================================      ============
属性名                                                              默认值                                                            含义
=============================================================      ===========================================================      ============
spark.acls.enable                                                  false                                                            是否启用Spark acls（访问控制列表）。如果启用，那么将会检查用户是否有权限查看或修改某个作业（job）。注意，检查的前提是需要知道用户是谁，所以如果用户是null，则不会做任何检查。你可以在Spark UI上设置过滤器（Filters）来做用户认证，并设置用户名。
spark.admin.acls                                                   Empty                                                            逗号分隔的用户列表，在该列表中的用户/管理员将能够访问和修改所有的Spark作业（job）。如果你的集群是共享的，并且有集群管理员，还有需要调试的开发人员，那么这个配置会很有用。如果想让所有人都有管理员权限，只需把该配置设置为”*”
spark.admin.acls.groups	                                           Empty	                                                          Comma separated list of groups that have view and modify access to all Spark jobs. This can be used if you have a set of administrators or developers who help maintain and debug the underlying infrastructure. Putting a "*" in the list means any user in any group can have the privilege of admin. The user groups are obtained from the instance of the groups mapping provider specified by spark.user.groups.mapping. Check the entry spark.user.groups.mapping for more details.
spark.user.groups.mapping                                          org.apache.spark.security.ShellBasedGroupsMappingProvider	      The list of groups for a user are determined by a group mapping service defined by the trait org.apache.spark.security.GroupMappingServiceProvider which can configured by this property. A default unix shell based implementation is provided org.apache.spark.security.ShellBasedGroupsMappingProvider which can be specified to resolve a list of groups for a user. Note: This implementation supports only a Unix/Linux based environment. Windows environment is currently not supported. However, a new platform/protocol can be supported by implementing the trait org.apache.spark.security.GroupMappingServiceProvider.
spark.authenticate                                                 false                                                            设置Spark是否认证集群内部连接。如果不是在YARN上运行，请参考 spark.authenticate.secret
spark.authenticate.secret                                          None                                                             设置Spark用于内部组件认证的秘钥。如果不是在YARN上运行，且启用了 spark.authenticate，那么该配置必须设置
spark.network.crypto.enabled	                                     false	                                                          Enable encryption using the commons-crypto library for RPC and block transfer service. Requires spark.authenticate to be enabled.
spark.network.crypto.keyLength	                                   128	                                                            The length in bits of the encryption key to generate. Valid values are 128, 192 and 256.
spark.network.crypto.keyFactoryAlgorithm	                         PBKDF2WithHmacSHA1	                                              The key factory algorithm to use when generating encryption keys. Should be one of the algorithms supported by the javax.crypto.SecretKeyFactory class in the JRE being used.
spark.network.crypto.saslFallback	                                 true	                                                            Whether to fall back to SASL authentication if authentication fails using Spark's internal mechanism. This is useful when the application is connecting to old shuffle services that do not support the internal Spark authentication protocol. On the server side, this can be used to block older clients from authenticating against a new shuffle service.
spark.network.crypto.config.*	                                     None	                                                            Configuration values for the commons-crypto library, such as which cipher implementations to use. The config name should be the name of commons-crypto configuration without the "commons.crypto" prefix.
spark.authenticate.enableSaslEncryption                            false                                                            是否对Spark内部组件认证使用加密通信。该配置目前只有 block transfer service 使用。
spark.network.sasl.serverAlwaysEncrypt                             false                                                            是否对支持SASL认证的service禁用非加密通信。该配置目前只有 external shuffle service 支持。
spark.core.connection.ack.wait.timeout                             60s                                                              网络连接等待应答信号的超时时间。为了避免由于GC等导致的意外超时，你可以设置一个较大的值。
spark.core.connection.auth.wait.timeout                            30s                                                              网络连接等待认证的超时时间。
spark.modify.acls                                                  Empty                                                            逗号分隔的用户列表，在改列表中的用户可以修改Spark作业。默认情况下，只有启动该Spark作业的用户可以修改之（比如杀死该作业）。如果想要任何用户都可以修改作业，请将该配置设置为”*”
spark.modify.acls.groups	                                         Empty	                                                          Comma separated list of groups that have modify access to the Spark job. This can be used if you have a set of administrators or developers from the same team to have access to control the job. Putting a "*" in the list means any user in any group has the access to modify the Spark job. The user groups are obtained from the instance of the groups mapping provider specified by spark.user.groups.mapping. Check the entry spark.user.groups.mapping for more details.
spark.ui.filters                                                   None                                                             逗号分隔的过滤器class列表，这些过滤器将用于Spark web UI。这里的过滤器应该是一个标准的 javax servlet Filter 。每个过滤器的参数可以通过java系统属性来设置，如下：spark.<class name of filer>.params=’param1=value1,param2=value2’例如：-Dspark.ui.filters=com.test.filter1 -Dspark.com.test.filter1.params=’param1=foo,param2=testing’
spark.ui.view.acls                                                 Empty                                                            逗号分隔的用户列表，在该列表中的用户可以查看Spark web UI。默认，只有启动该Spark作业的用户可以查看之。如果需要让所有用户都能查看，只需将该配置设为”*”
spark.ui.view.acls.groups	                                         Empty	                                                          Comma separated list of groups that have view access to the Spark web ui to view the Spark Job details. This can be used if you have a set of administrators or developers or users who can monitor the Spark job submitted. Putting a "*" in the list means any user in any group can view the Spark job details on the Spark web ui. The user groups are obtained from the instance of the groups mapping provider specified by spark.user.groups.mapping. Check the entry spark.user.groups.mapping for more details.
=============================================================      ===========================================================      ============


TLS / SSL
------------------

===============================      =======      ======================
属性名                                默认值        含义
===============================      =======      ======================
spark.ssl.enabled                    false        是否启用SSL连接（在所有所支持的协议上）。所有SSL相关配置（spark.ssl.xxx，其中xxx是一个特定的配置属性），都是全局的。如果需要在某些协议上覆盖全局设置，那么需要在该协议命名空间上进行单独配置。使用 spark.ssl.YYY.XXX 来为协议YYY覆盖全局配置XXX。目前YYY的可选值有 akka（用于基于AKKA框架的网络连接） 和 fs（用于应广播和文件服务器）
spark.ssl.[namespace].port           None         The port where the SSL service will listen on.The port must be defined within a namespace configuration; see SSL Configuration for the available namespaces.When not set, the SSL port will be derived from the non-SSL port for the same service. A value of "0" will make the service bind to an ephemeral port.
spark.ssl.enabledAlgorithms          Empty        逗号分隔的加密算法列表。这些加密算法必须是JVM所支持的。这里有个可用加密算法参考列表： this
spark.ssl.keyPassword                None         在key-store中私匙对应的密码。
spark.ssl.keyStore                   None         key-store文件路径。可以是绝对路径，或者以本组件启动的工作目录为基础的相对路径。
spark.ssl.keyStorePassword           None         key-store的密码。
spark.ssl.keyStoreType	             JKS	        The type of the key-store.
spark.ssl.protocol                   None         协议名称。该协议必须是JVM所支持的。这里有JVM支持的协议参考列表：this
spark.ssl.needClientAuth             false        Set true if SSL needs client authentication.
spark.ssl.trustStore                 None         trust-store文件路径。可以是绝对路径，或者以本组件启动的工作目录为基础的相对路径。
spark.ssl.trustStorePassword         None         trust-store的密码
spark.ssl.trustStoreType             JKS          The type of the trust-store.
===============================      =======      ======================


Spark SQL
------------------

Running the SET -v command will show the entire list of the SQL configuration.

Scala

.. code-block:: Scala

  // spark is an existing SparkSession
  spark.sql("SET -v").show(numRows = 200, truncate = false)


Java

.. code-block:: Java

  // spark is an existing SparkSesson
  spark.sql("SET -v").show(200, false);


Python

.. code-block:: Python

  # spark is an existing SparkSession
  spark.sql("SET -v").show(n=200, truncate=False)


R

.. code-block:: R

  sparkR.session()
  properties <- sql("SET -v")
  showDF(properties, numRows = 200, truncate = FALSE)



Spark Streaming
------------------

=============================================================      ==========      ===========
属性名                                                              默认值           含义
=============================================================      ==========      ===========
spark.streaming.backpressure.enabled                               false           是否启用Spark Streaming 的内部反压机制（spark 1.5以上支持）。启用后，Spark Streaming会根据当前批次的调度延迟和处理时长来控制接收速率，这样一来，系统的接收速度会和处理速度相匹配。该特性会在内部动态地设置接收速率。该速率的上限将由 spark.streaming.receiver.maxRate 和 spark.streaming.kafka.maxRatePerPartition 决定（如果它们设置了的话）。
spark.streaming.backpressure.initialRate	                         not             set	This is the initial maximum receiving rate at which each receiver will receive data for the first batch when the backpressure mechanism is enabled.
spark.streaming.blockInterval                                      200ms           在将数据保存到Spark之前，Spark Streaming接收器组装数据块的时间间隔。建议不少于50ms。关于Spark Streaming编程指南细节，请参考 performance tuning 这一节。
spark.streaming.receiver.maxRate                                   not set         接收速度的最大速率（每秒记录条数）。实际上，每个流每秒将消费这么多条记录。设置为0或者负数表示不限制速率。更多细节请参考： deployment guide
spark.streaming.receiver.writeAheadLog.enable                      false           是否启用接收器预写日志。所有的输入数据都会保存到预写日志中，这样在驱动器失败后，可以基于预写日志来恢复数据。更详细请参考：deployment guide
spark.streaming.unpersist                                          true            是否强制Spark Streaming 自动从内存中清理掉所生成并持久化的RDD。同时，Spark Streaming收到的原始数据也将会被自动清理掉。如果设置为false，那么原始数据以及持久化的RDD将不会被自动清理，以便外部程序可以访问这些数据。当然，这将导致Spark消耗更多的内存。
spark.streaming.stopGracefullyOnShutdown                           false           如果设为true，Spark将会在JVM关闭时，优雅地关停StreamingContext，而不是立即关闭之。
spark.streaming.kafka.maxRatePerPartition                          not set         在使用Kafka direct stream API时，从每个Kafka数据分区读取数据的最大速率（每秒记录条数）。更详细请参考：Kafka Integration guide
spark.streaming.kafka.maxRetries                                   1               驱动器连续重试的最大次数，这个配置是为了让驱动器找出每个Kafka分区上的最大offset（默认值为1，意味着驱动器将最多尝试2次）。只对新的Kafka direct stream API有效。
spark.streaming.ui.retainedBatches                                 1000            Spark Streaming UI 以及 status API 中保留的最大批次个数。
spark.streaming.driver.writeAheadLog.closeFileAfterWrite           false	         Whether to close the file after writing a write ahead log record on the driver. Set this to 'true' when you want to use S3 (or any file system that does not support flushing) for the metadata WAL on the driver.
spark.streaming.receiver.writeAheadLog.closeFileAfterWrite	       false	         Whether to close the file after writing a write ahead log record on the receivers. Set this to 'true' when you want to use S3 (or any file system that does not support flushing) for the data WAL on the receivers.
=============================================================      ==========      ===========


SparkR
------------------

==================================      ================      ================
属性名                                   默认值                 含义
==================================      ================      ================
spark.r.numRBackendThreads              2                     SparkR RBackEnd处理RPC调用的后台线程数
spark.r.command                         Rscript               集群模式下，驱动器和worker上执行的R脚本可执行文件
spark.r.driver.command                  spark.r.command       client模式的驱动器执行的R脚本。集群模式下会忽略
spark.r.shell.command	                  R	                    Executable for executing sparkR shell in client modes for driver. Ignored in cluster modes. It is the same as environment variable SPARKR_DRIVER_R, but take precedence over it. spark.r.shell.command is used for sparkR shell while spark.r.driver.command is used for running R script.
spark.r.backendConnectionTimeout	      6000	                Connection timeout set by R process on its connection to RBackend in seconds.
spark.r.heartBeatInterval	              100	                  Interval for heartbeats sent from SparkR backend to R process to prevent connection timeout.
==================================      ================      ================


GraphX
------------------


=======================================      ================      ================
属性名                                        默认值                 含义
=======================================      ================      ================
spark.graphx.pregel.checkpointInterval	     -1	                   Checkpoint interval for graph and message in Pregel. It used to avoid stackOverflowError due to long lineage chains after lots of iterations. The checkpoint is disabled by default.
=======================================      ================      ================


部署
------------------

============================      ================      ================
属性名                             默认值                 含义
============================      ================      ================
spark.deploy.recoveryMode	        NONE	                The recovery mode setting to recover submitted Spark jobs with cluster mode when it failed and relaunches. This is only applicable for cluster mode when running with Standalone or Mesos.
spark.deploy.zookeeper.url	      None	                When `spark.deploy.recoveryMode` is set to ZOOKEEPER, this configuration is used to set the zookeeper URL to connect to.
spark.deploy.zookeeper.dir	      None	                When `spark.deploy.recoveryMode` is set to ZOOKEEPER, this configuration is used to set the zookeeper directory to store recovery state.
============================      ================      ================


集群管理器
------------------

每个集群管理器都有一些额外的配置选项。详细请参考这里：

YARN
Mesos
Standalone Mode


******************
环境变量
******************

有些Spark设置需要通过环境变量来设定，这些环境变量可以在${SPARK_HOME}/conf/spark-env.sh脚本（Windows下是conf/spark-env.cmd）中设置。如果是独立部署或者Mesos模式，这个文件可以指定机器相关信息（如hostname）。运行本地Spark应用或者submit脚本时，也会引用这个文件。

注意，conf/spark-env.sh默认是不存在的。你需要复制conf/spark-env.sh.template这个模板来创建，还有注意给这个文件附上可执行权限。

以下变量可以在spark-env.sh中设置：

=======================      =======================================================
环境变量                       含义
=======================      =======================================================
JAVA_HOME                    Java安装目录（如果没有在PATH变量中指定）
PYSPARK_PYTHON               驱动器和worker上使用的Python二进制可执行文件（默认是python）
PYSPARK_DRIVER_PYTHON        仅在驱动上使用的Python二进制可执行文件（默认同PYSPARK_PYTHON）
SPARKR_DRIVER_R              SparkR shell使用的R二进制可执行文件（默认是R）
SPARK_LOCAL_IP               本地绑定的IP
SPARK_PUBLIC_DNS             Spark程序公布给其他机器的hostname
=======================      =======================================================

除了上面的环境变量之外，还有一些选项需要在Spark standalone cluster scripts里设置，如：每台机器上使用的core数量，和最大内存占用量。

spark-env.sh是一个shell脚本，因此一些参数可以通过编程方式来设定 – 例如，你可以获取本机IP来设置SPARK_LOCAL_IP。


******************
日志配置
******************

Spark使用log4j 打日志。你可以在conf目录下用log4j.properties来配置。复制该目录下已有的log4j.properties.template并改名为log4j.properties即可。


******************
覆盖配置目录
******************

默认Spark配置目录是”${SPARK_HOME}/conf”，你也可以通过 ${SPARK_CONF_DIR}指定其他目录。Spark会从这个目录下读取配置文件（spark-defaults.conf，spark-env.sh，log4j.properties等）


******************
继承Hadoop集群配置
******************

如果你打算用Spark从HDFS读取数据，那么有2个Hadoop配置文件必须放到Spark的classpath下：
* hdfs-site.xml，配置HDFS客户端的默认行为
* core-site.xml，默认文件系统名
这些配置文件的路径在不同发布版本中不太一样（如CDH和HDP版本），但通常都能在 ${HADOOP_HOME}/etc/hadoop/conf目录下找到。一些工具，如Cloudera Manager，可以动态修改配置，而且提供了下载一份拷贝的机制。

要想让这些配置对Spark可见，请在${SPARK_HOME}/spark-env.sh中设置HADOOP_CONF_DIR变量。
