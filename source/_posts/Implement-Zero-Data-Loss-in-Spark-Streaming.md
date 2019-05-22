---
title: Implement Zero Data Loss in Spark Streaming
date: 2019-05-22 18:19:14
tags:
---
This is a log from my own experience in spark streaming during my work. Base on different environment and servers, the strategy may vary. For here, I am working with **Kafka** and Google Cloud **PubSub**. <!-- more -->

## Background
I have such workflow that the spark streaming receives dataset from both Kafka and PubSub, after doing some clean up and modeling, push the data to the Cloud Datastore.
![my Workflow](sparkworkflow.JPG)
It works fine when the data stream is small and reports me the expected values; however while the number of end users is growing large, especially when the data stream turns to be erratic and sometimes considerably large if end users interact frequently with our UI pages, the spark streaming will get a lot of uncertain runtime errors. Those errors are most likely caused by the finite number of workers in our cluster. Also when there are too much stream rushing over to the server, the limited memory in cache will cause failures. After we upgraded the cluster on cloud, the number of errors is significantly reduced.
![CPU Utilization](cpudiagram.JPG)
But the data processed during the errors was permanently lost and cannot be recovered. 

```scala
val ssc = StreamingContext.createContext(...params)

def createContext(...params): StreamingContext = {
    val sparkConf = new SparkConf().setAppName("KpiAnalysis")
                                   .set("spark.streaming.receiver.writeAheadLog.enable","true")    
    val ssc = new StreamingContext(sparkConf, Seconds(1))
    ...
    ssc.checkpoint(CHECK_POINT_DIR)
    ssc
}
```

Set the spark automatically clean checkpoints to save disk memory.
```scala
val ssc = StreamingContext.createContext(...params)

def createContext(...params): StreamingContext = {
    val sparkConf = new SparkConf().setAppName("KpiAnalysis")
                                   .set("spark.streaming.receiver.writeAheadLog.enable","true")
                                   .set("spark.cleaner.referenceTracking.cleanCheckpoints", "true")    
    val ssc = new StreamingContext(sparkConf, Seconds(1))
    ...
    ssc.checkpoint(CHECK_POINT_DIR)
    ssc
}

```
This will not clean the latest checkpoint as the it is still referred to recover from dirver failures.



```scala
val ssc = StreamingContext.getOrCreate(CHECK_POINT_DIR, () => createContext(...params))

def createContext(...params): StreamingContext = {
    val sparkConf = new SparkConf().setAppName("KpiAnalysis")
                                   .set("spark.streaming.receiver.writeAheadLog.enable","true")
                                   .set("spark.cleaner.referenceTracking.cleanCheckpoints", "true")    
    val ssc = new StreamingContext(sparkConf, Seconds(1))
    ...
    ssc.checkpoint(CHECK_POINT_DIR)
    ssc
}
```


After switch to `getOrCreate()`, I had the following exception:
```
Exception in thread "main" org.apache.spark.SparkException: Failed to read checkpoint from directory checkpointDir
    at org.apache.spark.streaming.CheckpointReader$.read(Checkpoint.scala:368)
	at org.apache.spark.streaming.StreamingContext$.getOrCreate(StreamingContext.scala:827)
    ...
Caused by: java.io.IOException: java.lang.ClassCastException: cannot assign instance of com.some.project$$anonfun$1 to field org.apache.spark.streaming.dstream.MappedDStream.org$apache$spark$streaming$dstream$MappedDStream$$mapFunc of type scala.Function1 in instance of org.apache.spark.streaming.dstream.MappedDStream     
	at org.apache.spark.util.Utils$.tryOrIOException(Utils.scala:1310)
	at org.apache.spark.streaming.DStreamGraph.readObject(DStreamGraph.scala:194)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    ...
```
This can be solved by deleting the original checkpoint in HDFS locals (manually).


