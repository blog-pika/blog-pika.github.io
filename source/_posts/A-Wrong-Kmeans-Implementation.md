---
title: A Wrong K-Means Implementation
date: 2017-12-23 20:14:51
tags: [K-Means, Spark, Scala]
categories: [Coding Notes]
---

This article records a mistake I made when doing the coding assignment from course [Big Data Analysis with Scala and Spark](https://www.coursera.org/learn/scala-spark-big-data/programming/FWGnz/stackoverflow-2-week-long-assignment). The assignment requires us to implement a K-Means algorithm with Spark to analyze the data from StackOverflow.

The general algorithm for K-Means clustering is as follow:
```
Randomly initialize K cluster centroids μ1,μ2,...,μk
Repeat{
    // Update Cluster 
    for i = 1 to m 
        ci = index of cluster centroid closest to xi;
    //Update Cluster Average
    for k = 1 to K
        μk = average if points assigned to cluster k
}
```

I made a mistake when updating the average of the new cluster.
Here is my code:
```scala
@tailrec final def kmeans(means: Array[(Int, Int)], vectors: RDD[(Int, Int)], iter: Int = 1, debug: Boolean = false): Array[(Int, Int)] = {
    val updated = vectors.map(node => (findClosest(node, means), node))
      .groupByKey()
      .mapValues(nodes => averageVectors(nodes))
      .collect()
      .sortBy(_._1)
 
    val newMeans = updated.map(_._2)
 
   // Use newMeans to do a new iteration
  }
```

"means" is the previous array of centroids, and the "updated" is the new one after an iteration. **Some centroids may have zero associated nodes**, in that case, an assignment will lose these centroids. For example, there are 45 initial centroids, when debugging, I found that the first iteration loses centroid 5.

The correct way is using the new centroids to update the previous centroids.
```scala
@tailrec final def kmeans(means: Array[(Int, Int)], vectors: RDD[(Int, Int)], iter: Int = 1, debug: Boolean = false): Array[(Int, Int)] = {
    val newMeans = means.clone()

    val updated = vectors.map(node => (findClosest(node, means), node))
      .groupByKey()
      .mapValues(nodes => averageVectors(nodes))
      .collect()
      .foreach(x => newMeans(x._1) = x._2)
      
    // Use newMeans to do a new iteration
}
```