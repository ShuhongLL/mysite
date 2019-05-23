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
But the data processed during the errors was permanently lost and cannot be recovered. The data running on spark is buffered in memory (cache) and will be cleared meanwhile a failure occurs. This is not desired since some valuable KPIs may be lost as well. To prevent such data loss, I tried different strategies.
</br>

## Checkpoint
Checkpointing is a process supported by spark streaming after version which will save RDDs in log after being checkpointed. There are two level of checkpoints: reliable and local. **Reliable checkpoint** ensures that the data is stored permanentlly on HDFS, S3 or other distributed filesystems. Each job on cluster will create a directory for saving checkpointed logs, which will look pretty much like the directory below:
```
├── "SomeUUID"
│   └── rdd-#
│       ├── part-timestamp1
│       ├── .part-timestamp1.crc
│       ├── part-timestamp2
│       └── .part-timestamp2.crc
```
Since the data stream will be replicated on disk, the performance will slow down due to file I/O. **Local checkpoint** privileges performance over fault-tolerance which will persist RDDs on local storage in the executors. Read or write will be faster in this case; however if a driver fails, the data not yet executed may not be recoverable. As default, the data storeage level is set to `MEMORY_AND_DISK` which saves data in cache and disk (some in cache and some in disk). For here I changed to `StorageLevel.MEMORY_AND_DISK_SER_2` (more details can be referred to [here](https://stackoverflow.com/questions/30520428/what-is-the-difference-between-memory-only-and-memory-and-disk-caching-level-in)). This driver recovery mechanism is sufficient to ensure zero data loss if all data was reliably store in the system. However for my circumstance, some of the data buffered in memory could be lost. When the driver process fails, all the executors running are killed as well, along with any data in their memory. This pushes me to look for something more advanced.
</br>

## Write Ahead Logs



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

```scala
def stop(stopSparkContext: Boolean, stopGracefully: Boolean): Unit
```

```
ERROR ReceiverTracker: Deregistered receiver for stream 0: Stopped by driver
WARN BlockGenerator: Cannot stop BlockGenerator as its not in the Active state [state = StoppedAll]     
WARN BatchedWriteAheadLog: BatchedWriteAheadLog Writer queue interrupted.
```
are not recoverable across applications or Spark upgrades and hence not very reliable

## Kafka Direct API