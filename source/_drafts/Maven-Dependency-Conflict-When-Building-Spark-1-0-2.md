---
title:  Spark 1.0.2 Build Error
date: 2017-12-28 23:20:30
tags: [Spark]
categories: [Spark]
---

I want to learn the source code of Apache Spark, so I choose an early version to read and build it, referring to the official document [Building Spark with Maven](https://spark.apache.org/docs/1.0.2/building-with-maven.html). When running the test cases in Spark-Core, everything works well, but I met two problems when running Spark-Example.

# java.lang.NoClassDefFoundError: org/apache/spark/SparkConf
When running SparkPi, there is a runtime error:
```
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/spark/SparkConf
    at org.apache.spark.examples.SparkPi$.main(SparkPi.scala:27)
    at org.apache.spark.examples.SparkPi.main(SparkPi.scala)
Caused by: java.lang.ClassNotFoundException: org.apache.spark.SparkConf
    at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
```

I found the solution in [Apache Spark Developers List
](http://apache-spark-developers-list.1001551.n3.nabble.com/IntelliJ-Runtime-error-td11383.html).

Change the spark dependences in the spark-example model from "provided" to "compile". The modified maven POM.xml file of spark-example should be:
```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-core_${scala.binary.version}</artifactId>
    <version>${project.version}</version>
    <scope>compile</scope>
</dependency>
```


# Maven Dependency Conflict

Having solved the first error, I met a new error:
```
Exception in thread "main" java.lang.SecurityException: class "javax.servlet.FilterRegistration"'s signer information does not match signer information of other classes in the same package
```

The problem is caused by a conflict of servlet and there are lots of articles illustrating it. The solution is to remove the dependency of javax.servlet. Removed the following dependency of POM.xml for Spark-Example:
```xml
//Remove this 
<dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-server</artifactId>
</dependency>
```