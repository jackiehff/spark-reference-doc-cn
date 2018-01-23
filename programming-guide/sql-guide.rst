.. _sql_programming_guide:

############################################
Spark SQL, DataFrame 和 Dataset 编程指南
############################################


*****************
概述
*****************

Spark SQL 是 Spark 用于处理结构化数据的一个模块。不同于基础的 Spark RDD API，Spark SQL 提供的接口提供了更多关于数据和执行的计算任务的结构信息。Spark SQL 内部使用这些额外的信息来执行一些额外的优化操作。有几种方式可以与 Spark SQL 进
行交互，其中包括 SQL 和 Dataset API。当计算一个结果时 Spark SQL 使用的执行引擎是一样的, 它跟你使用哪种 API 或编程语
言来表达计算无关。这种统一意味着开发人员可以很容易地在不同的 API 之间来回切换，基于哪种 API 能够提供一种最自然的方式来表
达一个给定的变换（transformation）。

本文中所有的示例程序都使用 Spark 发行版本中自带的样本数据，并且可以在 spark-shell、pyspark shell 以及 sparkR shell 中运行。


SQL
========================

Spark SQL 的用法之一是执行 SQL 查询，它也可以从现有的 Hive 中读取数据，想要了解更多关于如何配置这个特性的细节, 请参考 Hive表 这节。
如果从其它编程语言内部运行 SQL，查询结果将作为一个 Dataset/DataFrame 返回。你还可以使用命令行或者通过 JDBC/ODBC 来
与 SQL 接口进行交互。


Dataset 和 DataFrame
========================

Dataset 是一个分布式数据集，它是 Spark 1.6 版本中新增的一个接口, 它结合了 RDD（强类型，可以使用强大的 lambda 表达式函数）
和 Spark SQL 的优化执行引擎的好处。Dataset 可以从 JVM 对象构造得到，随后可以使用函数式的变换（map，flatMap，filter 等）
进行操作。Dataset API 目前支持 Scala 和 Java 语言，还不支持 Python, 不过由于 Python 语言的动态性, Dataset API 的许
多好处早就已经可用了（例如，你可以使用 row.columnName 来访问数据行的某个字段）。这种场景对于 R 语言来说是类似的。

DataFrame 是按命名列方式组织的一个 Dataset。从概念上来讲，它等同于关系型数据库中的一张表或者 R 和 Python 中的一个 data frame，
只不过在底层进行了更多的优化。DataFrame 可以从很多数据源构造得到，比如：结构化的数据文件，Hive 表，外部数据库或现有的 RDD。
DataFrame API 支持 Scala, Java, Python 以及 R 语言。在 Scala 和 Java 语言中, DataFrame 由 Row 的 Dataset 来
表示的。在 Scala API 中, DataFrame 仅仅只是 Dataset[Row] 的一个类型别名，而在 Java API 中, 开发人员需要使用 Dataset<Row> 来
表示一个 DataFrame。

在本文档中, 我们将经常把 Scala/Java Row 的 Dataset 作为 DataFrame。


************************
入门
************************

入口: SparkSession
===================

**Scala**

Spark 中所有功能的入口是 SparkSession 类。要创建一个基本的 SparkSession 对象, 只需要使用 SparkSession.builder():

.. code-block:: Scala

  import org.apache.spark.sql.SparkSession

  val spark = SparkSession
    .builder()
    .appName("Spark SQL basic example")
    .config("spark.some.config.option", "some-value")
    .getOrCreate()

  // For implicit conversions like converting RDDs to DataFrames
  import spark.implicits._

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 文件。


**Java**

Spark 中所有功能的入口是 SparkSession 类。要创建一个基本的 SparkSession 对象, 只需要使用 SparkSession.builder():

.. code-block:: Java

  import org.apache.spark.sql.SparkSession;

  SparkSession spark = SparkSession
    .builder()
    .appName("Java Spark SQL basic example")
    .config("spark.some.config.option", "some-value")
    .getOrCreate();

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 文件。

**Python**

Spark 中所有功能的入口是 SparkSession 类。要创建一个基本的 SparkSession 对象, 只需要使用 SparkSession.builder:

.. code-block:: Python

  from pyspark.sql import SparkSession

  spark = SparkSession \
      .builder \
      .appName("Python Spark SQL basic example") \
      .config("spark.some.config.option", "some-value") \
      .getOrCreate()

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/basic.py" 文件。

**R**

Spark 中所有功能的入口是 SparkSession 类。要初始化一个基本的 SparkSession 对象, 只需要调用 sparkR.session():

.. code-block:: R

  sparkR.session(appName = "R Spark SQL basic example", sparkConfig = list(spark.some.config.option = "some-value"))

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。

.. attention:: 当第一次调用时, sparkR.session() 会初始化一个全局的 SparkSession 单例实例, 并且总是为后续的调用返回该实例的引用。这样的话, 用户只需要初始化 SparkSession 一次, 然后像 read.df 这样的 SparkR 函数就可以隐式地访问该全局实例, 并且用户不需要传递 SparkSession 实例。


Spark 2.0 中的 SparkSession 提供了对 Hive 特性的内置支持，包括使用 HiveQL 编写查询，访问 Hive UDF 以及从 Hive 表读取数据。要使用这些特性，你不需要预先安装 Hive。


创建 DataFrame
=================

**Scala**

应用程序可以使用 SparkSession 从一个现有的 RDD，Hive 表或 Spark 数据源创建 DataFrame。

举个例子, 下面基于一个 JSON 文件的内容创建一个 DataFrame:

.. code-block:: Scala

  val df = spark.read.json("examples/src/main/resources/people.json")

  // Displays the content of the DataFrame to stdout
  df.show()
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 文件。

**Java**

应用程序可以使用 SparkSession 从一个现有的 RDD，Hive 表或 Spark 数据源创建 DataFrame。

举个例子, 下面基于一个 JSON 文件的内容创建一个 DataFrame:

.. code-block:: Java

  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Row;

  Dataset<Row> df = spark.read().json("examples/src/main/resources/people.json");

  // Displays the content of the DataFrame to stdout
  df.show();
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 文件。

**Python**

应用程序可以使用 SparkSession 从一个现有的 RDD，Hive 表或 Spark 数据源创建 DataFrame。

举个例子, 下面基于一个 JSON 文件的内容创建一个 DataFrame:

.. code-block:: Python

  # spark is an existing SparkSession
  df = spark.read.json("examples/src/main/resources/people.json")
  # Displays the content of the DataFrame to stdout
  df.show()
  # +----+-------+
  # | age|   name|
  # +----+-------+
  # |null|Michael|
  # |  30|   Andy|
  # |  19| Justin|
  # +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/basic.py" 文件。

**R**

应用程序可以使用 SparkSession 从一个本地的 R data.frame, Hive 表或 Spark 数据源创建 DataFrame。

举个例子, 下面基于一个 JSON 文件的内容创建一个 DataFrame:

.. code-block:: R

  df <- read.json("examples/src/main/resources/people.json")

  # Displays the content of the DataFrame
  head(df)
  ##   age    name
  ## 1  NA Michael
  ## 2  30    Andy
  ## 3  19  Justin

  # Another method to print the first few rows and optionally truncate the printing of long values
  showDF(df)
  ## +----+-------+
  ## | age|   name|
  ## +----+-------+
  ## |null|Michael|
  ## |  30|   Andy|
  ## |  19| Justin|
  ## +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。


无类型的 Dataset 操作 (亦即 DataFrame 操作)
=======================================================

DataFrame 为 Scala, Java, Python 以及 R 语言中的结构化数据操作提供了一种领域特定语言。

正如上面所提到的,Spark 2.0 中, Scala 和 Java API 中的 DataFrame 只是 Row 的 Dataset。与使用强类型的 Scala/Java Dataset “强类型转换” 相比，这些操作也被称为 “非强类型转换” 。
These operations are also referred as “untyped transformations” in contrast to “typed transformations” come with strongly typed Scala/Java Datasets.

下面是使用 Dataset 处理结构化数据的几个基础示例：


**Scala**

.. code-block:: Scala

  // This import is needed to use the $-notation
  import spark.implicits._
  // Print the schema in a tree format
  df.printSchema()
  // root
  // |-- age: long (nullable = true)
  // |-- name: string (nullable = true)

  // Select only the "name" column
  df.select("name").show()
  // +-------+
  // |   name|
  // +-------+
  // |Michael|
  // |   Andy|
  // | Justin|
  // +-------+

  // Select everybody, but increment the age by 1
  df.select($"name", $"age" + 1).show()
  // +-------+---------+
  // |   name|(age + 1)|
  // +-------+---------+
  // |Michael|     null|
  // |   Andy|       31|
  // | Justin|       20|
  // +-------+---------+

  // Select people older than 21
  df.filter($"age" > 21).show()
  // +---+----+
  // |age|name|
  // +---+----+
  // | 30|Andy|
  // +---+----+

  // Count people by age
  df.groupBy("age").count().show()
  // +----+-----+
  // | age|count|
  // +----+-----+
  // |  19|    1|
  // |null|    1|
  // |  30|    1|
  // +----+-----+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 文件。


**Java**

.. code-block:: Java

  // col("...") is preferable to df.col("...")
  import static org.apache.spark.sql.functions.col;

  // Print the schema in a tree format
  df.printSchema();
  // root
  // |-- age: long (nullable = true)
  // |-- name: string (nullable = true)

  // Select only the "name" column
  df.select("name").show();
  // +-------+
  // |   name|
  // +-------+
  // |Michael|
  // |   Andy|
  // | Justin|
  // +-------+

  // Select everybody, but increment the age by 1
  df.select(col("name"), col("age").plus(1)).show();
  // +-------+---------+
  // |   name|(age + 1)|
  // +-------+---------+
  // |Michael|     null|
  // |   Andy|       31|
  // | Justin|       20|
  // +-------+---------+

  // Select people older than 21
  df.filter(col("age").gt(21)).show();
  // +---+----+
  // |age|name|
  // +---+----+
  // | 30|Andy|
  // +---+----+

  // Count people by age
  df.groupBy("age").count().show();
  // +----+-----+
  // | age|count|
  // +----+-----+
  // |  19|    1|
  // |null|    1|
  // |  30|    1|
  // +----+-----+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 文件。


**Python**

In Python it’s possible to access a DataFrame’s columns either by attribute (df.age) or by indexing (df['age']). While the former is convenient for interactive data exploration, users are highly encouraged to use the latter form, which is future proof and won’t break with column names that are also attributes on the DataFrame class.

.. code-block:: Python

  # spark, df are from the previous example
  # Print the schema in a tree format
  df.printSchema()
  # root
  # |-- age: long (nullable = true)
  # |-- name: string (nullable = true)

  # Select only the "name" column
  df.select("name").show()
  # +-------+
  # |   name|
  # +-------+
  # |Michael|
  # |   Andy|
  # | Justin|
  # +-------+

  # Select everybody, but increment the age by 1
  df.select(df['name'], df['age'] + 1).show()
  # +-------+---------+
  # |   name|(age + 1)|
  # +-------+---------+
  # |Michael|     null|
  # |   Andy|       31|
  # | Justin|       20|
  # +-------+---------+

  # Select people older than 21
  df.filter(df['age'] > 21).show()
  # +---+----+
  # |age|name|
  # +---+----+
  # | 30|Andy|
  # +---+----+

  # Count people by age
  df.groupBy("age").count().show()
  # +----+-----+
  # | age|count|
  # +----+-----+
  # |  19|    1|
  # |null|    1|
  # |  30|    1|
  # +----+-----+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/basic.py" 文件。


**R**

.. code-block:: R

  # Create the DataFrame
  df <- read.json("examples/src/main/resources/people.json")

  # Show the content of the DataFrame
  head(df)
  ##   age    name
  ## 1  NA Michael
  ## 2  30    Andy
  ## 3  19  Justin


  # Print the schema in a tree format
  printSchema(df)
  ## root
  ## |-- age: long (nullable = true)
  ## |-- name: string (nullable = true)

  # Select only the "name" column
  head(select(df, "name"))
  ##      name
  ## 1 Michael
  ## 2    Andy
  ## 3  Justin

  # Select everybody, but increment the age by 1
  head(select(df, df$name, df$age + 1))
  ##      name (age + 1.0)
  ## 1 Michael          NA
  ## 2    Andy          31
  ## 3  Justin          20

  # Select people older than 21
  head(where(df, df$age > 21))
  ##   age name
  ## 1  30 Andy

  # Count people by age
  head(count(groupBy(df, "age")))
  ##   age count
  ## 1  19     1
  ## 2  NA     1
  ## 3  30     1

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。


Running SQL Queries Programmatically
=========================================

**Scala**

The sql function on a SparkSession enables applications to run SQL queries programmatically and returns the result as a DataFrame.

.. code-block:: Scala

  // Register the DataFrame as a SQL temporary view
  df.createOrReplaceTempView("people")

  val sqlDF = spark.sql("SELECT * FROM people")
  sqlDF.show()
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 文件。


**Java**

The sql function on a SparkSession enables applications to run SQL queries programmatically and returns the result as a Dataset<Row>.

.. code-block:: Java

  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Row;

  // Register the DataFrame as a SQL temporary view
  df.createOrReplaceTempView("people");

  Dataset<Row> sqlDF = spark.sql("SELECT * FROM people");
  sqlDF.show();
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 文件。

**Python**

The sql function on a SparkSession enables applications to run SQL queries programmatically and returns the result as a DataFrame.

.. code-block:: Python

  # Register the DataFrame as a SQL temporary view
  df.createOrReplaceTempView("people")

  sqlDF = spark.sql("SELECT * FROM people")
  sqlDF.show()
  # +----+-------+
  # | age|   name|
  # +----+-------+
  # |null|Michael|
  # |  30|   Andy|
  # |  19| Justin|
  # +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/basic.py" 文件。


**R**

The sql function enables applications to run SQL queries programmatically and returns the result as a SparkDataFrame.

.. code-block:: R

  df <- sql("SELECT * FROM table")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。


Global Temporary View
==============================

Temporary views in Spark SQL are session-scoped and will disappear if the session that creates it terminates. If you want to have a temporary view that is shared among all sessions and keep alive until the Spark application terminates, you can create a global temporary view. Global temporary view is tied to a system preserved database global_temp, and we must use the qualified name to refer it, e.g. SELECT * FROM global_temp.view1.

**Scala**

.. code-block:: Scala

  // Register the DataFrame as a global temporary view
  df.createGlobalTempView("people")

  // Global temporary view is tied to a system preserved database `global_temp`
  spark.sql("SELECT * FROM global_temp.people").show()
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

  // Global temporary view is cross-session
  spark.newSession().sql("SELECT * FROM global_temp.people").show()
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 文件。

**Java**

.. code-block:: Java

  // Register the DataFrame as a global temporary view
  df.createGlobalTempView("people");

  // Global temporary view is tied to a system preserved database `global_temp`
  spark.sql("SELECT * FROM global_temp.people").show();
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

  // Global temporary view is cross-session
  spark.newSession().sql("SELECT * FROM global_temp.people").show();
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 文件。

**Python**

.. code-block:: Python

  # Register the DataFrame as a global temporary view
  df.createGlobalTempView("people")

  # Global temporary view is tied to a system preserved database `global_temp`
  spark.sql("SELECT * FROM global_temp.people").show()
  # +----+-------+
  # | age|   name|
  # +----+-------+
  # |null|Michael|
  # |  30|   Andy|
  # |  19| Justin|
  # +----+-------+

  # Global temporary view is cross-session
  spark.newSession().sql("SELECT * FROM global_temp.people").show()
  # +----+-------+
  # | age|   name|
  # +----+-------+
  # |null|Michael|
  # |  30|   Andy|
  # |  19| Justin|
  # +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/basic.py" 文件。

**Sql**

.. code-block:: SQL

  CREATE GLOBAL TEMPORARY VIEW temp_view AS SELECT a + 1, b * 2 FROM tbl
  SELECT * FROM global_temp.temp_view


创建 Dataset
==============================

Datasets are similar to RDDs, however, instead of using Java serialization or Kryo they use a specialized Encoder to serialize the objects for processing or transmitting over the network. While both encoders and standard serialization are responsible for turning an object into bytes, encoders are code generated dynamically and use a format that allows Spark to perform many operations like filtering, sorting and hashing without deserializing the bytes back into an object.

**Scala**

.. code-block:: Scala

  // Note: Case classes in Scala 2.10 can support only up to 22 fields. To work around this limit,
  // you can use custom classes that implement the Product interface
  case class Person(name: String, age: Long)

  // Encoders are created for case classes
  val caseClassDS = Seq(Person("Andy", 32)).toDS()
  caseClassDS.show()
  // +----+---+
  // |name|age|
  // +----+---+
  // |Andy| 32|
  // +----+---+

  // Encoders for most common types are automatically provided by importing spark.implicits._
  val primitiveDS = Seq(1, 2, 3).toDS()
  primitiveDS.map(_ + 1).collect() // Returns: Array(2, 3, 4)

  // DataFrames can be converted to a Dataset by providing a class. Mapping will be done by name
  val path = "examples/src/main/resources/people.json"
  val peopleDS = spark.read.json(path).as[Person]
  peopleDS.show()
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 文件。

**Java**

.. code-block:: Java

  import java.util.Arrays;
  import java.util.Collections;
  import java.io.Serializable;

  import org.apache.spark.api.java.function.MapFunction;
  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Row;
  import org.apache.spark.sql.Encoder;
  import org.apache.spark.sql.Encoders;

  public static class Person implements Serializable {
    private String name;
    private int age;

    public String getName() {
      return name;
    }

    public void setName(String name) {
      this.name = name;
    }

    public int getAge() {
      return age;
    }

    public void setAge(int age) {
      this.age = age;
    }
  }

  // Create an instance of a Bean class
  Person person = new Person();
  person.setName("Andy");
  person.setAge(32);

  // Encoders are created for Java beans
  Encoder<Person> personEncoder = Encoders.bean(Person.class);
  Dataset<Person> javaBeanDS = spark.createDataset(
    Collections.singletonList(person),
    personEncoder
  );
  javaBeanDS.show();
  // +---+----+
  // |age|name|
  // +---+----+
  // | 32|Andy|
  // +---+----+

  // Encoders for most common types are provided in class Encoders
  Encoder<Integer> integerEncoder = Encoders.INT();
  Dataset<Integer> primitiveDS = spark.createDataset(Arrays.asList(1, 2, 3), integerEncoder);
  Dataset<Integer> transformedDS = primitiveDS.map(
      (MapFunction<Integer, Integer>) value -> value + 1,
      integerEncoder);
  transformedDS.collect(); // Returns [2, 3, 4]

  // DataFrames can be converted to a Dataset by providing a class. Mapping based on name
  String path = "examples/src/main/resources/people.json";
  Dataset<Person> peopleDS = spark.read().json(path).as(personEncoder);
  peopleDS.show();
  // +----+-------+
  // | age|   name|
  // +----+-------+
  // |null|Michael|
  // |  30|   Andy|
  // |  19| Justin|
  // +----+-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 文件。


与 RDD 互操作
==============================

Spark SQL supports two different methods for converting existing RDDs into Datasets. The first method uses reflection to infer the schema of an RDD that contains specific types of objects. This reflection based approach leads to more concise code and works well when you already know the schema while writing your Spark application.

The second method for creating Datasets is through a programmatic interface that allows you to construct a schema and then apply it to an existing RDD. While this method is more verbose, it allows you to construct Datasets when the columns and their types are not known until runtime.

Inferring the Schema Using Reflection
-----------------------------------------

**Scala**

The Scala interface for Spark SQL supports automatically converting an RDD containing case classes to a DataFrame. The case class defines the schema of the table. The names of the arguments to the case class are read using reflection and become the names of the columns. Case classes can also be nested or contain complex types such as Seqs or Arrays. This RDD can be implicitly converted to a DataFrame and then be registered as a table. Tables can be used in subsequent SQL statements.

.. code-block:: Scala

  // For implicit conversions from RDDs to DataFrames
  import spark.implicits._

  // Create an RDD of Person objects from a text file, convert it to a Dataframe
  val peopleDF = spark.sparkContext
    .textFile("examples/src/main/resources/people.txt")
    .map(_.split(","))
    .map(attributes => Person(attributes(0), attributes(1).trim.toInt))
    .toDF()
  // Register the DataFrame as a temporary view
  peopleDF.createOrReplaceTempView("people")

  // SQL statements can be run by using the sql methods provided by Spark
  val teenagersDF = spark.sql("SELECT name, age FROM people WHERE age BETWEEN 13 AND 19")

  // The columns of a row in the result can be accessed by field index
  teenagersDF.map(teenager => "Name: " + teenager(0)).show()
  // +------------+
  // |       value|
  // +------------+
  // |Name: Justin|
  // +------------+

  // or by field name
  teenagersDF.map(teenager => "Name: " + teenager.getAs[String]("name")).show()
  // +------------+
  // |       value|
  // +------------+
  // |Name: Justin|
  // +------------+

  // No pre-defined encoders for Dataset[Map[K,V]], define explicitly
  implicit val mapEncoder = org.apache.spark.sql.Encoders.kryo[Map[String, Any]]
  // Primitive types and case classes can be also defined as
  // implicit val stringIntMapEncoder: Encoder[Map[String, Any]] = ExpressionEncoder()

  // row.getValuesMap[T] retrieves multiple columns at once into a Map[String, T]
  teenagersDF.map(teenager => teenager.getValuesMap[Any](List("name", "age"))).collect()
  // Array(Map("name" -> "Justin", "age" -> 19))

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 文件。


**Java**

Spark SQL supports automatically converting an RDD of JavaBeans into a DataFrame. The BeanInfo, obtained using reflection, defines the schema of the table. Currently, Spark SQL does not support JavaBeans that contain Map field(s). Nested JavaBeans and List or Array fields are supported though. You can create a JavaBean by creating a class that implements Serializable and has getters and setters for all of its fields.

.. code-block:: Java

  import org.apache.spark.api.java.JavaRDD;
  import org.apache.spark.api.java.function.Function;
  import org.apache.spark.api.java.function.MapFunction;
  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Row;
  import org.apache.spark.sql.Encoder;
  import org.apache.spark.sql.Encoders;

  // Create an RDD of Person objects from a text file
  JavaRDD<Person> peopleRDD = spark.read()
    .textFile("examples/src/main/resources/people.txt")
    .javaRDD()
    .map(line -> {
      String[] parts = line.split(",");
      Person person = new Person();
      person.setName(parts[0]);
      person.setAge(Integer.parseInt(parts[1].trim()));
      return person;
    });

  // Apply a schema to an RDD of JavaBeans to get a DataFrame
  Dataset<Row> peopleDF = spark.createDataFrame(peopleRDD, Person.class);
  // Register the DataFrame as a temporary view
  peopleDF.createOrReplaceTempView("people");

  // SQL statements can be run by using the sql methods provided by spark
  Dataset<Row> teenagersDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19");

  // The columns of a row in the result can be accessed by field index
  Encoder<String> stringEncoder = Encoders.STRING();
  Dataset<String> teenagerNamesByIndexDF = teenagersDF.map(
      (MapFunction<Row, String>) row -> "Name: " + row.getString(0),
      stringEncoder);
  teenagerNamesByIndexDF.show();
  // +------------+
  // |       value|
  // +------------+
  // |Name: Justin|
  // +------------+

  // or by field name
  Dataset<String> teenagerNamesByFieldDF = teenagersDF.map(
      (MapFunction<Row, String>) row -> "Name: " + row.<String>getAs("name"),
      stringEncoder);
  teenagerNamesByFieldDF.show();
  // +------------+
  // |       value|
  // +------------+
  // |Name: Justin|
  // +------------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 文件。


**Python**

Spark SQL can convert an RDD of Row objects to a DataFrame, inferring the datatypes. Rows are constructed by passing a list of key/value pairs as kwargs to the Row class. The keys of this list define the column names of the table, and the types are inferred by sampling the whole dataset, similar to the inference that is performed on JSON files.

.. code-block:: Python

  from pyspark.sql import Row

  sc = spark.sparkContext

  # Load a text file and convert each line to a Row.
  lines = sc.textFile("examples/src/main/resources/people.txt")
  parts = lines.map(lambda l: l.split(","))
  people = parts.map(lambda p: Row(name=p[0], age=int(p[1])))

  # Infer the schema, and register the DataFrame as a table.
  schemaPeople = spark.createDataFrame(people)
  schemaPeople.createOrReplaceTempView("people")

  # SQL can be run over DataFrames that have been registered as a table.
  teenagers = spark.sql("SELECT name FROM people WHERE age >= 13 AND age <= 19")

  # The results of SQL queries are Dataframe objects.
  # rdd returns the content as an :class:`pyspark.RDD` of :class:`Row`.
  teenNames = teenagers.rdd.map(lambda p: "Name: " + p.name).collect()
  for name in teenNames:
      print(name)
  # Name: Justin

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/basic.py" 文件。


Programmatically Specifying the Schema
-----------------------------------------

**Scala**

When case classes cannot be defined ahead of time (for example, the structure of records is encoded in a string, or a text dataset will be parsed and fields will be projected differently for different users), a DataFrame can be created programmatically with three steps.

Create an RDD of Rows from the original RDD;
Create the schema represented by a StructType matching the structure of Rows in the RDD created in Step 1.
Apply the schema to the RDD of Rows via createDataFrame method provided by SparkSession.
For example:

.. code-block:: Scala

  import org.apache.spark.sql.types._

  // Create an RDD
  val peopleRDD = spark.sparkContext.textFile("examples/src/main/resources/people.txt")

  // The schema is encoded in a string
  val schemaString = "name age"

  // Generate the schema based on the string of schema
  val fields = schemaString.split(" ")
    .map(fieldName => StructField(fieldName, StringType, nullable = true))
  val schema = StructType(fields)

  // Convert records of the RDD (people) to Rows
  val rowRDD = peopleRDD
    .map(_.split(","))
    .map(attributes => Row(attributes(0), attributes(1).trim))

  // Apply the schema to the RDD
  val peopleDF = spark.createDataFrame(rowRDD, schema)

  // Creates a temporary view using the DataFrame
  peopleDF.createOrReplaceTempView("people")

  // SQL can be run over a temporary view created using DataFrames
  val results = spark.sql("SELECT name FROM people")

  // The results of SQL queries are DataFrames and support all the normal RDD operations
  // The columns of a row in the result can be accessed by field index or by field name
  results.map(attributes => "Name: " + attributes(0)).show()
  // +-------------+
  // |        value|
  // +-------------+
  // |Name: Michael|
  // |   Name: Andy|
  // | Name: Justin|
  // +-------------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 文件。


**Java**

When JavaBean classes cannot be defined ahead of time (for example, the structure of records is encoded in a string, or a text dataset will be parsed and fields will be projected differently for different users), a Dataset<Row> can be created programmatically with three steps.

Create an RDD of Rows from the original RDD;
Create the schema represented by a StructType matching the structure of Rows in the RDD created in Step 1.
Apply the schema to the RDD of Rows via createDataFrame method provided by SparkSession.
For example:

.. code-block:: Java

  import java.util.ArrayList;
  import java.util.List;

  import org.apache.spark.api.java.JavaRDD;
  import org.apache.spark.api.java.function.Function;

  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Row;

  import org.apache.spark.sql.types.DataTypes;
  import org.apache.spark.sql.types.StructField;
  import org.apache.spark.sql.types.StructType;

  // Create an RDD
  JavaRDD<String> peopleRDD = spark.sparkContext()
    .textFile("examples/src/main/resources/people.txt", 1)
    .toJavaRDD();

  // The schema is encoded in a string
  String schemaString = "name age";

  // Generate the schema based on the string of schema
  List<StructField> fields = new ArrayList<>();
  for (String fieldName : schemaString.split(" ")) {
    StructField field = DataTypes.createStructField(fieldName, DataTypes.StringType, true);
    fields.add(field);
  }
  StructType schema = DataTypes.createStructType(fields);

  // Convert records of the RDD (people) to Rows
  JavaRDD<Row> rowRDD = peopleRDD.map((Function<String, Row>) record -> {
    String[] attributes = record.split(",");
    return RowFactory.create(attributes[0], attributes[1].trim());
  });

  // Apply the schema to the RDD
  Dataset<Row> peopleDataFrame = spark.createDataFrame(rowRDD, schema);

  // Creates a temporary view using the DataFrame
  peopleDataFrame.createOrReplaceTempView("people");

  // SQL can be run over a temporary view created using DataFrames
  Dataset<Row> results = spark.sql("SELECT name FROM people");

  // The results of SQL queries are DataFrames and support all the normal RDD operations
  // The columns of a row in the result can be accessed by field index or by field name
  Dataset<String> namesDS = results.map(
      (MapFunction<Row, String>) row -> "Name: " + row.getString(0),
      Encoders.STRING());
  namesDS.show();
  // +-------------+
  // |        value|
  // +-------------+
  // |Name: Michael|
  // |   Name: Andy|
  // | Name: Justin|
  // +-------------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 文件。


**Python**

When a dictionary of kwargs cannot be defined ahead of time (for example, the structure of records is encoded in a string, or a text dataset will be parsed and fields will be projected differently for different users), a DataFrame can be created programmatically with three steps.

Create an RDD of tuples or lists from the original RDD;
Create the schema represented by a StructType matching the structure of tuples or lists in the RDD created in the step 1.
Apply the schema to the RDD via createDataFrame method provided by SparkSession.
For example:

.. code-block:: Python

  # Import data types
  from pyspark.sql.types import *

  sc = spark.sparkContext

  # Load a text file and convert each line to a Row.
  lines = sc.textFile("examples/src/main/resources/people.txt")
  parts = lines.map(lambda l: l.split(","))
  # Each line is converted to a tuple.
  people = parts.map(lambda p: (p[0], p[1].strip()))

  # The schema is encoded in a string.
  schemaString = "name age"

  fields = [StructField(field_name, StringType(), True) for field_name in schemaString.split()]
  schema = StructType(fields)

  # Apply the schema to the RDD.
  schemaPeople = spark.createDataFrame(people, schema)

  # Creates a temporary view using the DataFrame
  schemaPeople.createOrReplaceTempView("people")

  # SQL can be run over DataFrames that have been registered as a table.
  results = spark.sql("SELECT name FROM people")

  results.show()
  # +-------+
  # |   name|
  # +-------+
  # |Michael|
  # |   Andy|
  # | Justin|
  # +-------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/basic.py" 文件。


聚合
==============================

The built-in DataFrames functions provide common aggregations such as count(), countDistinct(), avg(), max(), min(), etc. While those functions are designed for DataFrames, Spark SQL also has type-safe versions for some of them in Scala and Java to work with strongly typed Datasets. Moreover, users are not limited to the predefined aggregate functions and can create their own.

Untyped User-Defined Aggregate Functions
----------------------------------------------

Users have to extend the UserDefinedAggregateFunction abstract class to implement a custom untyped aggregate function. For example, a user-defined average can look like:

**Scala**

.. code-block:: Scala

  import org.apache.spark.sql.expressions.MutableAggregationBuffer
  import org.apache.spark.sql.expressions.UserDefinedAggregateFunction
  import org.apache.spark.sql.types._
  import org.apache.spark.sql.Row
  import org.apache.spark.sql.SparkSession

  object MyAverage extends UserDefinedAggregateFunction {
    // Data types of input arguments of this aggregate function
    def inputSchema: StructType = StructType(StructField("inputColumn", LongType) :: Nil)
    // Data types of values in the aggregation buffer
    def bufferSchema: StructType = {
      StructType(StructField("sum", LongType) :: StructField("count", LongType) :: Nil)
    }
    // The data type of the returned value
    def dataType: DataType = DoubleType
    // Whether this function always returns the same output on the identical input
    def deterministic: Boolean = true
    // Initializes the given aggregation buffer. The buffer itself is a `Row` that in addition to
    // standard methods like retrieving a value at an index (e.g., get(), getBoolean()), provides
    // the opportunity to update its values. Note that arrays and maps inside the buffer are still
    // immutable.
    def initialize(buffer: MutableAggregationBuffer): Unit = {
      buffer(0) = 0L
      buffer(1) = 0L
    }
    // Updates the given aggregation buffer `buffer` with new input data from `input`
    def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
      if (!input.isNullAt(0)) {
        buffer(0) = buffer.getLong(0) + input.getLong(0)
        buffer(1) = buffer.getLong(1) + 1
      }
    }
    // Merges two aggregation buffers and stores the updated buffer values back to `buffer1`
    def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
      buffer1(0) = buffer1.getLong(0) + buffer2.getLong(0)
      buffer1(1) = buffer1.getLong(1) + buffer2.getLong(1)
    }
    // Calculates the final result
    def evaluate(buffer: Row): Double = buffer.getLong(0).toDouble / buffer.getLong(1)
  }

  // Register the function to access it
  spark.udf.register("myAverage", MyAverage)

  val df = spark.read.json("examples/src/main/resources/employees.json")
  df.createOrReplaceTempView("employees")
  df.show()
  // +-------+------+
  // |   name|salary|
  // +-------+------+
  // |Michael|  3000|
  // |   Andy|  4500|
  // | Justin|  3500|
  // |  Berta|  4000|
  // +-------+------+

  val result = spark.sql("SELECT myAverage(salary) as average_salary FROM employees")
  result.show()
  // +--------------+
  // |average_salary|
  // +--------------+
  // |        3750.0|
  // +--------------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/UserDefinedUntypedAggregation.scala" 文件。


**Java**

.. code-block:: Java

  import java.util.ArrayList;
  import java.util.List;

  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Row;
  import org.apache.spark.sql.SparkSession;
  import org.apache.spark.sql.expressions.MutableAggregationBuffer;
  import org.apache.spark.sql.expressions.UserDefinedAggregateFunction;
  import org.apache.spark.sql.types.DataType;
  import org.apache.spark.sql.types.DataTypes;
  import org.apache.spark.sql.types.StructField;
  import org.apache.spark.sql.types.StructType;

  public static class MyAverage extends UserDefinedAggregateFunction {

    private StructType inputSchema;
    private StructType bufferSchema;

    public MyAverage() {
      List<StructField> inputFields = new ArrayList<>();
      inputFields.add(DataTypes.createStructField("inputColumn", DataTypes.LongType, true));
      inputSchema = DataTypes.createStructType(inputFields);

      List<StructField> bufferFields = new ArrayList<>();
      bufferFields.add(DataTypes.createStructField("sum", DataTypes.LongType, true));
      bufferFields.add(DataTypes.createStructField("count", DataTypes.LongType, true));
      bufferSchema = DataTypes.createStructType(bufferFields);
    }
    // Data types of input arguments of this aggregate function
    public StructType inputSchema() {
      return inputSchema;
    }
    // Data types of values in the aggregation buffer
    public StructType bufferSchema() {
      return bufferSchema;
    }
    // The data type of the returned value
    public DataType dataType() {
      return DataTypes.DoubleType;
    }
    // Whether this function always returns the same output on the identical input
    public boolean deterministic() {
      return true;
    }
    // Initializes the given aggregation buffer. The buffer itself is a `Row` that in addition to
    // standard methods like retrieving a value at an index (e.g., get(), getBoolean()), provides
    // the opportunity to update its values. Note that arrays and maps inside the buffer are still
    // immutable.
    public void initialize(MutableAggregationBuffer buffer) {
      buffer.update(0, 0L);
      buffer.update(1, 0L);
    }
    // Updates the given aggregation buffer `buffer` with new input data from `input`
    public void update(MutableAggregationBuffer buffer, Row input) {
      if (!input.isNullAt(0)) {
        long updatedSum = buffer.getLong(0) + input.getLong(0);
        long updatedCount = buffer.getLong(1) + 1;
        buffer.update(0, updatedSum);
        buffer.update(1, updatedCount);
      }
    }
    // Merges two aggregation buffers and stores the updated buffer values back to `buffer1`
    public void merge(MutableAggregationBuffer buffer1, Row buffer2) {
      long mergedSum = buffer1.getLong(0) + buffer2.getLong(0);
      long mergedCount = buffer1.getLong(1) + buffer2.getLong(1);
      buffer1.update(0, mergedSum);
      buffer1.update(1, mergedCount);
    }
    // Calculates the final result
    public Double evaluate(Row buffer) {
      return ((double) buffer.getLong(0)) / buffer.getLong(1);
    }
  }

  // Register the function to access it
  spark.udf().register("myAverage", new MyAverage());

  Dataset<Row> df = spark.read().json("examples/src/main/resources/employees.json");
  df.createOrReplaceTempView("employees");
  df.show();
  // +-------+------+
  // |   name|salary|
  // +-------+------+
  // |Michael|  3000|
  // |   Andy|  4500|
  // | Justin|  3500|
  // |  Berta|  4000|
  // +-------+------+

  Dataset<Row> result = spark.sql("SELECT myAverage(salary) as average_salary FROM employees");
  result.show();
  // +--------------+
  // |average_salary|
  // +--------------+
  // |        3750.0|
  // +--------------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaUserDefinedUntypedAggregation.java" 文件。


Type-Safe User-Defined Aggregate Functions
----------------------------------------------

User-defined aggregations for strongly typed Datasets revolve around the Aggregator abstract class. For example, a type-safe user-defined average can look like:

**Scala**

.. code-block:: Scala

  import org.apache.spark.sql.expressions.Aggregator
  import org.apache.spark.sql.Encoder
  import org.apache.spark.sql.Encoders
  import org.apache.spark.sql.SparkSession

  case class Employee(name: String, salary: Long)
  case class Average(var sum: Long, var count: Long)

  object MyAverage extends Aggregator[Employee, Average, Double] {
    // A zero value for this aggregation. Should satisfy the property that any b + zero = b
    def zero: Average = Average(0L, 0L)
    // Combine two values to produce a new value. For performance, the function may modify `buffer`
    // and return it instead of constructing a new object
    def reduce(buffer: Average, employee: Employee): Average = {
      buffer.sum += employee.salary
      buffer.count += 1
      buffer
    }
    // Merge two intermediate values
    def merge(b1: Average, b2: Average): Average = {
      b1.sum += b2.sum
      b1.count += b2.count
      b1
    }
    // Transform the output of the reduction
    def finish(reduction: Average): Double = reduction.sum.toDouble / reduction.count
    // Specifies the Encoder for the intermediate value type
    def bufferEncoder: Encoder[Average] = Encoders.product
    // Specifies the Encoder for the final output value type
    def outputEncoder: Encoder[Double] = Encoders.scalaDouble
  }

  val ds = spark.read.json("examples/src/main/resources/employees.json").as[Employee]
  ds.show()
  // +-------+------+
  // |   name|salary|
  // +-------+------+
  // |Michael|  3000|
  // |   Andy|  4500|
  // | Justin|  3500|
  // |  Berta|  4000|
  // +-------+------+

  // Convert the function to a `TypedColumn` and give it a name
  val averageSalary = MyAverage.toColumn.name("average_salary")
  val result = ds.select(averageSalary)
  result.show()
  // +--------------+
  // |average_salary|
  // +--------------+
  // |        3750.0|
  // +--------------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/UserDefinedTypedAggregation.scala" 文件。


**Java**

.. code-block:: Java

  import java.io.Serializable;

  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Encoder;
  import org.apache.spark.sql.Encoders;
  import org.apache.spark.sql.SparkSession;
  import org.apache.spark.sql.TypedColumn;
  import org.apache.spark.sql.expressions.Aggregator;

  public static class Employee implements Serializable {
    private String name;
    private long salary;

    // Constructors, getters, setters...

  }

  public static class Average implements Serializable  {
    private long sum;
    private long count;

    // Constructors, getters, setters...

  }

  public static class MyAverage extends Aggregator<Employee, Average, Double> {
    // A zero value for this aggregation. Should satisfy the property that any b + zero = b
    public Average zero() {
      return new Average(0L, 0L);
    }
    // Combine two values to produce a new value. For performance, the function may modify `buffer`
    // and return it instead of constructing a new object
    public Average reduce(Average buffer, Employee employee) {
      long newSum = buffer.getSum() + employee.getSalary();
      long newCount = buffer.getCount() + 1;
      buffer.setSum(newSum);
      buffer.setCount(newCount);
      return buffer;
    }
    // Merge two intermediate values
    public Average merge(Average b1, Average b2) {
      long mergedSum = b1.getSum() + b2.getSum();
      long mergedCount = b1.getCount() + b2.getCount();
      b1.setSum(mergedSum);
      b1.setCount(mergedCount);
      return b1;
    }
    // Transform the output of the reduction
    public Double finish(Average reduction) {
      return ((double) reduction.getSum()) / reduction.getCount();
    }
    // Specifies the Encoder for the intermediate value type
    public Encoder<Average> bufferEncoder() {
      return Encoders.bean(Average.class);
    }
    // Specifies the Encoder for the final output value type
    public Encoder<Double> outputEncoder() {
      return Encoders.DOUBLE();
    }
  }

  Encoder<Employee> employeeEncoder = Encoders.bean(Employee.class);
  String path = "examples/src/main/resources/employees.json";
  Dataset<Employee> ds = spark.read().json(path).as(employeeEncoder);
  ds.show();
  // +-------+------+
  // |   name|salary|
  // +-------+------+
  // |Michael|  3000|
  // |   Andy|  4500|
  // | Justin|  3500|
  // |  Berta|  4000|
  // +-------+------+

  MyAverage myAverage = new MyAverage();
  // Convert the function to a `TypedColumn` and give it a name
  TypedColumn<Employee, Double> averageSalary = myAverage.toColumn().name("average_salary");
  Dataset<Double> result = ds.select(averageSalary);
  result.show();
  // +--------------+
  // |average_salary|
  // +--------------+
  // |        3750.0|
  // +--------------+

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaUserDefinedTypedAggregation.java" 文件。


*****************
数据源
*****************

Spark SQL supports operating on a variety of data sources through the DataFrame interface. A DataFrame can be operated on using relational transformations and can also be used to create a temporary view. Registering a DataFrame as a temporary view allows you to run SQL queries over its data. This section describes the general methods for loading and saving data using the Spark Data Sources and then goes into specific options that are available for the built-in data sources.


Generic Load/Save Functions
==============================

In the simplest form, the default data source (parquet unless otherwise configured by spark.sql.sources.default) will be used for all operations.

**Scala**

.. code-block:: Scala

  val usersDF = spark.read.load("examples/src/main/resources/users.parquet")
  usersDF.select("name", "favorite_color").write.save("namesAndFavColors.parquet")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。

**Java**

.. code-block:: Java

  Dataset<Row> usersDF = spark.read().load("examples/src/main/resources/users.parquet");
  usersDF.select("name", "favorite_color").write().save("namesAndFavColors.parquet");

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。

**Python**

.. code-block:: Python

  df = spark.read.load("examples/src/main/resources/users.parquet")
  df.select("name", "favorite_color").write.save("namesAndFavColors.parquet")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。

**R**

.. code-block:: R

  df <- read.df("examples/src/main/resources/users.parquet")
  write.df(select(df, "name", "favorite_color"), "namesAndFavColors.parquet")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。


手动指定选项
---------------------------------

You can also manually specify the data source that will be used along with any extra options that you would like to pass to the data source. Data sources are specified by their fully qualified name (i.e., org.apache.spark.sql.parquet), but for built-in sources you can also use their short names (json, parquet, jdbc, orc, libsvm, csv, text). DataFrames loaded from any data source type can be converted into other types using this syntax.

**Scala**

.. code-block:: Scala

  val peopleDF = spark.read.format("json").load("examples/src/main/resources/people.json")
  peopleDF.select("name", "age").write.format("parquet").save("namesAndAges.parquet")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。

**Java**

.. code-block:: Java

  Dataset<Row> peopleDF =
    spark.read().format("json").load("examples/src/main/resources/people.json");
  peopleDF.select("name", "age").write().format("parquet").save("namesAndAges.parquet");

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。

**Python**

.. code-block:: Python

  df = spark.read.load("examples/src/main/resources/people.json", format="json")
  df.select("name", "age").write.save("namesAndAges.parquet", format="parquet")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。

**R**

.. code-block:: R

  df <- read.df("examples/src/main/resources/people.json", "json")
  namesAndAges <- select(df, "name", "age")
  write.df(namesAndAges, "namesAndAges.parquet", "parquet")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。


直接在文件上运行 SQL
---------------------------------

Instead of using read API to load a file into DataFrame and query it, you can also query that file directly with SQL.

**Scala**

.. code-block:: Scala

  val sqlDF = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。


**Java**

.. code-block:: Java

  Dataset<Row> sqlDF =
    spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`");

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。

**Python**

.. code-block:: Python

  df = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。

**R**

.. code-block:: R

  df <- sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。


Save Modes
---------------------------------

Save operations can optionally take a SaveMode, that specifies how to handle existing data if present. It is important to realize that these save modes do not utilize any locking and are not atomic. Additionally, when performing an Overwrite, the data will be deleted before writing out the new data.

==================================      ====================      =============
Scala/Java                              Any Language	            Meaning
==================================      ====================      =============
SaveMode.ErrorIfExists(default)	        "error" (default)	        When saving a DataFrame to a data source, if data already exists, an exception is expected to be thrown.
SaveMode.Append	                        "append"	                When saving a DataFrame to a data source, if data/table already exists, contents of the DataFrame are expected to be appended to existing data.
SaveMode.Overwrite	                    "overwrite"	              Overwrite mode means that when saving a DataFrame to a data source, if data/table already exists, existing data is expected to be overwritten by the contents of the DataFrame.
SaveMode.Ignore	                        "ignore"	                Ignore mode means that when saving a DataFrame to a data source, if data already exists, the save operation is expected to not save the contents of the DataFrame and to not change the existing data. This is similar to a CREATE TABLE IF NOT EXISTS in SQL.
==================================      ====================      =============

Saving to Persistent Tables
---------------------------------

DataFrames can also be saved as persistent tables into Hive metastore using the saveAsTable command. Notice that an existing Hive deployment is not necessary to use this feature. Spark will create a default local Hive metastore (using Derby) for you. Unlike the createOrReplaceTempView command, saveAsTable will materialize the contents of the DataFrame and create a pointer to the data in the Hive metastore. Persistent tables will still exist even after your Spark program has restarted, as long as you maintain your connection to the same metastore. A DataFrame for a persistent table can be created by calling the table method on a SparkSession with the name of the table.

For file-based data source, e.g. text, parquet, json, etc. you can specify a custom table path via the path option, e.g. df.write.option("path", "/some/path").saveAsTable("t"). When the table is dropped, the custom table path will not be removed and the table data is still there. If no custom table path is specified, Spark will write data to a default table path under the warehouse directory. When the table is dropped, the default table path will be removed too.

Starting from Spark 2.1, persistent datasource tables have per-partition metadata stored in the Hive metastore. This brings several benefits:

Since the metastore can return only necessary partitions for a query, discovering all the partitions on the first query to the table is no longer needed.
Hive DDLs such as ALTER TABLE PARTITION ... SET LOCATION are now available for tables created with the Datasource API.
Note that partition information is not gathered by default when creating external datasource tables (those with a path option). To sync the partition information in the metastore, you can invoke MSCK REPAIR TABLE.


Bucketing, Sorting and Partitioning
----------------------------------------

For file-based data source, it is also possible to bucket and sort or partition the output. Bucketing and sorting are applicable only to persistent tables:

**Scala**

.. code-block:: Scala

  peopleDF.write.bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。

while partitioning can be used with both save and saveAsTable when using the Dataset APIs.

.. code-block:: Scala

  usersDF.write.partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。
It is possible to use both partitioning and bucketing for a single table:

.. code-block:: Scala

  peopleDF
    .write
    .partitionBy("favorite_color")
    .bucketBy(42, "name")
    .saveAsTable("people_partitioned_bucketed")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。
partitionBy creates a directory structure as described in the Partition Discovery section. Thus, it has limited applicability to columns with high cardinality. In contrast bucketBy distributes data across a fixed number of buckets and can be used when a number of unique values is unbounded.

**Java**

.. code-block:: Java

  peopleDF.write().bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed");

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。
while partitioning can be used with both save and saveAsTable when using the Dataset APIs.

.. code-block:: Java

  usersDF
    .write()
    .partitionBy("favorite_color")
    .format("parquet")
    .save("namesPartByColor.parquet");

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。
It is possible to use both partitioning and bucketing for a single table:

.. code-block:: Java

  peopleDF
    .write()
    .partitionBy("favorite_color")
    .bucketBy(42, "name")
    .saveAsTable("people_partitioned_bucketed");

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。
partitionBy creates a directory structure as described in the Partition Discovery section. Thus, it has limited applicability to columns with high cardinality. In contrast bucketBy distributes data across a fixed number of buckets and can be used when a number of unique values is unbounded.


**Python**

.. code-block:: Python

  df.write.bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。

while partitioning can be used with both save and saveAsTable when using the Dataset APIs.

.. code-block:: Python

  df.write.partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet")

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。

It is possible to use both partitioning and bucketing for a single table:

.. code-block:: Python

  df = spark.read.parquet("examples/src/main/resources/users.parquet")
  (df
      .write
      .partitionBy("favorite_color")
      .bucketBy(42, "name")
      .saveAsTable("people_partitioned_bucketed"))

完整的示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。
partitionBy creates a directory structure as described in the Partition Discovery section. Thus, it has limited applicability to columns with high cardinality. In contrast bucketBy distributes data across a fixed number of buckets and can be used when a number of unique values is unbounded.

**Sql**

.. code-block:: SQL

  CREATE TABLE users_bucketed_by_name(
    name STRING,
    favorite_color STRING,
    favorite_numbers array<integer>
  ) USING parquet
  CLUSTERED BY(name) INTO 42 BUCKETS;

while partitioning can be used with both save and saveAsTable when using the Dataset APIs.

.. code-block:: SQL

  CREATE TABLE users_by_favorite_color(
    name STRING,
    favorite_color STRING,
    favorite_numbers array<integer>
  ) USING csv PARTITIONED BY(favorite_color);

It is possible to use both partitioning and bucketing for a single table:

.. code-block:: SQL

  CREATE TABLE users_bucketed_and_partitioned(
    name STRING,
    favorite_color STRING,
    favorite_numbers array<integer>
  ) USING parquet
  PARTITIONED BY (favorite_color)
  CLUSTERED BY(name) SORTED BY (favorite_numbers) INTO 42 BUCKETS;



partitionBy creates a directory structure as described in the Partition Discovery section. Thus, it has limited applicability to columns with high cardinality. In contrast bucketBy distributes data across a fixed number of buckets and can be used when a number of unique values is unbounded.


Parquet 文件
==============================

Parquet 是一种列式存储格式，很多其它的数据处理系统都支持它。Spark SQL 提供了对 Parquet 文件的读写支持，而且 Parquet 文件能够自动保存原始数据的 schema。写 Parquet 文件的时候，所有列都自动地转化成 nullable，以便向后兼容。

编程方式加载数据
-----------------------

仍然使用上面例子中的数据：

**Scala**

.. code-block:: Scala

  // Encoders for most common types are automatically provided by importing spark.implicits._
  import spark.implicits._

  val peopleDF = spark.read.json("examples/src/main/resources/people.json")

  // DataFrames can be saved as Parquet files, maintaining the schema information
  peopleDF.write.parquet("people.parquet")

  // Read in the parquet file created above
  // Parquet files are self-describing so the schema is preserved
  // The result of loading a Parquet file is also a DataFrame
  val parquetFileDF = spark.read.parquet("people.parquet")

  // Parquet files can also be used to create a temporary view and then used in SQL statements
  parquetFileDF.createOrReplaceTempView("parquetFile")
  val namesDF = spark.sql("SELECT name FROM parquetFile WHERE age BETWEEN 13 AND 19")
  namesDF.map(attributes => "Name: " + attributes(0)).show()
  // +------------+
  // |       value|
  // +------------+
  // |Name: Justin|
  // +------------+

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。

**Java**

.. code-block:: Java

  import org.apache.spark.api.java.function.MapFunction;
  import org.apache.spark.sql.Encoders;
  // import org.apache.spark.sql.Encoders;
  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Row;

  Dataset<Row> peopleDF = spark.read().json("examples/src/main/resources/people.json");

  // DataFrames can be saved as Parquet files, maintaining the schema information
  peopleDF.write().parquet("people.parquet");

  // Read in the Parquet file created above.
  // Parquet files are self-describing so the schema is preserved
  // The result of loading a parquet file is also a DataFrame
  Dataset<Row> parquetFileDF = spark.read().parquet("people.parquet");

  // Parquet files can also be used to create a temporary view and then used in SQL statements
  parquetFileDF.createOrReplaceTempView("parquetFile");
  Dataset<Row> namesDF = spark.sql("SELECT name FROM parquetFile WHERE age BETWEEN 13 AND 19");
  Dataset<String> namesDS = namesDF.map(new MapFunction<Row, String>() {
    public String call(Row row) {
      return "Name: " + row.getString(0);
    }
  }, Encoders.STRING());
  namesDS.show();
  // +------------+
  // |       value|
  // +------------+
  // |Name: Justin|
  // +------------+

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。

**Python**

.. code-block:: Python

  peopleDF = spark.read.json("examples/src/main/resources/people.json")

  # DataFrames can be saved as Parquet files, maintaining the schema information.
  peopleDF.write.parquet("people.parquet")

  # Read in the Parquet file created above.
  # Parquet files are self-describing so the schema is preserved.
  # The result of loading a parquet file is also a DataFrame.
  parquetFile = spark.read.parquet("people.parquet")

  # Parquet files can also be used to create a temporary view and then used in SQL statements.
  parquetFile.createOrReplaceTempView("parquetFile")
  teenagers = spark.sql("SELECT name FROM parquetFile WHERE age >= 13 AND age <= 19")
  teenagers.show()
  # +------+
  # |  name|
  # +------+
  # |Justin|
  # +------+

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。

**R**

.. code-block:: R

  df <- read.df("examples/src/main/resources/people.json", "json")

  # SparkDataFrame can be saved as Parquet files, maintaining the schema information.
  write.parquet(df, "people.parquet")

  # Read in the Parquet file created above. Parquet files are self-describing so the schema is preserved.
  # The result of loading a parquet file is also a DataFrame.
  parquetFile <- read.parquet("people.parquet")

  # Parquet files can also be used to create a temporary view and then used in SQL statements.
  createOrReplaceTempView(parquetFile, "parquetFile")
  teenagers <- sql("SELECT name FROM parquetFile WHERE age >= 13 AND age <= 19")
  head(teenagers)
  ##     name
  ## 1 Justin

  # We can also run custom R-UDFs on Spark DataFrames. Here we prefix all the names with "Name:"
  schema <- structType(structField("name", "string"))
  teenNames <- dapply(df, function(p) { cbind(paste("Name:", p$name)) }, schema)
  for (teenName in collect(teenNames)$name) {
    cat(teenName, "\n")
  }
  ## Name: Michael
  ## Name: Andy
  ## Name: Justin

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。

**Sql**

.. code-block:: SQL

  CREATE TEMPORARY VIEW parquetTable
  USING org.apache.spark.sql.parquet
  OPTIONS (
    path "examples/src/main/resources/people.parquet"
  )

  SELECT * FROM parquetTable


分区发现
-----------------------

像Hive这样的系统中，一个常用的优化方式就是表分区。在一个分区表中，数据通常存储在不同的目录中，分区列值被编码到各个分区目录的路径。Parquet数据源现在可以自动发现和推导分区信息。例如，我们可以使用下面的目录结构把之前使用的人口数据存储到一个分区表中，其中2个额外的字段，gender和country，作为分区列：

.. code-block:: TEXT

  path
  └── to
      └── table
          ├── gender=male
          │   ├── ...
          │   │
          │   ├── country=US
          │   │   └── data.parquet
          │   ├── country=CN
          │   │   └── data.parquet
          │   └── ...
          └── gender=female
              ├── ...
              │
              ├── country=US
              │   └── data.parquet
              ├── country=CN
              │   └── data.parquet
              └── ...

通过传递 path/to/table 给 SparkSession.read.parquet 或 SparkSession.read.load, Spark SQL将会自动从路径中提取分区信息。现在返回的DataFrame的schema如下：

.. code-block:: TEXT

  root
  |-- name: string (nullable = true)
  |-- age: long (nullable = true)
  |-- gender: string (nullable = true)
  |-- country: string (nullable = true)

注意，分区列的数据类型是自动推导出来的。目前，分区列只支持数值类型和字符串类型。有时候用户可能不想要自动推导分区列的数据类型，对于这种情况，自动类型推导可以通过 spark.sql.sources.partitionColumnTypeInference.enabled来配置，其默认值是true。当禁用类型推导后，字符串类型将用于分区列类型。

从Spark 1.6.0 版本开始，分区发现默认只查找给定路径下的分区。拿上面的例子来说，如果用户传递 path/to/table/gender=male 给 SparkSession.read.parquet 或者 SparkSession.read.load，那么gender将不会被当作分区列。如果用户想要指定分区发现开始的基础目录，可以在数据源选项中设置basePath。例如，如果把 path/to/table/gender=male作为数据目录，并且将basePath设为 path/to/table，那么gender仍然会最为分区键。

Schema 合并
-----------------------

和 ProtocolBuffer、Avro 以及 Thrift 一样，Parquet也支持 schema 演变。用户可以从一个简单的 schema 开始，逐渐增加所需要的列。这样的话，用户最终会得到多个Parquet文件, 这些文件的schema不同但互相兼容。Parquet数据源目前已经支持自动检测这种情况并合并所有这些文件的schema。

因为schema合并相对来说是一个代价高昂的操作，并且在大多数情况下不需要，所以从Spark 1.5.0 版本开始，默认禁用Schema合并。你可以这样启用这一功能：

1. 当读取Parquet文件时，将数据源选项 mergeSchema设置为true（见下面的示例代码）
2. 或者，将全局SQL选项 spark.sql.parquet.mergeSchema设置为true。

**Scala**

.. code-block:: Scala

  // This is used to implicitly convert an RDD to a DataFrame.
  import spark.implicits._

  // Create a simple DataFrame, store into a partition directory
  val squaresDF = spark.sparkContext.makeRDD(1 to 5).map(i => (i, i * i)).toDF("value", "square")
  squaresDF.write.parquet("data/test_table/key=1")

  // Create another DataFrame in a new partition directory,
  // adding a new column and dropping an existing column
  val cubesDF = spark.sparkContext.makeRDD(6 to 10).map(i => (i, i * i * i)).toDF("value", "cube")
  cubesDF.write.parquet("data/test_table/key=2")

  // Read the partitioned table
  val mergedDF = spark.read.option("mergeSchema", "true").parquet("data/test_table")
  mergedDF.printSchema()

  // The final schema consists of all 3 columns in the Parquet files together
  // with the partitioning column appeared in the partition directory paths
  // root
  // |-- value: int (nullable = true)
  // |-- square: int (nullable = true)
  // |-- cube: int (nullable = true)
  // |-- key : int (nullable = true)

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。

**Java**

.. code-block:: Java

  import java.io.Serializable;
  import java.util.ArrayList;
  import java.util.Arrays;
  import java.util.List;

  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Row;

  public static class Square implements Serializable {
    private int value;
    private int square;

    // Getters and setters...

  }

  public static class Cube implements Serializable {
    private int value;
    private int cube;

    // Getters and setters...

  }

  List<Square> squares = new ArrayList<>();
  for (int value = 1; value <= 5; value++) {
    Square square = new Square();
    square.setValue(value);
    square.setSquare(value * value);
    squares.add(square);
  }

  // Create a simple DataFrame, store into a partition directory
  Dataset<Row> squaresDF = spark.createDataFrame(squares, Square.class);
  squaresDF.write().parquet("data/test_table/key=1");

  List<Cube> cubes = new ArrayList<>();
  for (int value = 6; value <= 10; value++) {
    Cube cube = new Cube();
    cube.setValue(value);
    cube.setCube(value * value * value);
    cubes.add(cube);
  }

  // Create another DataFrame in a new partition directory,
  // adding a new column and dropping an existing column
  Dataset<Row> cubesDF = spark.createDataFrame(cubes, Cube.class);
  cubesDF.write().parquet("data/test_table/key=2");

  // Read the partitioned table
  Dataset<Row> mergedDF = spark.read().option("mergeSchema", true).parquet("data/test_table");
  mergedDF.printSchema();

  // The final schema consists of all 3 columns in the Parquet files together
  // with the partitioning column appeared in the partition directory paths
  // root
  //  |-- value: int (nullable = true)
  //  |-- square: int (nullable = true)
  //  |-- cube: int (nullable = true)
  //  |-- key: int (nullable = true)

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。

**Python**

.. code-block:: Python

  from pyspark.sql import Row

  # spark is from the previous example.
  # Create a simple DataFrame, stored into a partition directory
  sc = spark.sparkContext

  squaresDF = spark.createDataFrame(sc.parallelize(range(1, 6))
                                    .map(lambda i: Row(single=i, double=i ** 2)))
  squaresDF.write.parquet("data/test_table/key=1")

  # Create another DataFrame in a new partition directory,
  # adding a new column and dropping an existing column
  cubesDF = spark.createDataFrame(sc.parallelize(range(6, 11))
                                  .map(lambda i: Row(single=i, triple=i ** 3)))
  cubesDF.write.parquet("data/test_table/key=2")

  # Read the partitioned table
  mergedDF = spark.read.option("mergeSchema", "true").parquet("data/test_table")
  mergedDF.printSchema()

  # The final schema consists of all 3 columns in the Parquet files together
  # with the partitioning column appeared in the partition directory paths.
  # root
  #  |-- double: long (nullable = true)
  #  |-- single: long (nullable = true)
  #  |-- triple: long (nullable = true)
  #  |-- key: integer (nullable = true)

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。

**R**

.. code-block:: R

  df1 <- createDataFrame(data.frame(single=c(12, 29), double=c(19, 23)))
  df2 <- createDataFrame(data.frame(double=c(19, 23), triple=c(23, 18)))

  # Create a simple DataFrame, stored into a partition directory
  write.df(df1, "data/test_table/key=1", "parquet", "overwrite")

  # Create another DataFrame in a new partition directory,
  # adding a new column and dropping an existing column
  write.df(df2, "data/test_table/key=2", "parquet", "overwrite")

  # Read the partitioned table
  df3 <- read.df("data/test_table", "parquet", mergeSchema = "true")
  printSchema(df3)
  # The final schema consists of all 3 columns in the Parquet files together
  # with the partitioning column appeared in the partition directory paths
  ## root
  ##  |-- single: double (nullable = true)
  ##  |-- double: double (nullable = true)
  ##  |-- triple: double (nullable = true)
  ##  |-- key: integer (nullable = true)

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。


Hive metastore Parquet表转换
--------------------------------

当读写Hive metastore Parquet表时，为了达到更好的性能, Spark SQL使用它自己的Parquet支持库，而不是Hive SerDe。这一行为是由 spark.sql.hive.convertMetastoreParquet 这个配置项来控制的，它默认是开启的。

Hive/Parquet Schema调整
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

从表 schema 处理的角度来看, Hive和Parquet有2个关键的不同点：

1. Hive是非大小写敏感的，而Parquet是大小写敏感的。
2. Hive认为所有列都是nullable，而Parquet中为空性是很重要的。

基于以上原因，在将一个Hive metastore Parquet表转换成一个Spark SQL Parquet表的时候，必须要对Hive metastore schema做调整，调整规则如下：

1. 两个schema中字段名称一致的话那么字段类型也必须一致（不考虑为空性）。调整后的字段应该有Parquet端的数据类型，所以为空性也是需要考虑的。
2. 调整后的schema必须完全包含Hive metastore schema中定义的字段。
    * 只出现在Parquet schema中的字段将在调整后的schema中丢弃。
    * 只出现在Hive metastore schema中的字段将作为nullable字段添加到调整后的schema。

元数据刷新
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Spark SQL 会缓存 Parquet 元数据以提高性能。如果启用了Hive metastore Parquet table转换，那么转换后的表的schema也会被缓存起来。如果这些表被Hive或其它外部工具更新, 那么你需要手动地刷新它们以确保元数据一致性。

**Scala**

.. code-block:: Scala

  // spark is an existing SparkSession
  spark.catalog.refreshTable("my_table")

**Java**

.. code-block:: Java

  // spark is an existing SparkSession
  spark.catalog().refreshTable("my_table");

**Python**

.. code-block:: Python

  # spark is an existing HiveContext
  spark.refreshTable("my_table")

**Sql**

.. code-block:: SQL

  REFRESH TABLE my_table;


配置
---------------------------

Parquet配置可以使用 SparkSession 上的 setConf 方法或者使用 SQL 语句中的 SET key=value 命令来完成。

========================================      ========      ========
属性名                                         默认值         含义
========================================      ========      ========
spark.sql.parquet.binaryAsString              false         一些其它的Parquet生产系统, 特别是Impala，Hive以及老版本的Spark SQL，当写Parquet schema时都不区分二进制数据和字符串。这个标识告诉Spark SQL把二进制数据当字符串处理，以兼容老系统。
spark.sql.parquet.int96AsTimestamp            true          一些Parquet生产系统, 特别是Impala和Hive，把时间戳存成INT96。这个标识告诉Spark SQL将INT96数据解析成timestamp，以兼容老系统。
spark.sql.parquet.cacheMetadata               true          开启Parquet schema元数据缓存。可以提升查询静态数据的速度。
spark.sql.parquet.compression.codec           gzip          当写Parquet文件时，设置压缩编码格式。可接受的值有：uncompressed, snappy, gzip, lzo
spark.sql.parquet.filterPushdown              true          当设置为true时启用Parquet过滤器下推优化
spark.sql.hive.convertMetastoreParquet        true          当设置为false时，Spark SQL将使用Hive SerDe，而不是内建的Parquet tables支持
spark.sql.parquet.mergeSchema                 false         如果设为true，那么Parquet数据源将会合并所有数据文件的schema，否则，从汇总文件中选取schema，如果没有汇总文件，则随机选取一个数据文件）
========================================      ========      ========


JSON Datasets
==============================

**Scala**

Spark SQL可以自动推导JSON数据集的schema并且将其加载为一个 Dataset[Row]。这种转换可以在一个包含String的RDD或一个JSON文件上使用SparkSession.read.json() 来完成。

注意，作为json文件提供的文件并不是一个典型的JSON文件。JSON文件的每一行必须包含一个独立的、完整有效的JSON对象。因此，一个常规的多行json文件经常会加载失败。

.. code-block:: Scala

  // Primitive types (Int, String, etc) and Product types (case classes) encoders are
  // supported by importing this when creating a Dataset.
  import spark.implicits._

  // A JSON dataset is pointed to by path.
  // The path can be either a single text file or a directory storing text files
  val path = "examples/src/main/resources/people.json"
  val peopleDF = spark.read.json(path)

  // The inferred schema can be visualized using the printSchema() method
  peopleDF.printSchema()
  // root
  //  |-- age: long (nullable = true)
  //  |-- name: string (nullable = true)

  // Creates a temporary view using the DataFrame
  peopleDF.createOrReplaceTempView("people")

  // SQL statements can be run by using the sql methods provided by spark
  val teenagerNamesDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19")
  teenagerNamesDF.show()
  // +------+
  // |  name|
  // +------+
  // |Justin|
  // +------+

  // Alternatively, a DataFrame can be created for a JSON dataset represented by
  // a Dataset[String] storing one JSON object per string
  val otherPeopleDataset = spark.createDataset(
    """{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}""" :: Nil)
  val otherPeople = spark.read.json(otherPeopleDataset)
  otherPeople.show()
  // +---------------+----+
  // |        address|name|
  // +---------------+----+
  // |[Columbus,Ohio]| Yin|
  // +---------------+----+

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。

**Java**

Spark SQL 可以自动推导 JSON 数据集的 schema 并且将其加载为一个 Dataset<Row>. 这种转换可以在一个包含String的RDD或一个JSON文件上使用SparkSession.read.json() 来完成。

注意，作为json文件提供的文件并不是一个典型的JSON文件。JSON文件的每一行必须包含一个独立的、完整有效的JSON对象。因此，一个常规的多行json文件经常会加载失败。

.. code-block:: Java

  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Row;

  // A JSON dataset is pointed to by path.
  // The path can be either a single text file or a directory storing text files
  Dataset<Row> people = spark.read().json("examples/src/main/resources/people.json");

  // The inferred schema can be visualized using the printSchema() method
  people.printSchema();
  // root
  //  |-- age: long (nullable = true)
  //  |-- name: string (nullable = true)

  // Creates a temporary view using the DataFrame
  people.createOrReplaceTempView("people");

  // SQL statements can be run by using the sql methods provided by spark
  Dataset<Row> namesDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19");
  namesDF.show();
  // +------+
  // |  name|
  // +------+
  // |Justin|
  // +------+

  // Alternatively, a DataFrame can be created for a JSON dataset represented by
  // a Dataset<String> storing one JSON object per string.
  List<String> jsonData = Arrays.asList(
          "{\"name\":\"Yin\",\"address\":{\"city\":\"Columbus\",\"state\":\"Ohio\"}}");
  Dataset<String> anotherPeopleDataset = spark.createDataset(jsonData, Encoders.STRING());
  Dataset<Row> anotherPeople = spark.read().json(anotherPeopleDataset);
  anotherPeople.show();
  // +---------------+----+
  // |        address|name|
  // +---------------+----+
  // |[Columbus,Ohio]| Yin|
  // +---------------+----+

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。

**Python**

Spark SQL可以自动推导JSON数据集的schema并且将其加载为一个 DataFrame。这种转换可以在一个JSON文件上使用SparkSession.read.json 来完成。

注意，作为 json 文件提供的文件并不是一个典型的 JSON 文件。JSON 文件的每一行必须包含一个独立的、完整有效的JSON对象。因此，一个常规的多行json文件经常会加载失败。

.. code-block:: Python

  # spark is from the previous example.
  sc = spark.sparkContext

  # A JSON dataset is pointed to by path.
  # The path can be either a single text file or a directory storing text files
  path = "examples/src/main/resources/people.json"
  peopleDF = spark.read.json(path)

  # The inferred schema can be visualized using the printSchema() method
  peopleDF.printSchema()
  # root
  #  |-- age: long (nullable = true)
  #  |-- name: string (nullable = true)

  # Creates a temporary view using the DataFrame
  peopleDF.createOrReplaceTempView("people")

  # SQL statements can be run by using the sql methods provided by spark
  teenagerNamesDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19")
  teenagerNamesDF.show()
  # +------+
  # |  name|
  # +------+
  # |Justin|
  # +------+

  # Alternatively, a DataFrame can be created for a JSON dataset represented by
  # an RDD[String] storing one JSON object per string
  jsonStrings = ['{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}']
  otherPeopleRDD = sc.parallelize(jsonStrings)
  otherPeople = spark.read.json(otherPeopleRDD)
  otherPeople.show()
  # +---------------+----+
  # |        address|name|
  # +---------------+----+
  # |[Columbus,Ohio]| Yin|
  # +---------------+----+

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。

**R**

Spark SQL can automatically infer the schema of a JSON dataset and load it as a DataFrame. using the read.json() function, which loads data from a directory of JSON files where each line of the files is a JSON object.

Note that the file that is offered as a json file is not a typical JSON file. Each line must contain a separate, self-contained valid JSON object. For more information, please see JSON Lines text format, also called newline-delimited JSON.

For a regular multi-line JSON file, set a named parameter multiLine to TRUE.

.. code-block:: R

  # A JSON dataset is pointed to by path.
  # The path can be either a single text file or a directory storing text files.
  path <- "examples/src/main/resources/people.json"
  # Create a DataFrame from the file(s) pointed to by path
  people <- read.json(path)

  # The inferred schema can be visualized using the printSchema() method.
  printSchema(people)
  ## root
  ##  |-- age: long (nullable = true)
  ##  |-- name: string (nullable = true)

  # Register this DataFrame as a table.
  createOrReplaceTempView(people, "people")

  # SQL statements can be run by using the sql methods.
  teenagers <- sql("SELECT name FROM people WHERE age >= 13 AND age <= 19")
  head(teenagers)
  ##     name
  ## 1 Justin

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。

**Sql**

.. code-block:: SQL

  CREATE TEMPORARY VIEW jsonTable
  USING org.apache.spark.sql.json
  OPTIONS (
    path "examples/src/main/resources/people.json"
  )

  SELECT * FROM jsonTable


Hive Tables
==============================

Spark SQL 还支持从 Apache Hive 读写数据。然而，由于 Hive 依赖项太多，这些依赖没有包含在默认的 Spark 发行版本中。如果在classpath上配置了Hive依赖，那么 Spark 会自动加载它们。注意，Hive 依赖也必须放到所有的 worker 节点上，因为如果要访问 Hive 中的数据它们需要访问 Hive 序列化和反序列化库(SerDes)。

Hive配置是通过将 hive-site.xml，core-site.xml(用于安全配置)以及 hdfs-site.xml(用于 HDFS 配置)文件放置在 conf/ 目录下来完成的。

如果要使用 Hive, 你必须要实例化一个支持 Hive 的 SparkSession, 包括连接到一个持久化的 Hive metastore, 支持 Hive serdes 以及 Hive 用户自定义函数。即使用户没有安装部署 Hive 也仍然可以启用Hive支持。如果没有在 hive-site.xml 文件中配置, Spark 应用程序启动之后，上下文会自动在当前目录下创建一个 metastore_db 目录并创建一个由 spark.sql.warehouse.dir 配置的、默认值是当前目录下的 spark-warehouse 目录的目录。请注意: 从 Spark 2.0.0 版本开始, hive-site.xml 中的 hive.metastore.warehouse.dir 属性就已经过时了, 你可以使用 spark.sql.warehouse.dir 来指定仓库中数据库的默认存储位置。你可能还需要给启动Spark应用程序的用户赋予写权限。

**Scala**

.. code-block:: Scala

  import java.io.File

  import org.apache.spark.sql.Row
  import org.apache.spark.sql.SparkSession

  case class Record(key: Int, value: String)

  // warehouseLocation points to the default location for managed databases and tables
  val warehouseLocation = new File("spark-warehouse").getAbsolutePath

  val spark = SparkSession
    .builder()
    .appName("Spark Hive Example")
    .config("spark.sql.warehouse.dir", warehouseLocation)
    .enableHiveSupport()
    .getOrCreate()

  import spark.implicits._
  import spark.sql

  sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING) USING hive")
  sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src")

  // Queries are expressed in HiveQL
  sql("SELECT * FROM src").show()
  // +---+-------+
  // |key|  value|
  // +---+-------+
  // |238|val_238|
  // | 86| val_86|
  // |311|val_311|
  // ...

  // Aggregation queries are also supported.
  sql("SELECT COUNT(*) FROM src").show()
  // +--------+
  // |count(1)|
  // +--------+
  // |    500 |
  // +--------+

  // The results of SQL queries are themselves DataFrames and support all normal functions.
  val sqlDF = sql("SELECT key, value FROM src WHERE key < 10 ORDER BY key")

  // The items in DataFrames are of type Row, which allows you to access each column by ordinal.
  val stringsDS = sqlDF.map {
    case Row(key: Int, value: String) => s"Key: $key, Value: $value"
  }
  stringsDS.show()
  // +--------------------+
  // |               value|
  // +--------------------+
  // |Key: 0, Value: val_0|
  // |Key: 0, Value: val_0|
  // |Key: 0, Value: val_0|
  // ...

  // You can also use DataFrames to create temporary views within a SparkSession.
  val recordsDF = spark.createDataFrame((1 to 100).map(i => Record(i, s"val_$i")))
  recordsDF.createOrReplaceTempView("records")

  // Queries can then join DataFrame data with data stored in Hive.
  sql("SELECT * FROM records r JOIN src s ON r.key = s.key").show()
  // +---+------+---+------+
  // |key| value|key| value|
  // +---+------+---+------+
  // |  2| val_2|  2| val_2|
  // |  4| val_4|  4| val_4|
  // |  5| val_5|  5| val_5|
  // ...

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/hive/SparkHiveExample.scala" 文件。

**Java**

.. code-block:: Java

  import java.io.File;
  import java.io.Serializable;
  import java.util.ArrayList;
  import java.util.List;

  import org.apache.spark.api.java.function.MapFunction;
  import org.apache.spark.sql.Dataset;
  import org.apache.spark.sql.Encoders;
  import org.apache.spark.sql.Row;
  import org.apache.spark.sql.SparkSession;

  public static class Record implements Serializable {
    private int key;
    private String value;

    public int getKey() {
      return key;
    }

    public void setKey(int key) {
      this.key = key;
    }

    public String getValue() {
      return value;
    }

    public void setValue(String value) {
      this.value = value;
    }
  }

  // warehouseLocation points to the default location for managed databases and tables
  String warehouseLocation = new File("spark-warehouse").getAbsolutePath();
  SparkSession spark = SparkSession
    .builder()
    .appName("Java Spark Hive Example")
    .config("spark.sql.warehouse.dir", warehouseLocation)
    .enableHiveSupport()
    .getOrCreate();

  spark.sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING) USING hive");
  spark.sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src");

  // Queries are expressed in HiveQL
  spark.sql("SELECT * FROM src").show();
  // +---+-------+
  // |key|  value|
  // +---+-------+
  // |238|val_238|
  // | 86| val_86|
  // |311|val_311|
  // ...

  // Aggregation queries are also supported.
  spark.sql("SELECT COUNT(*) FROM src").show();
  // +--------+
  // |count(1)|
  // +--------+
  // |    500 |
  // +--------+

  // The results of SQL queries are themselves DataFrames and support all normal functions.
  Dataset<Row> sqlDF = spark.sql("SELECT key, value FROM src WHERE key < 10 ORDER BY key");

  // The items in DataFrames are of type Row, which lets you to access each column by ordinal.
  Dataset<String> stringsDS = sqlDF.map(
      (MapFunction<Row, String>) row -> "Key: " + row.get(0) + ", Value: " + row.get(1),
      Encoders.STRING());
  stringsDS.show();
  // +--------------------+
  // |               value|
  // +--------------------+
  // |Key: 0, Value: val_0|
  // |Key: 0, Value: val_0|
  // |Key: 0, Value: val_0|
  // ...

  // You can also use DataFrames to create temporary views within a SparkSession.
  List<Record> records = new ArrayList<>();
  for (int key = 1; key < 100; key++) {
    Record record = new Record();
    record.setKey(key);
    record.setValue("val_" + key);
    records.add(record);
  }
  Dataset<Row> recordsDF = spark.createDataFrame(records, Record.class);
  recordsDF.createOrReplaceTempView("records");

  // Queries can then join DataFrames data with data stored in Hive.
  spark.sql("SELECT * FROM records r JOIN src s ON r.key = s.key").show();
  // +---+------+---+------+
  // |key| value|key| value|
  // +---+------+---+------+
  // |  2| val_2|  2| val_2|
  // |  2| val_2|  2| val_2|
  // |  4| val_4|  4| val_4|
  // ...

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/hive/JavaSparkHiveExample.java" 文件。

**Python**

.. code-block:: Python

  from os.path import expanduser, join, abspath

  from pyspark.sql import SparkSession
  from pyspark.sql import Row

  # warehouse_location points to the default location for managed databases and tables
  warehouse_location = abspath('spark-warehouse')

  spark = SparkSession \
      .builder \
      .appName("Python Spark SQL Hive integration example") \
      .config("spark.sql.warehouse.dir", warehouse_location) \
      .enableHiveSupport() \
      .getOrCreate()

  # spark is an existing SparkSession
  spark.sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING) USING hive")
  spark.sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src")

  # Queries are expressed in HiveQL
  spark.sql("SELECT * FROM src").show()
  # +---+-------+
  # |key|  value|
  # +---+-------+
  # |238|val_238|
  # | 86| val_86|
  # |311|val_311|
  # ...

  # Aggregation queries are also supported.
  spark.sql("SELECT COUNT(*) FROM src").show()
  # +--------+
  # |count(1)|
  # +--------+
  # |    500 |
  # +--------+

  # The results of SQL queries are themselves DataFrames and support all normal functions.
  sqlDF = spark.sql("SELECT key, value FROM src WHERE key < 10 ORDER BY key")

  # The items in DataFrames are of type Row, which allows you to access each column by ordinal.
  stringsDS = sqlDF.rdd.map(lambda row: "Key: %d, Value: %s" % (row.key, row.value))
  for record in stringsDS.collect():
      print(record)
  # Key: 0, Value: val_0
  # Key: 0, Value: val_0
  # Key: 0, Value: val_0
  # ...

  # You can also use DataFrames to create temporary views within a SparkSession.
  Record = Row("key", "value")
  recordsDF = spark.createDataFrame([Record(i, "val_" + str(i)) for i in range(1, 101)])
  recordsDF.createOrReplaceTempView("records")

  # Queries can then join DataFrame data with data stored in Hive.
  spark.sql("SELECT * FROM records r JOIN src s ON r.key = s.key").show()
  # +---+------+---+------+
  # |key| value|key| value|
  # +---+------+---+------+
  # |  2| val_2|  2| val_2|
  # |  4| val_4|  4| val_4|
  # |  5| val_5|  5| val_5|
  # ...

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/hive.py" 文件。

**R**

When working with Hive one must instantiate SparkSession with Hive support. This adds support for finding tables in the MetaStore and writing queries using HiveQL.

.. code-block:: R

  # enableHiveSupport defaults to TRUE
  sparkR.session(enableHiveSupport = TRUE)
  sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING) USING hive")
  sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src")

  # Queries can be expressed in HiveQL.
  results <- collect(sql("FROM src SELECT key, value"))

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件


Specifying storage format for Hive tables
=============================================

When you create a Hive table, you need to define how this table should read/write data from/to file system, i.e. the “input format” and “output format”. You also need to define how this table should deserialize the data to rows, or serialize rows to data, i.e. the “serde”. The following options can be used to specify the storage format(“serde”, “input format”, “output format”), e.g. CREATE TABLE src(id int) USING hive OPTIONS(fileFormat 'parquet'). By default, we will read the table files as plain text. Note that, Hive storage handler is not supported yet when creating table, you can create a table using storage handler at Hive side, and use Spark SQL to read it.

==========================================          ===============
属性名         	                                      含义
==========================================          ===============
fileFormat	                                        A fileFormat is kind of a package of storage format specifications, including "serde", "input format" and "output format". Currently we support 6 fileFormats: 'sequencefile', 'rcfile', 'orc', 'parquet', 'textfile' and 'avro'.
inputFormat, outputFormat	                          These 2 options specify the name of a corresponding `InputFormat` and `OutputFormat` class as a string literal, e.g. `org.apache.hadoop.hive.ql.io.orc.OrcInputFormat`. These 2 options must be appeared in pair, and you can not specify them if you already specified the `fileFormat` option.
serde	                                              This option specifies the name of a serde class. When the `fileFormat` option is specified, do not specify this option if the given `fileFormat` already include the information of serde. Currently "sequencefile", "textfile" and "rcfile" don't include the serde information and you can use this option with these 3 fileFormats.
fieldDelim, escapeDelim,                            These options can only be used with "textfile" fileFormat. They define how to read delimited files into rows.
collectionDelim, mapkeyDelim, lineDelim
==========================================          ===============

All other properties defined with OPTIONS will be regarded as Hive serde properties.


与不同版本的Hive Metastore交互
=================================

Spark SQL对Hive最重要的一个支持就是可以和Hive metastore进行交互，这使得Spark SQL可以访问Hive表的元数据。从Spark 1.4.0版本开始，通过使用下面描述的配置, Spark SQL一个简单的二进制编译版本可以用来查询不同版本的Hive metastore。注意，不管用于访问 metastore的Hive是什么版本，Spark SQL内部都使用 Hive 1.2.1 版本进行编译, 并且使用这个版本的一些类用于内部执行（serdes，UDFs，UDAFs等）。

下面的选项可用来配置用于检索元数据的Hive版本：

==========================================      ==========================      ==========
属性名         	                                  默认值	                          含义
==========================================      ==========================      ==========
spark.sql.hive.metastore.version	              1.2.1	                          Version of the Hive metastore. Available options are 0.12.0 through 1.2.1.
spark.sql.hive.metastore.jars	                  builtin	                        Location of the jars that should be used to instantiate the HiveMetastoreClient. This property can be one of three options:
                                                                                builtin  Use Hive 1.2.1, which is bundled with the Spark assembly when -Phive is enabled. When this option is chosen, spark.sql.hive.metastore.version must be either 1.2.1 or not defined.
                                                                                maven Use Hive jars of specified version downloaded from Maven repositories. This configuration is not generally recommended for production deployments.
                                                                                A classpath in the standard format for the JVM. This classpath must include all of Hive and its dependencies, including the correct version of Hadoop. These jars only need to be present on the driver, but if you are running in yarn cluster mode then you must ensure they are packaged with your application.
spark.sql.hive.metastore.sharedPrefixes	        com.mysql.jdbc,                 A comma separated list of class prefixes that should be loaded using the classloader that is shared between Spark SQL and a specific version of Hive. An example of classes that should be shared is JDBC drivers that are needed to talk to the metastore. Other classes that need to be shared are those that interact with classes that are already shared. For example, custom appenders that are used by log4j.
                                                org.postgresql,
                                                com.microsoft.sqlserver,
                                                oracle.jdbc
spark.sql.hive.metastore.barrierPrefixes	      (empty)                         A comma separated list of class prefixes that should explicitly be reloaded for each version of Hive that Spark SQL is communicating with. For example, Hive UDFs that are declared in a prefix that typically would be shared (i.e. org.apache.spark.*).
==========================================      ==========================      ==========


JDBC To Other Databases
==============================

Spark SQL也包括一个可以使用JDBC从其它数据库读取数据的数据源。该功能应该优于使用JdbcRDD，因为它的返回结果是一个DataFrame，而在Spark SQL中DataFrame处理简单，且和其它数据源进行关联操作。JDBC数据源在Java和Python中用起来很简单，因为不需要用户提供一个ClassTag。（注意，这和 Spark SQL JDBC server不同，Spark SQL JDBC server 允许其它应用程序使用Spark SQL执行查询）

首先，你需要在 Spark classpath 中包含对应数据库的 JDBC driver。例如，为了从 Spark Shell 连接到 postgres 数据库，你需要运行下面的命令：

.. code-block:: Shell

  bin/spark-shell --driver-class-path postgresql-9.4.1207.jar --jars postgresql-9.4.1207.jar

通过使用 Data Sources API, 远程数据库的表可以加载为一个 DataFrame 或 Spark SQL 临时表。支持的选项如下：

==========================================        ====================
属性名	                                            含义
==========================================        ====================
url	                                              The JDBC URL to connect to. The source-specific connection properties may be specified in the URL. e.g., jdbc:postgresql://localhost/test?user=fred&password=secret
dbtable	                                          The JDBC table that should be read. Note that anything that is valid in a FROM clause of a SQL query can be used. For example, instead of a full table you could also use a subquery in parentheses.
driver	                                          The class name of the JDBC driver to use to connect to this URL.
partitionColumn, lowerBound, upperBound	          These options must all be specified if any of them is specified. In addition, numPartitions must be specified. They describe how to partition the table when reading in parallel from multiple workers. partitionColumn must be a numeric column from the table in question. Notice that lowerBound and upperBound are just used to decide the partition stride, not for filtering the rows in table. So all rows in the table will be partitioned and returned. This option applies only to reading.
numPartitions	                                    The maximum number of partitions that can be used for parallelism in table reading and writing. This also determines the maximum number of concurrent JDBC connections. If the number of partitions to write exceeds this limit, we decrease it to this limit by calling coalesce(numPartitions) before writing.
fetchsize	                                        The JDBC fetch size, which determines how many rows to fetch per round trip. This can help performance on JDBC drivers which default to low fetch size (eg. Oracle with 10 rows). This option applies only to reading.
batchsize	                                        The JDBC batch size, which determines how many rows to insert per round trip. This can help performance on JDBC drivers. This option applies only to writing. It defaults to 1000.
isolationLevel	                                  The transaction isolation level, which applies to current connection. It can be one of NONE, READ_COMMITTED, READ_UNCOMMITTED, REPEATABLE_READ, or SERIALIZABLE, corresponding to standard transaction isolation levels defined by JDBC's Connection object, with default of READ_UNCOMMITTED. This option applies only to writing. Please refer the documentation in java.sql.Connection.
truncate	                                        This is a JDBC writer related option. When SaveMode.Overwrite is enabled, this option causes Spark to truncate an existing table instead of dropping and recreating it. This can be more efficient, and prevents the table metadata (e.g., indices) from being removed. However, it will not work in some cases, such as when the new data has a different schema. It defaults to false. This option applies only to writing.
createTableOptions	                              This is a JDBC writer related option. If specified, this option allows setting of database-specific table and partition options when creating a table (e.g., CREATE TABLE t (name string) ENGINE=InnoDB.). This option applies only to writing.
createTableColumnTypes	                          The database column data types to use instead of the defaults, when creating the table. Data type information should be specified in the same format as CREATE TABLE columns syntax (e.g: "name CHAR(64), comments VARCHAR(1024)"). The specified types should be valid spark sql data types. This option applies only to writing.
==========================================        ====================

**Scala**

.. code-block:: Scala

  // Note: JDBC loading and saving can be achieved via either the load/save or jdbc methods
  // Loading data from a JDBC source
  val jdbcDF = spark.read
    .format("jdbc")
    .option("url", "jdbc:postgresql:dbserver")
    .option("dbtable", "schema.tablename")
    .option("user", "username")
    .option("password", "password")
    .load()

  val connectionProperties = new Properties()
  connectionProperties.put("user", "username")
  connectionProperties.put("password", "password")
  val jdbcDF2 = spark.read
    .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)

  // Saving data to a JDBC source
  jdbcDF.write
    .format("jdbc")
    .option("url", "jdbc:postgresql:dbserver")
    .option("dbtable", "schema.tablename")
    .option("user", "username")
    .option("password", "password")
    .save()

  jdbcDF2.write
    .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)

  // Specifying create table column data types on write
  jdbcDF.write
    .option("createTableColumnTypes", "name CHAR(64), comments VARCHAR(1024)")
    .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" 文件。

**Java**

.. code-block:: Java

  // Note: JDBC loading and saving can be achieved via either the load/save or jdbc methods
  // Loading data from a JDBC source
  Dataset<Row> jdbcDF = spark.read()
    .format("jdbc")
    .option("url", "jdbc:postgresql:dbserver")
    .option("dbtable", "schema.tablename")
    .option("user", "username")
    .option("password", "password")
    .load();

  Properties connectionProperties = new Properties();
  connectionProperties.put("user", "username");
  connectionProperties.put("password", "password");
  Dataset<Row> jdbcDF2 = spark.read()
    .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties);

  // Saving data to a JDBC source
  jdbcDF.write()
    .format("jdbc")
    .option("url", "jdbc:postgresql:dbserver")
    .option("dbtable", "schema.tablename")
    .option("user", "username")
    .option("password", "password")
    .save();

  jdbcDF2.write()
    .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties);

  // Specifying create table column data types on write
  jdbcDF.write()
    .option("createTableColumnTypes", "name CHAR(64), comments VARCHAR(1024)")
    .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties);

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" 文件。

**Python**

.. code-block:: Python

  # Note: JDBC loading and saving can be achieved via either the load/save or jdbc methods
  # Loading data from a JDBC source
  jdbcDF = spark.read \
      .format("jdbc") \
      .option("url", "jdbc:postgresql:dbserver") \
      .option("dbtable", "schema.tablename") \
      .option("user", "username") \
      .option("password", "password") \
      .load()

  jdbcDF2 = spark.read \
      .jdbc("jdbc:postgresql:dbserver", "schema.tablename",
            properties={"user": "username", "password": "password"})

  # Saving data to a JDBC source
  jdbcDF.write \
      .format("jdbc") \
      .option("url", "jdbc:postgresql:dbserver") \
      .option("dbtable", "schema.tablename") \
      .option("user", "username") \
      .option("password", "password") \
      .save()

  jdbcDF2.write \
      .jdbc("jdbc:postgresql:dbserver", "schema.tablename",
            properties={"user": "username", "password": "password"})

  # Specifying create table column data types on write
  jdbcDF.write \
      .option("createTableColumnTypes", "name CHAR(64), comments VARCHAR(1024)") \
      .jdbc("jdbc:postgresql:dbserver", "schema.tablename",
            properties={"user": "username", "password": "password"})

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/python/sql/datasource.py" 文件。

**R**

.. code-block:: R

  # Loading data from a JDBC source
  df <- read.jdbc("jdbc:postgresql:dbserver", "schema.tablename", user = "username", password = "password")

  # Saving data to a JDBC source
  write.jdbc(df, "jdbc:postgresql:dbserver", "schema.tablename", user = "username", password = "password")

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/r/RSparkSQLExample.R" 文件。

**Sql**

.. code-block:: SQL

  CREATE TEMPORARY VIEW jdbcTable
  USING org.apache.spark.sql.jdbc
  OPTIONS (
    url "jdbc:postgresql:dbserver",
    dbtable "schema.tablename",
    user 'username',
    password 'password'
  )

  INSERT INTO TABLE jdbcTable
  SELECT * FROM resultTable


Troubleshooting
==============================

* 在client session以及所有的executor上，JDBC驱动器类必须对启动类加载器可见。这是因为Java的DriverManager类在打开一个连接之前会做一个安全检查，这样就导致它忽略了对于启动类加载器不可见的所有驱动器。一种简单的方法就是修改所有worker节点上的compute_classpath.sh以包含你驱动器的jar包。
* 有些数据库，比如H2，会把所有的名称转换成大写。在Spark SQL中你也需要使用大写来引用这些名称。


*****************
性能调优
*****************

对于有一定计算量的Spark任务，可以将数据放入内存缓存或开启一些试验选项来提升性能。

缓存数据到内存中
==============================

通过调用 spark.cacheTable(“tableName”) 或者 dataFrame.cache() 方法, Spark SQL可以使用一种内存列存储格式缓存表。接着Spark SQL只扫描必要的列，并且自动调整压缩比例，以最小化内存占用和GC压力。你可以调用 spark.uncacheTable(“tableName”) 方法删除内存中的表。

内存缓存配置可以使用 SparkSession 类中的 setConf 方法或在SQL语句中运行 SET key=value命令来完成。

==============================================         ==========          ===============
属性名                                                   默认值               含义
==============================================         ==========          ===============
spark.sql.inMemoryColumnarStorage.compressed            true                如果设置为true，Spark SQL将会基于统计数据自动地为每一列选择一种压缩编码方式。
spark.sql.inMemoryColumnarStorage.batchSize             10000               控制列式缓存批处理大小。缓存数据时, 较大的批处理大小可以提高内存利用率和压缩率，但同时也会带来OOM（Out Of Memory）的风险。
==============================================         ==========          ===============


其它配置选项
==============================

下面的选项也可以用来提升执行的查询性能。随着Spark自动地执行越来越多的优化操作, 这些选项在未来的发布版本中可能会过时。

========================================        ======================          ======================
属性名                                            默认值                           含义
========================================        ======================          ======================
spark.sql.files.maxPartitionBytes                134217728 (128 MB)              读取文件时单个分区可容纳的最大字节数
spark.sql.files.openCostInBytes                  4194304 (4 MB)                  打开文件的估算成本, 按照同一时间能够扫描的字节数来测量。当往一个分区写入多个文件的时候会使用。高估更好, 这样的话小文件分区将比大文件分区更快 (先被调度).
spark.sql.autoBroadcastJoinThreshold             10485760 (10 MB)                用于配置一个表在执行 join 操作时能够广播给所有worker节点的最大字节大小。通过将这个值设置为 -1 可以禁用广播。注意，当前数据统计仅支持已经运行了ANALYZE TABLE <tableName> COMPUTE STATISTICS noscan 命令的Hive Metastore表。
spark.sql.shuffle.partitions                     200                             用于配置join或聚合操作混洗（shuffle）数据时使用的分区数。
========================================        ======================          ======================


*****************
分布式 SQL 引擎
*****************

通过使用 JDBC/ODBC或者命令行接口，Spark SQL还可以作为一个分布式查询引擎。在这种模式下，终端用户或应用程序可以运行SQL查询来直接与Spark SQL交互，而不需要编写任何代码。

运行 Thrift JDBC/ODBC 服务器
===================================

这里实现的Thrift JDBC/ODBC server对应于Hive 1.2.1 版本中的HiveServer2。你可以使用Spark或者Hive 1.2.1自带的beeline脚本来测试这个JDBC server。

要启动JDBC/ODBC server， 需要在Spark安装目录下运行下面这个命令：

.. code-block:: Shell

  ./sbin/start-thriftserver.sh

这个脚本能接受所有 bin/spark-submit 命令行选项，外加一个用于指定Hive属性的 --hiveconf 选项。你可以运行 ./sbin/start-thriftserver.sh —help 来查看所有可用选项的完整列表。默认情况下，启动的 server 将会在 localhost:10000 上进行监听。你可以覆盖该行为, 比如使用以下环境变量：

.. code-block:: Shell

  export HIVE_SERVER2_THRIFT_PORT=<listening-port>
  export HIVE_SERVER2_THRIFT_BIND_HOST=<listening-host>
  ./sbin/start-thriftserver.sh \
    --master <master-uri> \
    ...

或者系统属性：

.. code-block:: Shell

  ./sbin/start-thriftserver.sh \
    --hiveconf hive.server2.thrift.port=<listening-port> \
    --hiveconf hive.server2.thrift.bind.host=<listening-host> \
    --master <master-uri>
    ...

现在你可以使用 beeline来测试这个Thrift JDBC/ODBC server:

.. code-block:: Shell

  ./bin/beeline

在beeline中使用以下命令连接到JDBC/ODBC server：

.. code-block:: Shell

  beeline> !connect jdbc:hive2://localhost:10000

Beeline会要求你输入用户名和密码。在非安全模式下，只需要输入你本机的用户名和一个空密码即可。对于安全模式，请参考beeline文档中的指示.

将 hive-site.xml，core-site.xml以及hdfs-site.xml文件放置在conf目录下可以完成Hive配置。

你也可以使用Hive 自带的 beeline 的脚本。

Thrift JDBC server还支持通过HTTP传输来发送Thrift RPC消息。使用下面的设置来启用HTTP模式：

hive.server2.transport.mode - Set this to value: http
hive.server2.thrift.http.port - HTTP port number fo listen on; default is 10001
hive.server2.http.endpoint - HTTP endpoint; default is cliservice

为了测试，下面在HTTP模式中使用beeline连接到JDBC/ODBC server:

.. code-block:: Shell

  beeline> !connect jdbc:hive2://<host>:<port>/<database>?hive.server2.transport.mode=http;hive.server2.thrift.http.path=<http_endpoint>


运行 Spark SQL CLI
===================================

Spark SQL CLI是一个很方便的工具，它可以在本地模式下运行Hive metastore服务，并且执行从命令行中输入的查询语句。注意：Spark SQL CLI无法与Thrift JDBC server通信。

要启动用Spark SQL CLI, 可以在Spark安装目录运行下面的命令:

.. code-block:: Shell

  ./bin/spark-sql

将 hive-site.xml，core-site.xml以及hdfs-site.xml文件放置在conf目录下可以完成Hive配置。你可以运行 ./bin/spark-sql –help 来获取所有可用选项的完整列表。


*****************
迁移指南
*****************

Spark SQL 从 2.1 版本升级到2.2 版本
=========================================

* Spark 2.1.1 introduced a new configuration key: spark.sql.hive.caseSensitiveInferenceMode. It had a default setting of NEVER_INFER, which kept behavior identical to 2.1.0. However, Spark 2.2.0 changes this setting’s default value to INFER_AND_SAVE to restore compatibility with reading Hive metastore tables whose underlying file schema have mixed-case column names. With the INFER_AND_SAVE configuration value, on first access Spark will perform schema inference on any Hive metastore table for which it has not already saved an inferred schema. Note that schema inference can be a very time consuming operation for tables with thousands of partitions. If compatibility with mixed-case column names is not a concern, you can safely set spark.sql.hive.caseSensitiveInferenceMode to NEVER_INFER to avoid the initial overhead of schema inference. Note that with the new default INFER_AND_SAVE setting, the results of the schema inference are saved as a metastore key for future use. Therefore, the initial schema inference occurs only at a table’s first access.

Spark SQL 从 2.0 版本升级到 2.1 版本
=========================================

* Datasource tables now store partition metadata in the Hive metastore. This means that Hive DDLs such as ALTER TABLE PARTITION ... SET LOCATION are now available for tables created with the Datasource API.
Legacy datasource tables can be migrated to this format via the MSCK REPAIR TABLE command. Migrating legacy tables is recommended to take advantage of Hive DDL support and improved planning performance.
To determine if a table has been migrated, look for the PartitionProvider: Catalog attribute when issuing DESCRIBE FORMATTED on the table.

* Changes to INSERT OVERWRITE TABLE ... PARTITION ... behavior for Datasource tables.
In prior Spark versions INSERT OVERWRITE overwrote the entire Datasource table, even when given a partition specification. Now only partitions matching the specification are overwritten.
Note that this still differs from the behavior of Hive tables, which is to overwrite only partitions overlapping with newly inserted data.


Spark SQL 从 1.6 版本升级到 2.0 版本
=========================================

* SparkSession 现在是 Spark 新的切入点, 它替代了老的 SQLContext 和 HiveContext。注意：为了向下兼容, 老的 SQLContext 和 HiveContext 仍然保留。可以从 SparkSession 获取一个新的 catalog 接口- 现有的访问数据库和表的API, 如 listTables, createExternalTable, dropTempView, cacheTable 都被移到该接口。
* Dataset API 和 DataFrame API 进行了统一。在 Scala 中, DataFrame 变成了 Dataset[Row]的一个类型别名, 而Java API使用者必须将 DataFrame 替换成 Dataset<Row>。Dataset 类既提供了强类型转换操作 (如 map, filter 以及 groupByKey) 也提供了非强类型转换操作 (如 select 和 groupBy) 。由于编译期的类型安全不是 Python 和 R 语言的一个特性,  Dataset 的概念并不适用于这些语言的 API。相反, DataFrame 仍然是最基本的编程抽象, 就类似于这些语言中单节点数据帧的概念。
* Dataset 和 DataFrame API 中 unionAll 已经过时并且由 union 替代。
* Dataset 和 DataFrame API 中 explode 已经过时。或者 functions.explode() 可以结合 select 或 flatMap 一起使用。
* Dataset 和 DataFrame API 中 registerTempTable 已经过时并且由 createOrReplaceTempView 替代。

Spark SQL 从 1.5 版本升级到 1.6 版本
=========================================

* 从 Spark 1.6 版本开始，Thrift server 默认运行于多会话模式下, 这意味着每个JDBC/ODBC连接都有独有一份SQL配置和临时函数注册表的拷贝。尽管如此, 缓存的表仍然可以共享。如果你更喜欢在老的单会话模式中运行Thrift server，只需要将spark.sql.hive.thriftServer.singleSession选项设置为true即可。当然，你也可在spark-defaults.conf文件中添加这个选项，或者通过--conf将其传递给start-thriftserver.sh：

.. code-block:: Shell

  ./sbin/start-thriftserver.sh \
       --conf spark.sql.hive.thriftServer.singleSession=true \
       ...

* 从 Spark 1.6.1 版本开始, sparkR 中的 withColumn 方法支持向 DataFrame 新增一列 或 替换已有的名称相同的列。

* 从 Spark 1.6 版本开始, LongType 转换成 TimestampType 将源值以秒而不是毫秒作为单位处理。做出这个变更是为了的匹配Hive 1.2 版本中从数值类型转换成TimestampType的这个行为以获得更一致的类型。更多细节请参见 SPARK-11724 。

Spark SQL 从 1.4 版本升级到 1.5 版本
=========================================

* 使用手动管理内存(Tungsten引擎)的执行优化以及用于表达式求值的代码自动生成现在默认是启用的。这些特性可以通过将spark.sql.tungsten.enabled的值设置为false来同时禁用。
* 默认不启用 Parquet schema 合并。可以将 spark.sql.parquet.mergeSchema 的值设置为true来重新启用。
* Python 中对于列的字符串分解现在支持使用点号(.)来限定列或访问内嵌值，例如 df[‘table.column.nestedField’]。然而这也意味着如果你的列名包含任何点号(.)的话，你就必须要使用反引号来转义它们(例如：table.`column.with.dots`.nested)。
* 默认启用内存中列式存储分区修剪。可以通过设置 spark.sql.inMemoryColumarStorage.partitionPruning 值为false来禁用它。
* 不再支持无精度限制的decimal，相反, Spark SQL现在强制限制最大精度为38位。从BigDecimal对象推导schema时会使用（38，18）这个精度。如果在DDL中没有指定精度，则默认使用精度Decimal(10，0)。
* 存储的时间戳(Timestamp)现在精确到1us（微秒），而不是1ns（纳秒）
* 在 sql 方言中，浮点数现在被解析成decimal。HiveQL 的解析保持不变。
* SQL/DataFrame函数的规范名称均为小写(例如：sum vs SUM)。
* JSON数据源不会再自动地加载其他应用程序创建的新文件（例如，不是由Spark SQL插入到dataset中的文件）。对于一个JSON持久化表（例如：存储在Hive metastore中的表的元数据），用户可以使用 REFRESH TABLE 这个SQL命令或者 HiveContext 的 refreshTable 方法来把新文件添加进表。对于一个表示JSON数据集的DataFrame, 用户需要重建这个 DataFrame, 这样新的 DataFrame 就会包含新的文件。
* pySpark 中的 DataFrame.withColumn 方法支持新增一列或是替换名称相同列。

Spark SQL 从 1.3 版本升级到 1.4 版本
=========================================

DataFrame数据读写接口
---------------------------

根据用户的反馈，我们提供了一个用于数据读入（SQLContext.read）和数据写出（DataFrame.write）的新的、更加流畅的API，同时老的API（如：SQLCOntext.parquetFile, SQLContext.jsonFile）将被废弃。

有关 SQLContext.read ( Scala, Java, Python ) 和 DataFrame.write ( Scala, Java, Python ) 的更多信息，请参考API文档。

DataFrame.groupBy保留分组的列
------------------------------

根据用户的反馈，我们改变了DataFrame.groupBy().agg()的默认行为，就是在返回的DataFrame结果中保留分组的列。如果你想保持1.3版本中的行为，可以将spark.sql.retainGroupColumns设置为false。

**Scala**

.. code-block:: Scala

  // In 1.3.x, in order for the grouping column "department" to show up,
  // it must be included explicitly as part of the agg function call.
  df.groupBy("department").agg($"department", max("age"), sum("expense"))

  // In 1.4+, grouping column "department" is included automatically.
  df.groupBy("department").agg(max("age"), sum("expense"))

  // Revert to 1.3 behavior (not retaining grouping column) by:
  sqlContext.setConf("spark.sql.retainGroupColumns", "false")

**Java**

.. code-block:: Java

  // In 1.3.x, in order for the grouping column "department" to show up,
  // it must be included explicitly as part of the agg function call.
  df.groupBy("department").agg(col("department"), max("age"), sum("expense"));

  // In 1.4+, grouping column "department" is included automatically.
  df.groupBy("department").agg(max("age"), sum("expense"));

  // Revert to 1.3 behavior (not retaining grouping column) by:
  sqlContext.setConf("spark.sql.retainGroupColumns", "false");

**Python**

.. code-block:: Python

  import pyspark.sql.functions as func

  # In 1.3.x, in order for the grouping column "department" to show up,
  # it must be included explicitly as part of the agg function call.
  df.groupBy("department").agg(df["department"], func.max("age"), func.sum("expense"))

  # In 1.4+, grouping column "department" is included automatically.
  df.groupBy("department").agg(func.max("age"), func.sum("expense"))

  # Revert to 1.3.x behavior (not retaining grouping column) by:
  sqlContext.setConf("spark.sql.retainGroupColumns", "false")

Behavior change on DataFrame.withColumn
-------------------------------------------

1.4版本之前, DataFrame.withColumn() 只支持新增一列。在DataFrame结果中指定名称的列总是作为一个新列添加进来，即使已经存在了相同名称的列。从1.4版本开始, DataFrame.withColumn() 支持新增一个和现有列名不重复的新列和替换有相同名称的列。

注意：这个变更只针对 Scala API, 不针对 PySpark 和 SparkR。

Spark SQL 从 1.0-1.2 版本升级到 1.3 版本
=========================================

Spark 1.3版本我们去掉了Spark SQL的 ”Alpha“ 标签并且作为其中的一部分我们在现有的API上做了清理。从Spark 1.3版本开始，Spark SQL将提供1.x系列中其它发行版本的二进制兼容。这个兼容性保证不包括显式地标注为不稳定（例如：DeveloperAPI 或 Experimental）的API。

SchemaRDD重命名为DataFrame
--------------------------

升级到Spark SQL 1.3后，用户将会注意到最大的改动就是 SchemaRDD 改名为 DataFrame。主要原因是DataFrame不再直接继承于RDD，而是通过自己的实现来提供RDD中提供的绝大多数功能。通过调用 .rdd 方法 DataFrame 仍然可以转换成RDD。

在Scala中有一个从SchemaRDD到DataFrame的类型别名来提供某些使用场景下的代码兼容性。但仍然建议用户在代码中改用DataFrame。Java和Python用户必须要修改代码。

统一Java和Scala API
--------------------------

Spark 1.3 之前的版本中有两个单独的Java兼容类（JavaSQLContext 和 JavaSchemaRDD）可以映射到 Scala API。Spark 1.3版本将Java API和Scala API进行了统一。两种语言的用户都应该使用SQLContext和DataFrame。通常情况下这些类都会使用两种语言中都支持的类型（例如：使用Array来取代语言特有的集合）。有些情况下没有通用的类型（例如：闭包或maps中用于传值），则会使用函数重载。

另外，移除了Java特有的类型API。Scala 和 Java 用户都应该使用 org.apache.spark.sql.types 包中的类来编程式地描述 schema。

隔离隐式转换并删除dsl包(仅针对Scala)
--------------------------------------------------

Spark 1.3版本之前的很多示例代码都以 import sqlContext._ 语句作为开头，这样会引入sqlContext的所有函数。在Spark 1.3版本中我们隔离了RDD到DataFrame的隐式转换，将其单独放到SQLContext内部的一个对象中。用户现在应该这样写：import sqlContext.implicits._。

另外，隐式转换现在也只能使用toDF方法来增加由Product（例如：case classes 或 元祖）组成的RDD，而不是自动转换。

使用 DSL（现在被DataFrame API取代）的内部方法时，用户需要引入 import org.apache.spark.sql.catalyst.dsl。而现在应该要使用公用的  DataFrame函数API：import org.apache.spark.sql.functions._

移除org.apache.spark.sql中DataType的类型别名(仅针对Scala)
-------------------------------------------------------------

Spark 1.3版本删除了基础sql包中DataType的类型别名。开发人员应该引入 org.apache.spark.sql.types 中的类。

UDF注册迁移到sqlContext.udf中(Java&Scala)
--------------------------------------------

用于注册UDF的函数，不管是DataFrame DSL还是SQL中用到的，都被迁移到SQLContext中的udf对象中。

**Scala**

.. code-block:: Scala

  sqlContext.udf.register("strLen", (s: String) => s.length())

**Java**

.. code-block:: Java

  sqlContext.udf().register("strLen", (String s) -> s.length(), DataTypes.IntegerType);

Python UDF注册保持不变。

Python的DataType不再是单例的
-----------------------------

在 Python 中使用DataTypes时，你需要先构造它们（如：StringType()），而不是引用一个单例对象。


兼容Apache Hive
=========================================

Spark SQL 在设计时就考虑到了和 Hive metastore，SerDes 以及 UDF 之间的兼容性。目前 Hive SerDes 和 UDF 都是基于Hive 1.2.1版本，并且Spark SQL可以连接到不同版本的Hive metastore（从0.12.0到1.2.1，可以参考[与不同版本的Hive Metastore交互]）

在已有的Hive仓库中部署
-----------------------

Spark SQL Thrift JDBC server采用了开箱即用的设计以兼容已有的 Hive 安装版本。你不需要修改现有的Hive Metastore ,  或者改变数据的位置和表的分区。

支持的 Hive 功能
-----------------------

Spark SQL 支持绝大部分的Hive功能，如：

* Hive查询语句, 包括：
    * SELECT
    * GROUP BY
    * ORDER BY
    * CLUSTER BY
    * SORT BY
* 所有的Hive运算符， 包括：
    * 关系运算符 (=, ⇔, ==, <>, <, >, >=, <=, etc)
    * 算术运算符 (+, -, *, /, %, etc)
    * 逻辑运算符 (AND, &&, OR, ||, etc)
    * 复杂类型构造器
    * 数学函数 (sign, ln, cos等)
    * String 函数 (instr, length, printf等)
* 用户自定义函数（UDF）
* 用户自定义聚合函数（UDAF）
* 用户自定义序列化格式（SerDes）
* 窗口函数
* Joins
    * JOIN
    * {LEFT|RIGHT|FULL} OUTER JOIN
    * LEFT SEMI JOIN
    * CROSS JOIN
* Unions
* 子查询
    * SELECT col FROM ( SELECT a + b AS col from t1) t2
* 采样
* Explain
* 分区表，包括动态分区插入
* 视图
* 所有Hive DDL功能, 包括：
    * CREATE TABLE
    * CREATE TABLE AS SELECT
    * ALTER TABLE
* 绝大多数 Hive 数据类型，包括：
    * TINYINT
    * SMALLINT
    * INT
    * BIGINT
    * BOOLEAN
    * FLOAT
    * DOUBLE
    * STRING
    * BINARY
    * TIMESTAMP
    * DATE
    * ARRAY<>
    * MAP<>
    * STRUCT<>

不支持的 Hive 功能
-----------------------

以下是目前还不支持的 Hive 功能列表。在 Hive 部署中这些功能大部分都用不到。

Hive 核心功能
^^^^^^^^^^^^^^^^

* bucket：bucket是 Hive 表分区内的一个哈希分区，Spark SQL 目前还不支持 bucket。

Hive 高级功能
^^^^^^^^^^^^^^^^

* UNION 类型
* Unique join
* 列统计数据收集：Spark SQL 目前不依赖扫描来收集列统计数据并且仅支持填充Hive metastore 的 sizeInBytes 字段。

Hive输入输出格式
^^^^^^^^^^^^^^^^

* CLI文件格式：对于回显到CLI中的结果，Spark SQL 仅支持 TextOutputFormat。
* Hadoop archive

Hive优化
^^^^^^^^^^^^^^^^

有少数Hive优化还没有包含在Spark中。其中一些（比如索引）由于Spark SQL的这种内存计算模型而显得不那么重要。另外一些在Spark SQL未来的版本中会持续跟踪。

* 块级别位图索引和虚拟列（用来建索引）
* 自动为 join 和 groupBy 计算 reducer 个数：目前在 Spark SQL 中，你需要使用 ”SET spark.sql.shuffle.partitions=[num_tasks];” 来控制后置混洗的并行程度。
* 仅查询元数据：对于只需要使用元数据的查询请求，Spark SQL 仍需要启动任务来计算结果
* 数据倾斜标志：Spark SQL 不遵循 Hive 中的数据倾斜标志
* STREAMTABLE join操作提示：Spark SQL 不遵循 STREAMTABLE 提示。
* 对于查询结果合并多个小文件：如果返回的结果有很多小文件，Hive有个选项设置，来合并小文件，以避免超过HDFS的文件数额度限制。Spark SQL不支持这个。


*****************
参考
*****************

数据类型
=================

Spark SQL 和 DataFrame 支持以下数据类型：

* 数值类型
    * ByteType: 表示1字节长的有符号整型，数值范围：-128 到 127。
    * ShortType: 表示2字节长的有符号整型，数值范围：-32768 到 32767。
    * IntegerType: 表示4字节长的有符号整型，数值范围：-2147483648 到 2147483647。
    * LongType: 表示8字节长的有符号整型，数值范围： -9223372036854775808 to 9223372036854775807。
    * FloatType: 表示4字节长的单精度浮点数。
    * DoubleType: 表示8字节长的双精度浮点数。
    * DecimalType: 表示任意精度的有符号的十进制数。内部使用 java.math.BigDecimal 实现。一个 BigDecimal 由一个任意精度的整数非标度值和一个32位的整数标度组成。
* 字符串类型
    * StringType: 表示字符串值。
* 二进制类型
    * BinaryType: 表示字节序列值。
* 布尔类型
    * BooleanType: 表示布尔值。
* 时间类型
    * TimestampType: 表示由年、月、日、时、分以及秒等字段值组成的时间值。
    * DateType: 表示由年、月、日字段值组成的日期值。
* 复杂类型
    * ArrayType(elementType, containsNull)：表示由元素类型为 elementType 的序列组成的值，containsNull 用来标识 ArrayType 中的元素值能否为 null。
    * MapType(keyType, valueType, valueContainsNull)：表示由一组键值对组成的值。键的数据类型由 keyType 表示，值的数据类型由 valueType 表示。对于 MapType 值，键值不允许为 null。valueContainsNull 用来表示一个 MapType 的值是否能为 null。
* StructType(fields)：表示由 StructField 序列描述的结构。
        * StructField(name, datatype, nullable): 表示 StructType 中的一个字段，name 表示字段名。dataType 表示字段的数据类型，nullable 用来表示该字段的值是否可以为 null。


**Scala**

Spark SQL 所有的数据类型都位于 org.apache.spark.sql.types 包中。你可以使用下面的语句访问他们:

.. code-block:: Scala

  import org.apache.spark.sql.types._

完整示例代码参见 Spark 源码仓库中的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 文件。

=============================      ============================     ============================
数据类型	                           Scala 中的值类型	                  用于获取或创建一个数据类型的 API
=============================      ============================     ============================
ByteType	                         Byte	                            ByteType
ShortType	                         Short	                          ShortType
IntegerType	                       Int	                            IntegerType
LongType	                         Long	                            LongType
FloatType	                         Float	                          FloatType
DoubleType	                       Double	                          DoubleType
DecimalType	                       java.math.BigDecimal	            DecimalType
StringType	                       String	                          StringType
BinaryType	                       Array[Byte]	                    BinaryType
BooleanType	                       Boolean	                        BooleanType
TimestampType	                     java.sql.Timestamp	              TimestampType
DateType	                         java.sql.Date	                  DateType
ArrayType	                         scala.collection.Seq	            ArrayType(elementType, [containsNull]) 注意：containsNull 的默认值是 true。
MapType	                           scala.collection.Map	            MapType(keyType, valueType, [valueContainsNull]) 注意：valueContainsNull 的默认值是 true。
StructType	                       org.apache.spark.sql.Row	        StructType(fields) 注意: fields 表示一个 StructField 序列。另外不允许出现名称重复的字段。
StructField	                       Scala 中该字段的数据类型对应的值      StructField(name, dataType, [nullable]) 注意: nullable 的默认值是 true。
                                   类型(例如, StructField 的数据类
                                   型是 IntegerType，则对应的值类型
                                   Int)
=============================      ============================     ============================

**Java**

Spark SQL 所有的数据类型都位于 org.apache.spark.sql.types 包中。如果想要访问或创建一个数据类型, 请使用 org.apache.spark.sql.types.DataTypes 中提供的工厂方法。

=================================       =============================         ======================
数据类型   	                              Java 中的值类型	                        用于获取或创建一个数据类型的 API
=================================       =============================         ======================
ByteType	                              byte or Byte	                        DataTypes.ByteType
ShortType	                              short or Short	                      DataTypes.ShortType
IntegerType	                            int or Integer	                      DataTypes.IntegerType
LongType	                              long or Long	                        DataTypes.LongType
FloatType	                              float or Float	                      DataTypes.FloatType
DoubleType	                            double or Double	                    DataTypes.DoubleType
DecimalType	                            java.math.BigDecimal	                DataTypes.createDecimalType()。DataTypes.createDecimalType(precision, scale).
StringType	                            String	                              DataTypes.StringType
BinaryType	                            byte[]	                              DataTypes.BinaryType
BooleanType	                            boolean or Boolean	                  DataTypes.BooleanType
TimestampType	                          java.sql.Timestamp	                  DataTypes.TimestampType
DateType	                              java.sql.Date	                        DataTypes.DateType
ArrayType	                              java.util.List	                      DataTypes.createArrayType(elementType) 注意:  containsNull 的默认值是 true。DataTypes.createArrayType(elementType, containsNull).
MapType	                                java.util.Map	                        DataTypes.createMapType(keyType, valueType) 注意: valueContainsNull 的默认值是true。DataTypes.createMapType(keyType, valueType, valueContainsNull)
StructType	                            org.apache.spark.sql.Row	            DataTypes.createStructType(fields) fields 表示一个 StructField 列表或数组, 另外不允许出现名称重复的字段。
StructField	                            Java 中该字段的数据类型对应的值            DataTypes.createStructField(name, dataType, nullable)
                                        类型(例如, StructField 的数据
                                        类型是 IntegerType，则对应的值
                                        类型是 int)
=================================       =============================         ======================

**Python**

Spark SQL 所有的数据类型都位于 pyspark.sql.types 包中。你可以使用下面的语句访问他们:

.. code-block:: Python

  from pyspark.sql.types import *

=======================================       ======================================      ===================================
数据类型  	                                    Python 中的值类型 	                          用于获取或创建一个数据类型的 API
=======================================       ======================================      ===================================
ByteType	                                    int 或 long                                 ByteType()
                                              注意：数字在运行时会被转化成1字节的有符号整
                                              数。请确保数字在 -128 到 127 这个范围内。
ShortType	                                    int 或 long                                 ShortType()
                                              注意: 数字在运行时会被转化成2字节的有符号整
                                              数。请确保数字在 -32768 到 32767 这个范
                                              围内。
IntegerType	                                  int 或 long	                                IntegerType()
LongType	                                    long                                        LongType()
                                              注意: 数字在运行时会被转化成8字节的有符号整
                                              数。请确保数字在 -9223372036854775808 到
                                              9223372036854775807 这个范围内。不然的
                                              话，需要将数据转化成 decimal.Decimal 并
                                              使用 DecimalType。
FloatType	                                    float                                       FloatType()
                                              注意: 数字在运行时会被转化成4字节的单精度浮
                                              点数。
DoubleType	                                  float	                                      DoubleType()
DecimalType	                                  decimal.Decimal	                            DecimalType()
StringType	                                  string	                                    StringType()
BinaryType	                                  bytearray	                                  BinaryType()
BooleanType	                                  bool	                                      BooleanType()
TimestampType	                                datetime.datetime	                          TimestampType()
DateType	                                    datetime.date	                              DateType()
ArrayType	                                    list, tuple 或 array	                        ArrayType(elementType, [containsNull]) 注意: containsNull 的默认值是 True。
MapType	                                      dict	                                      MapType(keyType, valueType, [valueContainsNull]) 注意: valueContainsNull 的默认值是True。
StructType	                                  list 或 tuple	                              StructType(fields) Note: fields 表示一个 StructField 序列，另外不允许出现名称重复的字段。
StructField                                   Python 中该字段的数据类型对应的值类型(例如,       StructField(name, dataType, [nullable]) 注意: nullable 的默认值是 True。
                                              StructField 的数据类型是 IntegerType, 则
                                              对应的值类型是 Int)
=======================================       ======================================      ===================================


**R**

===============================================       =============================================       ======================================
数据类型	                                              R	中的值类型                                          用于获取或创建一个数据类型的 API
===============================================       =============================================       ======================================
ByteType	                                            integer                                             "byte"
                                                      注意：数字在运行时会被转化成1字节的有符号整数。请确保
                                                      数字在 -128 到 127 这个范围内。
ShortType	                                            integer                                             "short"
                                                      注意: 数字在运行时会被转化成2字节的有符号整数。请确保
                                                      数字在 -32768 到 32767 这个范围内。
IntegerType	                                          integer	                                            "integer"
LongType	                                            integer                                             "long"
                                                      注意: 数字在运行时会被转化成8字节的有符号整数。请确保
                                                      数字在 -9223372036854775808 到
                                                      9223372036854775807 这个范围内。不然的话，需要将数
                                                      据转化成 decimal.Decimal 并使用 DecimalType。
FloatType	                                            numeric                                             "float"
                                                      注意: 数字在运行时会被转化成4字节的单精度浮点数。
DoubleType	                                          numeric	                                            "double"
DecimalType	                                          不支持	                                              不支持
StringType	                                          character	                                          "string"
BinaryType	                                          raw	                                                "binary"
BooleanType	                                          logical	                                            "bool"
TimestampType	                                        POSIXct	                                            "timestamp"
DateType	                                            Date	                                              "date"
ArrayType	                                            vector 或 list	                                      list(type="array", elementType=elementType, containsNull=[containsNull]) 注意: containsNull 的默认值是 TRUE。
MapType	                                              environment	                                        list(type="map", keyType=keyType, valueType=valueType, valueContainsNull=[valueContainsNull]) 注意: valueContainsNull 的默认值是 TRUE。
StructType	                                          named list	                                        list(type="struct", fields=fields) 注意: fields 表示一个 StructField 序列。另外不允许出现名称重复的字段。
StructField	                                          R 中该字段的数据类型对应的值类型 (例如, StructField       list(name=name, type=dataType, nullable=[nullable]) 注意: nullable 的默认值是 TRUE。
                                                      的数据类型是 IntegerType, 则对应的值类型是 integer)
===============================================       =============================================       ======================================

NaN 语义
=================

当处理一些不符合标准浮点语义的 float 或 double 类型时，会对 Not-a-Number(NaN) 做一些特殊处理。具体如下：

* NaN = NaN 返回true。
* 在聚合操作中，所有 NaN 值都被分到同一组。
* 在连接键中 NaN 被当做普通值。
* NaN 值按升序排序时排最后，比其他任何数值都大。