---
title: Storm SQL integration
layout: documentation
documentation: true
---

The Storm SQL integration allows users to run SQL queries over streaming data in Storm. Not only the SQL interface allows faster development cycles on streaming analytics, but also opens up the opportunities to unify batch data processing like [Apache Hive](///hive.apache.org) and real-time streaming data analytics.

At a very high level StormSQL compiles the SQL queries to [Trident](Trident-API-Overview.html) topologies and executes them in Storm clusters. This document provides information of how to use StormSQL as end users. For people that are interested in more details in the design and the implementation of StormSQL please refer to the [this](storm-sql-internal.html) page.

## Usage

Run the ``storm sql`` command to compile SQL statements into Trident topology, and submit it to the Storm cluster

```
$ bin/storm sql <sql-file> <topo-name>
```

In which `sql-file` contains a list of SQL statements to be executed, and `topo-name` is the name of the topology.


## Supported Features

The following features are supported in the current repository:

* Streaming from and to external data sources
* Filtering tuples
* Projections
* Aggregations (Grouping)
* User defined function (scalar and aggregate)

## Specifying External Data Sources

In StormSQL data is represented by external tables. Users can specify data sources using the `CREATE EXTERNAL TABLE` statement. The syntax of `CREATE EXTERNAL TABLE` closely follows the one defined in [Hive Data Definition Language](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL):

```
CREATE EXTERNAL TABLE table_name field_list
    [ STORED AS
      INPUTFORMAT input_format_classname
      OUTPUTFORMAT output_format_classname
    ]
    LOCATION location
    [ TBLPROPERTIES tbl_properties ]
    [ AS select_stmt ]
```

You can find detailed explanations of the properties in [Hive Data Definition Language](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL). For example, the following statement specifies a Kafka spouts and sink:

```
CREATE EXTERNAL TABLE FOO (ID INT PRIMARY KEY) LOCATION 'kafka://localhost:2181/brokers?topic=test' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
```

## Plugging in External Data Sources

Users plug in external data sources through implementing the `ISqlTridentDataSource` interface and registers them using the mechanisms of Java's service loader. The external data source will be chosen based on the scheme of the URI of the tables. Please refer to the implementation of `storm-sql-kafka` for more details.

## Specifying User Defined Function (UDF)

Users can define user defined function (scalar or aggregate) using `CREATE FUNCTION` statement.
For example, the following statement defines `MYPLUS` function which uses `org.apache.storm.sql.TestUtils$MyPlus` class.

```
CREATE FUNCTION MYPLUS AS 'org.apache.storm.sql.TestUtils$MyPlus'
```

Storm SQL determines whether the function as scalar or aggregate by checking which methods are defined.
If the class defines `evaluate` method, Storm SQL treats the function as `scalar`,
and if the class defines `add` method, Storm SQL treats the function as `aggregate`.

Example of class for scalar function is here:

```
  public class MyPlus {
    public static Integer evaluate(Integer x, Integer y) {
      return x + y;
    }
  }

```

and class for aggregate function is here:

```
  public class MyConcat {
    public static String init() {
      return "";
    }
    public static String add(String accumulator, String val) {
      return accumulator + val;
    }
    public static String result(String accumulator) {
      return accumulator;
    }
  }
```

If users doesn't define `result` method, result is the last return value of `add` method.
Users need to define `result` method only when we need to transform accumulated value.

## Example: Filtering Kafka Stream

Let's say there is a Kafka stream that represents the transactions of orders. Each message in the stream contains the id of the order, the unit price of the product and the quantity of the orders. The goal is to filter orders where the transactions are significant and to insert these orders into another Kafka stream for further analysis.

The user can specify the following SQL statements in the SQL file:

```
CREATE EXTERNAL TABLE ORDERS (ID INT PRIMARY KEY, UNIT_PRICE INT, QUANTITY INT) LOCATION 'kafka://localhost:2181/brokers?topic=orders' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
CREATE EXTERNAL TABLE LARGE_ORDERS (ID INT PRIMARY KEY, TOTAL INT) LOCATION 'kafka://localhost:2181/brokers?topic=large_orders' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
INSERT INTO LARGE_ORDERS SELECT ID, UNIT_PRICE * QUANTITY AS TOTAL FROM ORDERS WHERE UNIT_PRICE * QUANTITY > 50
```

The first statement defines the table `ORDER` which represents the input stream. The `LOCATION` clause specifies the ZkHost (`localhost:2181`), the path of the brokers in ZooKeeper (`/brokers`) and the topic (`orders`). The `TBLPROPERTIES` clause specifies the configuration of [KafkaProducer](http://kafka.apache.org/documentation.html#producerconfigs).
Current implementation of `storm-sql-kafka` requires specifying both `LOCATION` and `TBLPROPERTIES` clauses even though the table is read-only or write-only.

Similarly, the second statement specifies the table `LARGE_ORDERS` which represents the output stream. The third statement is a `SELECT` statement which defines the topology: it instructs StormSQL to filter all orders in the external table `ORDERS`, calculates the total price and inserts matching records into the Kafka stream specified by `LARGE_ORDER`.

To run this example, users need to include the data sources (`storm-sql-kafka` in this case) and its dependency in the
class path. Dependencies for Storm SQL are automatically handled when users run `storm sql`. Users can include data sources at the submission step like below:

```
$ bin/storm sql order_filtering.sql order_filtering --artifacts "org.apache.storm:storm-sql-kafka:2.0.0-SNAPSHOT,org.apache.storm:storm-kafka:2.0.0-SNAPSHOT,org.apache.kafka:kafka_2.10:0.8.2.2\!org.slf4j:slf4j-log4j12,org.apache.kafka:kafka-clients:0.8.2.2"
```

Above command submits the SQL statements to StormSQL. Users need to modify each artifacts' version if users are using different version of Storm or Kafka. 

By now you should be able to see the `order_filtering` topology in the Storm UI.

## Current Limitations

Aggregation, windowing and joining tables are yet to be implemented. Specifying parallelism hints in the topology is not yet supported. All processors have a parallelism hint of 1.

The current implementation of the Kafka connector in StormSQL assumes both the input and the output are in JSON formats. The connector has not yet recognized the `INPUTFORMAT` and `OUTPUTFORMAT` clauses yet.
