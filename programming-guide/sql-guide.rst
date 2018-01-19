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

Scala
---------

The entry point into all functionality in Spark is the SparkSession class. To create a basic SparkSession, just use SparkSession.builder():

.. code-block:: Scala

  import org.apache.spark.sql.SparkSession

  val spark = SparkSession
    .builder()
    .appName("Spark SQL basic example")
    .config("spark.some.config.option", "some-value")
    .getOrCreate()

  // For implicit conversions like converting RDDs to DataFrames
  import spark.implicits._

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" in the Spark repo.


Java
---------

The entry point into all functionality in Spark is the SparkSession class. To create a basic SparkSession, just use SparkSession.builder():

.. code-block:: Java

  import org.apache.spark.sql.SparkSession;

  SparkSession spark = SparkSession
    .builder()
    .appName("Java Spark SQL basic example")
    .config("spark.some.config.option", "some-value")
    .getOrCreate();

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" in the Spark repo.

Python
---------

The entry point into all functionality in Spark is the SparkSession class. To create a basic SparkSession, just use SparkSession.builder:

.. code-block:: Python

  from pyspark.sql import SparkSession

  spark = SparkSession \
      .builder \
      .appName("Python Spark SQL basic example") \
      .config("spark.some.config.option", "some-value") \
      .getOrCreate()

Find full example code at "examples/src/main/python/sql/basic.py" in the Spark repo.

R
---------

The entry point into all functionality in Spark is the SparkSession class. To initialize a basic SparkSession, just call sparkR.session():

.. code-block:: R

  sparkR.session(appName = "R Spark SQL basic example", sparkConfig = list(spark.some.config.option = "some-value"))

Find full example code at "examples/src/main/r/RSparkSQLExample.R" in the Spark repo.

Note that when invoked for the first time, sparkR.session() initializes a global SparkSession singleton instance, and always returns a reference to this instance for successive invocations. In this way, users only need to initialize the SparkSession once, then SparkR functions like read.df will be able to access this global instance implicitly, and users don’t need to pass the SparkSession instance around.


SparkSession in Spark 2.0 provides builtin support for Hive features including the ability to write queries using HiveQL, access to Hive UDFs, and the ability to read data from Hive tables. To use these features, you do not need to have an existing Hive setup.



创建 DataFrame
=================

Scala
---------

With a SparkSession, applications can create DataFrames from an existing RDD, from a Hive table, or from Spark data sources.

As an example, the following creates a DataFrame based on the content of a JSON file:

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

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" in the Spark repo.

Java
---------

With a SparkSession, applications can create DataFrames from an existing RDD, from a Hive table, or from Spark data sources.

As an example, the following creates a DataFrame based on the content of a JSON file:

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

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" in the Spark repo.

Python
---------

With a SparkSession, applications can create DataFrames from an existing RDD, from a Hive table, or from Spark data sources.

As an example, the following creates a DataFrame based on the content of a JSON file:

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

Find full example code at "examples/src/main/python/sql/basic.py" in the Spark repo.

R
---------

With a SparkSession, applications can create DataFrames from a local R data.frame, from a Hive table, or from Spark data sources.

As an example, the following creates a DataFrame based on the content of a JSON file:

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

Find full example code at "examples/src/main/r/RSparkSQLExample.R" in the Spark repo.


Untyped Dataset Operations (aka DataFrame Operations)
=======================================================

DataFrames provide a domain-specific language for structured data manipulation in Scala, Java, Python and R.

As mentioned above, in Spark 2.0, DataFrames are just Dataset of Rows in Scala and Java API. These operations are also referred as “untyped transformations” in contrast to “typed transformations” come with strongly typed Scala/Java Datasets.

Here we include some basic examples of structured data processing using Datasets


Scala
---------

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

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" in the Spark repo.


Java
---------

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

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" in the Spark repo.


Python
---------

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

Find full example code at "examples/src/main/python/sql/basic.py" in the Spark repo.


R
---------

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

Find full example code at "examples/src/main/r/RSparkSQLExample.R" in the Spark repo.


Running SQL Queries Programmatically
=========================================

Scala
---------

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

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" in the Spark repo.


Java
---------

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

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" in the Spark repo.

Python
---------

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

Find full example code at "examples/src/main/python/sql/basic.py" in the Spark repo.


R
---------

The sql function enables applications to run SQL queries programmatically and returns the result as a SparkDataFrame.

.. code-block:: R

  df <- sql("SELECT * FROM table")

Find full example code at "examples/src/main/r/RSparkSQLExample.R" in the Spark repo.


Global Temporary View
==============================

Temporary views in Spark SQL are session-scoped and will disappear if the session that creates it terminates. If you want to have a temporary view that is shared among all sessions and keep alive until the Spark application terminates, you can create a global temporary view. Global temporary view is tied to a system preserved database global_temp, and we must use the qualified name to refer it, e.g. SELECT * FROM global_temp.view1.

Scala
---------

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

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" in the Spark repo.

Java
---------

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

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" in the Spark repo.

Python
---------

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

Find full example code at "examples/src/main/python/sql/basic.py" in the Spark repo.

Sql
---------

.. code-block:: SQL

  CREATE GLOBAL TEMPORARY VIEW temp_view AS SELECT a + 1, b * 2 FROM tbl
  SELECT * FROM global_temp.temp_view


创建 Dataset
==============================

Datasets are similar to RDDs, however, instead of using Java serialization or Kryo they use a specialized Encoder to serialize the objects for processing or transmitting over the network. While both encoders and standard serialization are responsible for turning an object into bytes, encoders are code generated dynamically and use a format that allows Spark to perform many operations like filtering, sorting and hashing without deserializing the bytes back into an object.

Scala
---------

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

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" in the Spark repo.

Java
---------

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

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" in the Spark repo.


与 RDD 互操作
==============================

Spark SQL supports two different methods for converting existing RDDs into Datasets. The first method uses reflection to infer the schema of an RDD that contains specific types of objects. This reflection based approach leads to more concise code and works well when you already know the schema while writing your Spark application.

The second method for creating Datasets is through a programmatic interface that allows you to construct a schema and then apply it to an existing RDD. While this method is more verbose, it allows you to construct Datasets when the columns and their types are not known until runtime.

Inferring the Schema Using Reflection
-----------------------------------------

Scala
^^^^^^^

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

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" in the Spark repo.


Java
^^^^^^^

.. code-block:: Java

Spark SQL supports automatically converting an RDD of JavaBeans into a DataFrame. The BeanInfo, obtained using reflection, defines the schema of the table. Currently, Spark SQL does not support JavaBeans that contain Map field(s). Nested JavaBeans and List or Array fields are supported though. You can create a JavaBean by creating a class that implements Serializable and has getters and setters for all of its fields.

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

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" in the Spark repo.


Python
^^^^^^^

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

Find full example code at "examples/src/main/python/sql/basic.py" in the Spark repo.


Programmatically Specifying the Schema
-----------------------------------------

Scala
^^^^^^^

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

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" in the Spark repo.


Java
^^^^^^^

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

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" in the Spark repo.


Python
^^^^^^^

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

Find full example code at "examples/src/main/python/sql/basic.py" in the Spark repo.


聚合
==============================

The built-in DataFrames functions provide common aggregations such as count(), countDistinct(), avg(), max(), min(), etc. While those functions are designed for DataFrames, Spark SQL also has type-safe versions for some of them in Scala and Java to work with strongly typed Datasets. Moreover, users are not limited to the predefined aggregate functions and can create their own.

Untyped User-Defined Aggregate Functions
----------------------------------------------

Users have to extend the UserDefinedAggregateFunction abstract class to implement a custom untyped aggregate function. For example, a user-defined average can look like:

Scala
^^^^^^^^^^

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

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/UserDefinedUntypedAggregation.scala" in the Spark repo.


Java
^^^^^^^^^^

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

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaUserDefinedUntypedAggregation.java" in the Spark repo.


Type-Safe User-Defined Aggregate Functions
----------------------------------------------

User-defined aggregations for strongly typed Datasets revolve around the Aggregator abstract class. For example, a type-safe user-defined average can look like:

Scala
^^^^^^^^^^^^

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

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/UserDefinedTypedAggregation.scala" in the Spark repo.


Java
^^^^^^^^^^^^

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

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaUserDefinedTypedAggregation.java" in the Spark repo.


*****************
数据源
*****************

Spark SQL supports operating on a variety of data sources through the DataFrame interface. A DataFrame can be operated on using relational transformations and can also be used to create a temporary view. Registering a DataFrame as a temporary view allows you to run SQL queries over its data. This section describes the general methods for loading and saving data using the Spark Data Sources and then goes into specific options that are available for the built-in data sources.


Generic Load/Save Functions
==============================

In the simplest form, the default data source (parquet unless otherwise configured by spark.sql.sources.default) will be used for all operations.

Scala
^^^^^^^^

.. code-block:: Scala

  val usersDF = spark.read.load("examples/src/main/resources/users.parquet")
  usersDF.select("name", "favorite_color").write.save("namesAndFavColors.parquet")

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" in the Spark repo.

Java
^^^^^^^^

.. code-block:: Java

  Dataset<Row> usersDF = spark.read().load("examples/src/main/resources/users.parquet");
  usersDF.select("name", "favorite_color").write().save("namesAndFavColors.parquet");

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" in the Spark repo.

Python
^^^^^^^^

.. code-block:: Python

  df = spark.read.load("examples/src/main/resources/users.parquet")
  df.select("name", "favorite_color").write.save("namesAndFavColors.parquet")

Find full example code at "examples/src/main/python/sql/datasource.py" in the Spark repo.

R
^^^^^^^^

.. code-block:: R

  df <- read.df("examples/src/main/resources/users.parquet")
  write.df(select(df, "name", "favorite_color"), "namesAndFavColors.parquet")

Find full example code at "examples/src/main/r/RSparkSQLExample.R" in the Spark repo.


Manually Specifying Options
---------------------------------

You can also manually specify the data source that will be used along with any extra options that you would like to pass to the data source. Data sources are specified by their fully qualified name (i.e., org.apache.spark.sql.parquet), but for built-in sources you can also use their short names (json, parquet, jdbc, orc, libsvm, csv, text). DataFrames loaded from any data source type can be converted into other types using this syntax.

Scala
^^^^^^^

.. code-block:: Scala

  val peopleDF = spark.read.format("json").load("examples/src/main/resources/people.json")
  peopleDF.select("name", "age").write.format("parquet").save("namesAndAges.parquet")

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" in the Spark repo.

Java
^^^^^^^

.. code-block:: Java

  Dataset<Row> peopleDF =
    spark.read().format("json").load("examples/src/main/resources/people.json");
  peopleDF.select("name", "age").write().format("parquet").save("namesAndAges.parquet");

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" in the Spark repo.

Python
^^^^^^^

.. code-block:: Python

  df = spark.read.load("examples/src/main/resources/people.json", format="json")
  df.select("name", "age").write.save("namesAndAges.parquet", format="parquet")

Find full example code at "examples/src/main/python/sql/datasource.py" in the Spark repo.

R
^^^^^^^

.. code-block:: R

  df <- read.df("examples/src/main/resources/people.json", "json")
  namesAndAges <- select(df, "name", "age")
  write.df(namesAndAges, "namesAndAges.parquet", "parquet")

Find full example code at "examples/src/main/r/RSparkSQLExample.R" in the Spark repo.


Run SQL on files directly
---------------------------------

Instead of using read API to load a file into DataFrame and query it, you can also query that file directly with SQL.

Scala
^^^^^^^

.. code-block:: Scala

  val sqlDF = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" in the Spark repo.


Java
^^^^^^^

.. code-block:: Java

  Dataset<Row> sqlDF =
    spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`");

Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" in the Spark repo.

Python
^^^^^^^

.. code-block:: Python

  df = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")

Find full example code at "examples/src/main/python/sql/datasource.py" in the Spark repo.

R
^^^^^^^

.. code-block:: R

  df <- sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")

Find full example code at "examples/src/main/r/RSparkSQLExample.R" in the Spark repo.


Save Modes
---------------------------------

Save operations can optionally take a SaveMode, that specifies how to handle existing data if present. It is important to realize that these save modes do not utilize any locking and are not atomic. Additionally, when performing an Overwrite, the data will be deleted before writing out the new data.

Scala/Java	Any Language	Meaning
SaveMode.ErrorIfExists (default)	"error" (default)	When saving a DataFrame to a data source, if data already exists, an exception is expected to be thrown.
SaveMode.Append	"append"	When saving a DataFrame to a data source, if data/table already exists, contents of the DataFrame are expected to be appended to existing data.
SaveMode.Overwrite	"overwrite"	Overwrite mode means that when saving a DataFrame to a data source, if data/table already exists, existing data is expected to be overwritten by the contents of the DataFrame.
SaveMode.Ignore	"ignore"	Ignore mode means that when saving a DataFrame to a data source, if data already exists, the save operation is expected to not save the contents of the DataFrame and to not change the existing data. This is similar to a CREATE TABLE IF NOT EXISTS in SQL.

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

Scala
^^^^^^^

.. code-block:: Scala

  peopleDF.write.bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed")

Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" in the Spark repo.
while partitioning can be used with both save and saveAsTable when using the Dataset APIs.

Scala
Java
Python
Sql
usersDF.write.partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet")
Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" in the Spark repo.
It is possible to use both partitioning and bucketing for a single table:

Scala
Java
Python
Sql
peopleDF
  .write
  .partitionBy("favorite_color")
  .bucketBy(42, "name")
  .saveAsTable("people_partitioned_bucketed")
Find full example code at "examples/src/main/scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala" in the Spark repo.
partitionBy creates a directory structure as described in the Partition Discovery section. Thus, it has limited applicability to columns with high cardinality. In contrast bucketBy distributes data across a fixed number of buckets and can be used when a number of unique values is unbounded.

Java
^^^^^^^

peopleDF.write().bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed");
Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" in the Spark repo.
while partitioning can be used with both save and saveAsTable when using the Dataset APIs.

Scala
Java
Python
Sql
usersDF
  .write()
  .partitionBy("favorite_color")
  .format("parquet")
  .save("namesPartByColor.parquet");
Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" in the Spark repo.
It is possible to use both partitioning and bucketing for a single table:

Scala
Java
Python
Sql
peopleDF
  .write()
  .partitionBy("favorite_color")
  .bucketBy(42, "name")
  .saveAsTable("people_partitioned_bucketed");
Find full example code at "examples/src/main/java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java" in the Spark repo.
partitionBy creates a directory structure as described in the Partition Discovery section. Thus, it has limited applicability to columns with high cardinality. In contrast bucketBy distributes data across a fixed number of buckets and can be used when a number of unique values is unbounded.


Python
^^^^^^^

df.write.bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed")
Find full example code at "examples/src/main/python/sql/datasource.py" in the Spark repo.
while partitioning can be used with both save and saveAsTable when using the Dataset APIs.

Scala
Java
Python
Sql
df.write.partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet")
Find full example code at "examples/src/main/python/sql/datasource.py" in the Spark repo.
It is possible to use both partitioning and bucketing for a single table:

Scala
Java
Python
Sql
df = spark.read.parquet("examples/src/main/resources/users.parquet")
(df
    .write
    .partitionBy("favorite_color")
    .bucketBy(42, "name")
    .saveAsTable("people_partitioned_bucketed"))
Find full example code at "examples/src/main/python/sql/datasource.py" in the Spark repo.
partitionBy creates a directory structure as described in the Partition Discovery section. Thus, it has limited applicability to columns with high cardinality. In contrast bucketBy distributes data across a fixed number of buckets and can be used when a number of unique values is unbounded.

Sql
^^^^^^^

CREATE TABLE users_bucketed_by_name(
  name STRING,
  favorite_color STRING,
  favorite_numbers array<integer>
) USING parquet
CLUSTERED BY(name) INTO 42 BUCKETS;
while partitioning can be used with both save and saveAsTable when using the Dataset APIs.

Scala
Java
Python
Sql
CREATE TABLE users_by_favorite_color(
  name STRING,
  favorite_color STRING,
  favorite_numbers array<integer>
) USING csv PARTITIONED BY(favorite_color);
It is possible to use both partitioning and bucketing for a single table:

Scala
Java
Python
Sql
CREATE TABLE users_bucketed_and_partitioned(
  name STRING,
  favorite_color STRING,
  favorite_numbers array<integer>
) USING parquet
PARTITIONED BY (favorite_color)
CLUSTERED BY(name) SORTED BY (favorite_numbers) INTO 42 BUCKETS;

partitionBy creates a directory structure as described in the Partition Discovery section. Thus, it has limited applicability to columns with high cardinality. In contrast bucketBy distributes data across a fixed number of buckets and can be used when a number of unique values is unbounded.



Parquet Files
==============================

Parquet is a columnar format that is supported by many other data processing systems. Spark SQL provides support for both reading and writing Parquet files that automatically preserves the schema of the original data. When writing Parquet files, all columns are automatically converted to be nullable for compatibility reasons.

JSON Datasets
==============================

Hive Tables
==============================

JDBC To Other Databases
==============================

Troubleshooting
==============================


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

运行 Spark SQL CLI
===================================


*****************
迁移指南
*****************

Spark SQL 从 2.1 版本升级到2.2 版本
=========================================

Spark SQL 从 2.0 版本升级到 2.1 版本
=========================================

Spark SQL 从 1.6 版本升级到 2.0 版本
=========================================

* SparkSession 现在是 Spark 新的切入点, 它替代了老的 SQLContext 和 HiveContext。注意：为了向下兼容, 老的 SQLContext 和 HiveContext 仍然保留。可以从 SparkSession 获取一个新的 catalog 接口- 现有的访问数据库和表的API, 如 listTables, createExternalTable, dropTempView, cacheTable 都被移到该接口。
* Dataset API 和 DataFrame API 进行了统一。在 Scala 中, DataFrame 变成了 Dataset[Row]的一个类型别名, 而Java API使用者必须将 DataFrame 替换成 Dataset<Row>。Dataset 类既提供了强类型转换操作 (如 map, filter 以及 groupByKey) 也提供了非强类型转换操作 (如 select 和 groupBy) 。由于编译期的类型安全不是 Python 和 R 语言的一个特性,  Dataset 的概念并不适用于这些语言的 API。相反, DataFrame 仍然是最基本的编程抽象, 就类似于这些语言中单节点数据帧的概念。
* Dataset 和 DataFrame API 中 unionAll 已经过时并且由 union 替代。
* Dataset 和 DataFrame API 中 explode 已经过时。或者 functions.explode() 可以结合 select 或 flatMap 一起使用。
* Dataset 和 DataFrame API 中 registerTempTable 已经过时并且由 createOrReplaceTempView 替代。

Spark SQL 从 1.5 版本升级到 1.6 版本
=========================================

Spark SQL 从 1.4 版本升级到 1.5 版本
=========================================

* 使用手动管理内存(Tungsten引擎)的执行优化以及用于表达式求值的代码自动生成现在默认是启用的。这些特性可以通过将spark.sql.tungsten.enabled的值设置为false来同时禁用。
* 默认不启用Parquet schema合并。可以将spark.sql.parquet.mergeSchema的值设置为true来重新启用。
* Python中对于列的字符串分解现在支持使用点号(.)来限定列或访问内嵌值，例如 df[‘table.column.nestedField’]。然而这也意味着如果你的列名包含任何点号(.)的话，你就必须要使用反引号来转义它们(例如：table.`column.with.dots`.nested)。
* 默认启用内存中列式存储分区修剪。可以通过设置 spark.sql.inMemoryColumarStorage.partitionPruning 值为false来禁用它。
* 不再支持无精度限制的decimal，相反, Spark SQL现在强制限制最大精度为38位。从BigDecimal对象推导schema时会使用（38，18）这个精度。如果在DDL中没有指定精度，则默认使用精度Decimal(10，0)。
* 存储的时间戳(Timestamp)现在精确到1us（微秒），而不是1ns（纳秒）
* 在 sql 方言中，浮点数现在被解析成decimal。HiveQL的解析保持不变。
* SQL/DataFrame函数的规范名称均为小写(例如：sum vs SUM)。
* JSON数据源不会再自动地加载其他应用程序创建的新文件（例如，不是由Spark SQL插入到dataset中的文件）。对于一个JSON持久化表（例如：存储在Hive metastore中的表的元数据），用户可以使用 REFRESH TABLE 这个SQL命令或者 HiveContext 的 refreshTable 方法来把新文件添加进表。对于一个表示JSON数据集的DataFrame, 用户需要重建这个 DataFrame, 这样新的 DataFrame 就会包含新的文件。
* pySpark 中的 DataFrame.withColumn 方法支持新增一列或是替换名称相同列。

Spark SQL 从 1.3 版本升级到 1.4 版本
=========================================

Spark SQL 从 1.0-1.2 版本升级到 1.3 版本
=========================================


兼容Apache Hive
=========================================

Spark SQL在设计时就考虑到了和Hive metastore，SerDes以及UDF之间的兼容性。目前 Hive SerDes 和 UDF 都是基于Hive 1.2.1版本，并且Spark SQL可以连接到不同版本的Hive metastore（从0.12.0到1.2.1，可以参考[与不同版本的Hive Metastore交互]）

在已有的Hive仓库中部署
-----------------------

Spark SQL Thrift JDBC server采用了开箱即用的设计以兼容已有的Hive安装版本。你不需要修改现有的Hive Metastore ,  或者改变数据的位置和表的分区。

支持的Hive功能
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
* 绝大多数Hive数据类型，包括：
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

不支持的Hive功能
-----------------------

以下是目前还不支持的Hive功能列表。在Hive部署中这些功能大部分都用不到。

Hive核心功能

* bucket：bucket是Hive表分区内的一个哈希分区，Spark SQL目前还不支持bucket。

Hive高级功能

* UNION 类型
* Unique join
* 列统计数据收集：Spark SQL目前不依赖扫描来收集列统计数据并且仅支持填充Hive metastore 的 sizeInBytes 字段。

Hive输入输出格式

* CLI文件格式：对于回显到CLI中的结果，Spark SQL仅支持TextOutputFormat。
* Hadoop archive

Hive优化

有少数Hive优化还没有包含在Spark中。其中一些（比如索引）由于Spark SQL的这种内存计算模型而显得不那么重要。另外一些在Spark SQL未来的版本中会持续跟踪。

* 块级别位图索引和虚拟列（用来建索引）
* 自动为join和groupBy计算reducer个数：目前在Spark SQL中，你需要使用 ”SET spark.sql.shuffle.partitions=[num_tasks];” 来控制后置混洗的并行程度。
* 仅查询元数据：对于只需要使用元数据的查询请求，Spark SQL仍需要启动任务来计算结果
* 数据倾斜标志：Spark SQL不遵循Hive中的数据倾斜标志
* STREAMTABLE join操作提示：Spark SQL不遵循 STREAMTABLE 提示。
* 对于查询结果合并多个小文件：如果返回的结果有很多小文件，Hive有个选项设置，来合并小文件，以避免超过HDFS的文件数额度限制。Spark SQL不支持这个。


*****************
参考
*****************

数据类型
=================

Spark SQL 和 DataFrame 支持下面的数据类型：

* 数值类型
    * ByteType: 表示1字节长的有符号整型，数值范围：-128 到 127.
    * ShortType: 表示2字节长的有符号整型，数值范围：-32768 到 32767.
    * IntegerType: 表示4字节长的有符号整型，数值范围：-2147483648 到 2147483647.
    * LongType: 表示8字节长的有符号整型，数值范围： -9223372036854775808 to 9223372036854775807.
    * FloatType: 表示4字节长的单精度浮点数。
    * DoubleType: 表示8字节长的双精度浮点数
    * DecimalType: 表示任意精度有符号带小数的数值。内部使用 java.math.BigDecimal, 一个 BigDecimal 由一个任意精度的整数非标度值和一个32位的整数标度 (scale) 组成。
* 字符串类型
    * StringType: 表示字符串值
* 二进制类型
    * BinaryType: 表示字节序列值
* 布尔类型
    * BooleanType: 表示布尔值
* 日期类型
    * TimestampType: 表示包含年月日、时分秒等字段的日期值
    * DateType: 表示包含年月日字段的日期值
* Complex types（复杂类型）
    * ArrayType(elementType, containsNull)：数组类型，表示一个由类型为elementType的元素组成的序列，containsNull用来表示ArrayType中的元素是否能为null值。
    * MapType(keyType, valueType, valueContainsNull)：映射类型，表示一个键值对的集合。键的类型由keyType表示，值的类型则由valueType表示。对于一个MapType值，键是不允许为null值。valueContainsNull用来表示一个MapType的值是否能为null值。
* StructType(fields)：表示由StructField序列描述的结构。
        * StructField(name, datatype, nullable): 表示 StructType 中的一个字段，name表示字段名，datatype是字段的数据类型，nullable用来表示该字段是否可以为空值。


NaN 语义
=================

当处理一些不符合标准浮点数语义的loat或double类型时，对于Not-a-Number(NaN)需要做一些特殊处理。具体如下：
* NaN = NaN返回true。
* 在聚合操作中，所有NaN值都被分到同一组。
* 在join key中NaN可以当做一个普通的值。
* NaN值在升序排序中排到最后，比任何其他数值都大。