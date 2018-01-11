Spark SQL, DataFrame和Dataset编程指南
============================================

概述
----------------

Spark SQL 是 Spark 用于处理结构化数据的一个模块。不同于基础的 Spark RDD API，Spark SQL 提供的接口提供了更多关于数
据和执行的计算任务的结构信息。Spark SQL 内部使用这些额外的信息来执行一些额外的优化操作。有几种方式可以与 Spark SQL 进
行交互，其中包括 SQL 和 Dataset API。当计算一个结果时 Spark SQL 使用的执行引擎是一样的, 它跟你使用哪种 API 或编程语
言来表达计算无关。这种统一意味着开发人员可以很容易地在不同的 API 之间来回切换，基于哪种 API 能够提供一种最自然的方式来表
达一个给定的变换（transformation）。

本文中所有的示例程序都使用 Spark 发行版本中自带的样本数据，并且可以在 spark-shell、pyspark shell 以及 sparkR shell 中运行。


SQL
^^^^^^^^^^^^^^^^^^^

Spark SQL 的用法之一是执行 SQL 查询，它也可以从现有的 Hive 中读取数据，想要了解更多关于如何配置这个特性的细节, 请参考 Hive表 这节。
如果从其它编程语言内部运行 SQL，查询结果将作为一个 Dataset/DataFrame 返回。你还可以使用命令行或者通过 JDBC/ODBC 来
与 SQL 接口进行交互。


Dataset和DataFrame
^^^^^^^^^^^^^^^^^^^

Dataset 是一个分布式数据集，它是 Spark 1.6 版本中新增的一个接口, 它结合了 RDD（强类型，可以使用强大的 lambda 表达式函数）
和 Spark SQL 的优化执行引擎的好处。Dataset 可以从 JVM 对象构造得到，随后可以使用函数式的变换（map，flatMap，filter 等）
进行操作。Dataset API 目前支持 Scala 和 Java 语言，还不支持 Python, 不过由于 Python 语言的动态性, Dataset API 的许
多好处早就已经可用了（例如，你可以使用 row.columnName 来访问数据行的某个字段）。这种场景对于 R 语言来说是类似的。

DataFrame 是按命名列方式组织的一个 Dataset。从概念上来讲，它等同于关系型数据库中的一张表或者 R 和 Python 中的一个 data frame，
只不过在底层进行了更多的优化。DataFrame 可以从很多数据源构造得到，比如：结构化的数据文件，Hive 表，外部数据库或现有的 RDD。
DataFrame API 支持 Scala, Java, Python 以及 R 语言。在 Scala 和 Java 语言中, DataFrame 由 Row 的 Dataset 来
表示的。在 Scala API 中, DataFrame 仅仅只是 Dataset[Row] 的一个类型别名，而在 Java API 中, 开发人员需要使用 Dataset<Row> 来
表示一个 DataFrame。


入门
----------------



入口: SparkSession
^^^^^^^^^^^^^^^^^^^

创建 DataFrame
^^^^^^^^^^^^^^^^^^^

入口:SparkSession
^^^^^^^^^^^^^^^^^^^

入口:SparkSession
^^^^^^^^^^^^^^^^^^^

创建 Dataset
^^^^^^^^^^^^^^^^^^^

与 RDD 互操作
^^^^^^^^^^^^^^^^^^^

聚合
^^^^^^^^^^^^^^^^^^^

