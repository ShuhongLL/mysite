---
title: Implement Zero Data Loss in Spark Streaming
date: 2019-05-22 18:19:14
tags: [Spark, Kafka]
photos: ["../images/spark_kafka.JPG"]
---
This is a log from my own experience in spark streaming during my work. Base on different environment and servers, the strategy may vary. For here, I am working with **Kafka** and Google Cloud **PubSub**. <!-- more -->

## Background
I have such workflow that the spark streaming receives dataset from both Kafka and PubSub, after doing some clean up and modeling, push the data to the Cloud Datastore.
![my Workflow](sparkworkflow.png)
It works fine when the data stream is small and reports me the expected values; however while the number of end users is growing large, especially when the data stream turns to be erratic and sometimes considerably large if end users interact frequently with our UI pages, the spark streaming will get a lot of uncertain runtime errors. Those errors are most likely caused by the finite number of workers in our cluster. Also when there are too much stream rushing over to the server, the limited memory in cache will cause failures. After we upgraded the cluster on cloud, the number of errors is significantly reduced.
![CPU Utilization](cpudiagram.png)
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
Since the data stream will be replicated on disk, the performance will slow down due to file I/O. **Local checkpoint** privileges performance over fault-tolerance which will persist RDDs on local storage in the executors. Read or write will be faster in this case; however if a driver fails, the data not yet executed may not be recoverable. As default, the data storeage level is set to `MEMORY_AND_DISK` which saves data in cache and disk (some in cache and some in disk). For here I changed to `MEMORY_AND_DISK_SER_2` (more details can be referred to [here](https://stackoverflow.com/questions/30520428/what-is-the-difference-between-memory-only-and-memory-and-disk-caching-level-in)). The different is that, unlike **cache** only, the checkpoints doesn't save DAG with all the parents of RDDs; instead, they only save particular RDDs and remain for a longer time than cache. The time of persistance is strictly related to the executed computation and ends with the end of the spark application. To apply the checkpoint machanism, you just simply need to set the checkpoint directory when you are creating the **StreamingContext**
```scala
def createContext(...params): StreamingContext = {
    val sparkConf = new SparkConf().setAppName("KpiAnalysis")   
     val ssc = new StreamingContext(sparkConf, Seconds(1))
     ...
     ssc.checkpoint(CHECK_POINT_DIR)
     ssc
}
```
and before an action is operated on RDD:
```scala
    sRDD.checkpoint()
    sRDD.foreachPartition { partitionOfRecords => {     
        ...
        }
    }
```
and `sRDD.isCheckpointed()` will return **true**. For cleaning, the RDDs stored in cache will be cleaned with all other memory after the whole spark application is finished or terminated; the reliable RDDs stored on disk can be cleaned manually or set 
`spark.cleaner.referenceTracking.cleanCheckpoints` property to be **true** to enable automatic cleaning. This driver recovery mechanism is sufficient to ensure zero data loss if all data was reliably store in the system. However for my circumstance, the data is read from **kafka** and some of the data buffered in memory could be lost. If the driver process fails, all the executors running will be killed as well, along with any data in their memory. This pushes me to look for other mechanisms which are more advanced.
</br>

## Write Ahead Logs
Write Ahead Logs are used in database and file systems to ensure the durability of any data operations. The intention of the operation is first written down into a durable log , and then the operation is applied to the data. If the system fails in the middle of applying the operation, it can recover by reading the log and reapplying the operations it had intended to do.

Spark streaming uses **Receiver** to read data from **Kafka**. They run as long-running tasks in the executors and store the revecived data in the memory of the executors. If you enable the **checkpoint**, the data will be checkpointed either in cache or disk **in executors** before porceed into the application drivers. Unlike **checkpoint**, applying **WAL** will instead backup the recevied data in an **external fault-tolerant filesystem**. And after the executor batches the received data and sends to the driver, **WAL** supports another log to store the block metadata into external filesystem before being executed.
![https://databricks.com/blog/2015/01/15/improved-driver-fault-tolerance-and-zero-data-loss-in-spark-streaming.html](wal_spark.png)
(diagram from https://databricks.com/blog/2015/01/15/improved-driver-fault-tolerance-and-zero-data-loss-in-spark-streaming.html)
Spark streaming starts supporting WAL after version 1.2 and can be enabled by setting the config:

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
Set the spark automatically clean checkpoints to release disk memory:
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
This will not clean the latest checkpoint as it is still referred to by the application to recover from possible failures. If you expect to restart the application driver if it crashed due to some errors and exactly start from where it crashed last time instead of performing the whole operation once again, you can create **StreamingContext** from the previous checkpoint by using the method `getOrCreate()`:
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
However after I tried switching to `getOrCreate()`, I had the following exception:
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
It is because this is not the first time I use checkpoints, and there already exsists some other checkpoints in the same root directory (etc, the simple checkpoint discussed previously). Since the application tried to read from those unrelated checkpoints and expected to cast them to generate new context, it would throw such **ClassCastException**. This can be easily solved by deleting original checkpoints in HDFS locals (manually).

WAL is an advanced checkpoint mechanism and also simple to be applied. Compared with checkpoint, it saves data into an external filesystem so that even though if the executor is terminated and the data in memory is clean-uped, data still can be recovered from the external filesystem. However. it can only recover the data which is logged in the filesystem, if the drivers fail due to some error, the executor will be terminated as well, so as the WAL writer. Then the rest incomming data will not be logged into the filesystem and hence, is not recoverable. It can be tested by calling **stop** to the **StreamContext**:
```scala
def stop(stopSparkContext: Boolean, stopGracefully: Boolean): Unit
```
Once it is called, the following console log interpretes that the **WAL Writer** is interrupted as well:
```
ERROR ReceiverTracker: Deregistered receiver for stream 0: Stopped by driver
WARN BlockGenerator: Cannot stop BlockGenerator as its not in the Active state [state = StoppedAll]     
WARN BatchedWriteAheadLog: BatchedWriteAheadLog Writer queue interrupted.
```
And also the data will not be recoverable across applications or Spark upgrades and hence not very reliable
</br>

## Kafka Direct API
This mechanism is only available when you data source is **Kafka**. Kafka supports a commit strategy which is able to help you manage offsets of each topics. Each offset points to a slot in a topic. When the data stored in this slot is consumed by any receiver, Kafka will be acknowledged by this consumption and moves the index to the next data slot.
![https://blog.cloudera.com/blog/2017/06/offset-management-for-apache-kafka-with-apache-spark-streaming/](Spark-Streaming-flow-for-offsets.png)
(diagram from https://blog.cloudera.com/blog/2017/06/offset-management-for-apache-kafka-with-apache-spark-streaming/)
When `enable.auto.commit` is set to be true, as soon as any receiver retrieves the data from the offset datastore in Kafka (here I use Kafka to store offsets), the receiver will automatically commit, which doesn't ensure that the data is successfully executed in the spark streaming. Therefore, we have to disbale the auto-commit when we are creating DStream from Kafka. After the data is successfully processed, we manually commit the offset to the datastore by calling the Kafka direct API:
```scala
stream.foreachRDD { rdd =>
      //get current offset
      val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

      //process data

      //store data in some datastore
      DataStoreDB.push(...param)

      //commit offset to kafka
      stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
    }
  }
```
If an error occurs at any point during the execution, the offset won;t be committed to the kafka offset store and hence the same data will be resent by kafka, which ensures that the data will be executed only once.
