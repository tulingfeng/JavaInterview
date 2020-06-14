# spark-demo

## 1.spark demo

1.`xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>SparkDemo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <spark.version>2.4.3</spark.version>
        <scala.version>2.11</scala.version>

    </properties>

    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-sql -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-catalyst -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-catalyst_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-core -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.scala-lang/scala-library -->
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>2.11.12</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <encoding>utf-8</encoding>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```



2.`java`

```java
package org.tlf.spark;

import org.apache.spark.sql.*;
import org.apache.spark.sql.streaming.StreamingQuery;
import org.apache.spark.sql.streaming.StreamingQueryException;

import java.util.Arrays;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.Dataset;

public class SparkDemo {
    public static void main(String[] args){
//        SparkSession spark = SparkSession
//                .builder()
//                .appName("JavaStructuredNetworkWordCount")
//                .getOrCreate();
//
//        // Create DataFrame representing the stream of input lines from connection to localhost:9999
//        Dataset<Row> lines = spark
//                .readStream()
//                .format("socket")
//                .option("host", "localhost")
//                .option("port", 9999)
//                .load();
//
//        // Split the lines into words
//        Dataset<String> words = lines
//                .as(Encoders.STRING())
//                .flatMap((FlatMapFunction<String, String>) x -> Arrays.asList(x.split(" ")).iterator(), Encoders.STRING());
//
//        // Generate running word count
//        Dataset<Row> wordCounts = words.groupBy("value").count();
//
//        StreamingQuery query = wordCounts.writeStream()
//                .outputMode("complete")
//                .format("console")
//                .start();
//
//        query.awaitTermination();
        String logFile = "/Users/tulingfeng/IdeaProjects/SparkDemo/ab.txt"; // Should be some file on your system
        SparkSession spark = SparkSession.builder().appName("Simple Application").getOrCreate();
        Dataset<String> logData = spark.read().textFile(logFile).cache();

        long num0s = logData.filter(s -> s.contains("0")).count();
        long num1s = logData.filter(s -> s.contains("1")).count();

        System.out.println("Lines with 0: " + num0s + ", lines with 1: " + num1s);

        spark.stop();
    }
}
```



3.`spark-submit`

```shell
spark-submit --class org.tlf.spark.SparkDemo --master spark://192.168.0.102:7077 SparkDemo-1.0-SNAPSHOT.jar
```



其中`Mac`上需要安装[spark-bin-hadoop](<https://archive.apache.org/dist/spark/spark-2.4.3/>)

具体安装步骤参见：[install spark local](<https://blog.51cto.com/6193721/2163571?source=dra>)



## 2.kafka structured streaming

[参考](<http://itechseeker.com/en/tutorials-2/apache-kafka/writing-applications-with-apache-kafka/integrating-kafka-with-spark-using-structured-streaming/>)

1.`xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>SparkDemo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <spark.version>2.4.3</spark.version>
        <scala.version>2.11</scala.version>

    </properties>

    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-sql -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-sql-kafka-0-10 -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql-kafka-0-10_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
            <version>2.2.4.RELEASE</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-catalyst -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-catalyst_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-core -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.scala-lang/scala-library -->
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>2.11.12</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-common -->
<!--        <dependency>-->
<!--            <groupId>org.apache.hadoop</groupId>-->
<!--            <artifactId>hadoop-common</artifactId>-->
<!--            <version>2.7.0</version>-->
<!--        </dependency>-->


    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <encoding>utf-8</encoding>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```



2.`java`

```java
package org.tlf.spark;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.streaming.StreamingQuery;
import org.apache.spark.sql.streaming.StreamingQueryException;

import java.util.Arrays;

public class SparkTest {
    public static void main(String[] args) throws StreamingQueryException {
        // Define a Spark Session
        SparkSession spark = SparkSession
                .builder()
                .appName("Spark Kafka Integration using Structured Streaming")
                .master("local")
                .getOrCreate();

        //Subscribe to topic 'test'
        Dataset< Row > ds = spark
                .readStream()
                .format("kafka")
                .option("kafka.bootstrap.servers", "120.27.71.78:9092,120.27.71.78:9093,120.27.71.78:9094")
                .option("subscribe", "spark-test")
                .load();

        // Start running the query that prints the data getting from Kafka 'test' topic
        StreamingQuery query = ds.writeStream()
                .outputMode("append")
                .format("console")
                .start();

        //Getting the data value as String
        Dataset lines=ds.selectExpr("CAST(value AS STRING)");

        // Split the lines into words
        Dataset<String> words = lines.as(Encoders.STRING())
                .flatMap((FlatMapFunction<String, String>) x -> Arrays.asList(x.split(" ")).iterator(), Encoders.STRING());

        // Count the occurrence of each word
        Dataset<Row> wordCounts = words.groupBy("value").count();

        // Start running the query that prints the word count
        StreamingQuery query1 = wordCounts.writeStream()
                .outputMode("complete")
                .format("console")
                .start();

        //Await the termination of each query
        query.awaitTermination();
        query1.awaitTermination();
    }
}
```



3.`spark-submit`

```shell
spark-submit --packages org.apache.spark:spark-sql-kafka-0-10:2.4.3 --class org.tlf.spark.SparkTest --master spark://192.168.0.102:7077 SparkDemo-1.0-SNAPSHOT.jar
```

