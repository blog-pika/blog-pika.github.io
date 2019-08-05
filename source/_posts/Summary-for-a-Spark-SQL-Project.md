---
title: Summary for a Spark SQL Project
date: 2017-12-25 11:25:29
tags: [Spark, Scala, Spark SQL]
categories: [Coding Notes]
---

This article records some mistakes I made when doing the coding assignment from course [Big Data Analysis with Scala and Spark](https://www.coursera.org/learn/scala-spark-big-data/programming/T19Ec/time-usage). The assignment requires us to analyze data with Spark SQL.

# Create Row

When creating a Row object, the constructer requires unpacked list of arguments, rather than a list.
Wrong version:

```scala
def row(line: List[String]): Row = {
    val record = line.zipWithIndex.map(_.swap).map{
      case (index, rowValue) => if(index == 0) rowValue.trim else rowValue.toDouble
    }

    // Row([List(foo, 1.0, 2.0)]) is wrong
    Row(record.toArray)
}
```

Correct Version:

```scala
def row(line: List[String]): Row = {
    val record = line.zipWithIndex.map(_.swap).map{
      case (index, rowValue) => if(index == 0) rowValue.trim else rowValue.toDouble
    }

    //Row(foo,1.0,2.0) is correct
    Row(record: _*)
}
```

# Sum Different Columns

The code is as follow, it sums all the columns in each list to get a new column. For more detail you can refer to [Adding a column of rowsums across a list of columns in Spark Dataframe
](https://stackoverflow.com/questions/37624699/adding-a-column-of-rowsums-across-a-list-of-columns-in-spark-dataframe).

```scala
def timeUsageSummary(
    primaryNeedsColumns: List[Column],
    workColumns: List[Column],
    otherColumns: List[Column],
    df: DataFrame
  ): DataFrame = {
    // ... some unrealated codes
    def sum_(cols: Column*) = cols.foldLeft(lit(0))(_ + _)

    val primaryNeedsProjection: Column = (sum_(primaryNeedsColumns: _*) / 60).as("primaryNeeds")
    val workProjection: Column = (sum_(workColumns: _*) / 60).as("work")
    val otherProjection: Column = (sum_(otherColumns: _*) / 60).as("other")
    df
      .select(workingStatusProjection, sexProjection, ageProjection, primaryNeedsProjection, workProjection, otherProjection)
      .where($"telfs" <= 4) // Discard people who are not in labor force
  }
```

> By the way, the Column object does not contain any data. Instead, it contains the column name and the opeations associated with this column, in the form of SQL statements. When it is called by dataframe, these SQL statements will be executed.

<img src="https://bytebucket.org/LarryTaoWang/pictureofblog/raw/a8a037eb56415c8432b523e6358b28d43ca7065d/Spark/SparkSQL-Column.png">

# Convert DataFrame to DataSet
[Convert DataFrame to DataSet](https://stackoverflow.com/questions/44516627/how-to-convert-a-dataframe-to-dataset-in-apache-spark-in-scala)

# Reference

> [1][Is there a Scala equivalent of the Python list unpack (a.k.a. “*”) operator?
](https://stackoverflow.com/questions/37624699/adding-a-column-of-rowsums-across-a-list-of-columns-in-spark-dataframe)
> [2][Adding a column of rowsums across a list of columns in Spark Dataframe
](https://stackoverflow.com/questions/37624699/adding-a-column-of-rowsums-across-a-list-of-columns-in-spark-dataframe)
> [3][Convert DataFrame to DataSet](https://stackoverflow.com/questions/44516627/how-to-convert-a-dataframe-to-dataset-in-apache-spark-in-scala)