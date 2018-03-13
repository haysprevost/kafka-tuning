# Kafka Topic Configuration and Tuning Guide
The information in this guide is a collection of "things you should probably know before going to prod" which have been gathered from Kafka's documentation, Confluent's documentation and blogs, training provided by Confluent, and input from their professional services team.  

Know your data pipeline. Since tuning is highly dependent on your use case, there is no silver bullet config that will serve everyones needs perfectly. The purpose of this guide is to help you make those choices and give some general suggestions based on common consumer/producer choices, and CAP preferences.
##### There are three levels of configuration to be concerned about when working with the Kafka log.

* Brokers(Topic Level)
* Producers
* Consumers

####  Topic Level Config ####
The Broker topic level configuration will take default values from the server, but each of the following properties can be overridden at the topic level.

* **Partition count** - Required
  + The partitioning of data in Kafka's context is the routing of a stream of logs based on a key. The key of the log gets hashed and mapped to a partition. This mapping ensures that a keyed message will always land on the same partition, but a single partition can store logs for many keys. Multiple producers can produce to the same partition, but a broker will only feed data back out to a single consumer thread per partition. This one to one mapping between partition and consumer ensures ordering of messages within the partition, but also becomes the bottleneck for parallelism.
  + Another very important fact about partitions is that they only be increased, but once a topic's partitions have been increased guarantees for ordering across a partition are lost. It is also important not to over partition, over partitioning can result in undue stress on the cluster via number of file handles, strain on intra-cluster replication, and slower overall end-to-end latency and recovery times for your topic. There is a max theoretical partition count that a cluster can handle based on its size before cluster-wide latency starts to become a problem.
  + A good goal is to try to partition for the maximum future throughput you might need. Final partition count selection might be based on the max throughput you can achieve at the producer or consumer level divided into your total estimated throughput. More on throughput tuning explained later, its dependent on many things including your data schema, compression choice, client choices, and consumer/producer code implementations, etc.
  + Partitions are the main tuning knob for throughput


* **Replication Factor** - Required - Defaults to 3
  + This does not not affect parallelism as one might assume, and is only used for durability.
  + The unit of replication is the topic partition
  + For a topic with replication factor N, we will tolerate up to N-1 server failures without losing any records committed to the log. Meaning with a replication factor of 3, up to two brokers could fail. replication factor of 5, up to 4 brokers could fail. This does not mean producers will continue to send data in this degraded state, see producer settings for more details.
  + The trade off for increasing replication factor is total cluster capacity reduction. That being said, we can't suggest broker size = replication factor, which would be the highest level of redundancy. The suggested replication factor would be 3 to start. If you need a better durability guarantee we should have accurate estimates on peak data retention sizes and throughput estimates as this can get out of control quickly. When it absolutely, positively, always has to be there. I could see 4 replicas being a reasonable choice.
  + [Confluent Blog on Intra-Broker Replication](https://www.confluent.io/blog/hands-free-kafka-replication-a-lesson-in-operational-simplicity/)    




* **Min In-sync Replicas**
 + Min ISR has to be met before the leader of a topic will commit that offset as the high water mark.
 + Whats a Commit? – The commit propagates the high water mark from the leader to all ISRs, which then allows consumers to read up to the high water mark from the topic leader.
 + When a failed replica is restarted, it first recovers the latest HWM from disk and truncates its log to the HWM. This is necessary since messages after the HWM are not guaranteed to be committed and may need to be thrown away. Then, the replica becomes a follower and starts fetching messages after the HWM from the leader. Once it has fully caught up, the replica is added back to the ISR and the system is back to the fully replicated mode
 + The default min.insync replicas is 1 which favors throughput and availability. This setting is good for topics which need low latency over consistency.
 + In a mission critical application, we expect a Kafka cluster with 3 brokers can satisfy two requirements:
   * When 1 broker is down, no message loss or service blocking happens.
   * In worse cases such as two brokers are down, service can be blocked, but no message loss happens.or a mission critical application
   * To achieve this you would need a min insync replica value to be 2. This means that as soon as the timeout for the 2nd to last replica triggers, an exception is thrown to the producer clients which have ACKS = ALL. Producers with acks =1 would still make it through to the last remain ISR, the leader.


* **Cleanup Policy**
  * Delete - the deletion policy means that we will either use time or size to manage your disk usage.
    * retention.ms and retention.bytes manage these values, whichever comes first would be honored by the broker.
  * Compact - Log compaction ensures that Kafka will always retain at least the last known value for each message key within the log of data for a single topic partition
    * min.cleanable.dirty.ratio
      * This ratio bounds the maximum space wasted in the log by duplicates. Once the log cleaner sees that you have hit the configured ratio, it will begin adding deletion tombstones to the offset of keys with old data. This value defaults to 0.5, which means up to 50% of the log can be uncompacted before the log compactor will evaluate it for cleaning.
* Compression Type
  * lz4, gzip, and snappy are your choices but it defaults to none. The compression is done by the producer by the batch.size, keep this in mind when tuning for throughput.  
* Handling large messages - reduce your message size
  * Use compression!
  * Split large messages into 1 KB segments with the producing client, using partition keys to ensure that all segments are sent to the same Kafka partition in the correct order. The consuming client can then reconstruct the original large message.
  * If shared storage (such as NAS, HDFS, or S3) is available, consider placing large files on the shared storage and using Kafka to send a message with the file location. In many cases, this can be much faster than using Kafka to send the large file itself.
  * Although Kafka is capable of increasing the 1MB message payload, it causes issues across the board and should be avoided at all costs.


* Tuning tip - Back Pressure
   * When a Kafka cluster starts suffering from high load / increased latency, broker failure, GC Pause, or other unfortunate circumstances its important to handle these scenarios gracefully. A producer with a very short lifespan or small internal buffer might prefer to send messages with weaker consistency guarantees vs not sending them at all. How long can your data producer queue up messages before dropping them? Which producer settings work for you? lets explore:

####  Producer Level Config ####
* Acks - Whats an Ack? – this is when a broker acknowledges to producer that a message has been written.  
0 – producer gets an ack when the message is written to the network buffer. Make sure you have set block.on.buffer.full=true if you use this to prevent the buffer from throwing away messages.  
1 – producer gets an ack after the leader has written message batch to the WAL before proceeding.  
All (-1) – The producer only gets an ack after all in-sync replicas have written the WAL, and the offset has been committed, data is available for reads by consumer.

|Acks   |Throughput|Latency|Durability   |
|-------|----------|-------|-------------|
|0      |High      |Low    |No guarantees|
|1      |Medium    |Medium |Leader       |
|-1(All)|Low       |High   |ISR         ||     

Retries- make sure this does not get exhausted or your producer will drop messages  

* Suggested settings for consistency ( prevent data loss )
  * block.on.buffer.full=true
  * retries=Long.MAX_VALUE
  * acks=all
  * resend in your callback when message send fails.

* Suggested settings to prevent re-ordering  
  * This might happen if max.in.flight.connections > 1 and retries are enabled OR if the producer is closed carelessly ( without using close(0))
  * close the producer w/ close(0) on callback w/ error
  * set max.in.flight.connections = 1 ( at the expense of throughput here)  

* End to End latency
  * Higher # of acks on the producer, partitions, or Replicas will increase this time as more nodes have to perform ops before the producer can continue with the next round of batches.


**Producer Tuning**
The goal of tuning the producer is to maximize throughput under your durability constraints.
 * Critical Configurations
   * batch.size - size based batching, this
   * linger.ms - time based batching
   * compression.type
   * max.in.flight.requets.per.connection
   * acks

* a batch will ship to the leader as soon as one of the following is true:
  * batch.size is reached
  * linger.ms is reached
  * another batch to the same broker is ready
  * flush() or close() is called

* on every send(), the message is batched, serialized, partitioned, and then compressed if enabled. multiple batches can go out to the same broker at once.

* Producer JMX Metrics to watch while tuning:
  * Select_Rate_Avg
  * Request_Rate_Avg
  * Request_Latency_Avg
  * Request_Size_Avg
  * Batch_Size_Avg
  * Records_Per_Request_Avg
  * Record_Queue_Time_Avg
  * Compression_Rate_Avg  


* You can then compute more meaningful data off of these metrics:
    * MB/Sec = (Request_Rate_Avg * Request_Size_Avg) / Compression_Rate_Avg
    * Request rate upper limit = 1000 / Request_Latency_Avg * Num brokers  
    * Throughput Avg = (Request_Rate_Avg * Request_Size_Avg) / Compression_Rate_Avg
* You can increase the request size w/ more user threads,partitions,or bigger batches, but be carefull because this can also increase compression time if compression is enabled.
* A small batch size will make batching less common and may reduce throughput (a batch size of zero will disable batching entirely). A very large batch size may use memory a bit more wastefully as we will always allocate a buffer of the specified batch size in anticipation of additional records. Eventually bigger batches increase the latency. You have to play with it and watch your throughput averages.
* Finding a throughput bottleneck:
  * Is the bottleneck in a user thread?
    * increase the number of threads while paying attention to lock contention
  * Is the bottlneck in the sender thread?
    * play with batch size
  * Is throughput hitting your network bandwidth?
    * If not, you still have room for improvement
  * Is the Record_Queue_Time_Avg large? Larger than linger ms?
    * if so, you might be hitting a user thread limit
  * Is Batch_Size_Avg almost equal to batch size?
    * you are getting close to a sweet spot, but maybe the linger.ms is firing before the batch is full.
  * With acks = all, if Replica fetch request remote time > 0, the replica fetch requests are waiting for the message to arrive.

#####  Consumer Level Config #####
If you've been using the kafka consumer up until now, you've likely been on the 'old' consumer which uses a zookeeper connection string. As of now the 'new' consumer from the 0.9 release is default. This consumer re-write uses a group coordination protocol built into Kafka itself. Make sure to use the bootstrap.server property and remove your zookeeper connection string.

 For each consumer group, one of the brokers is selected as the group coordinator. It controls partition assignment for new members, old members departure, and when topic metadata changes. The act of reassigning partitions is known as rebalancing the group. From the perspective of the consumer, the main thing to know is that you can only read up to the high watermark. This prevents the consumer from reading unreplicated data which could later be lost.

 * Always close the consumer when you are finished with it. Not only does this clean up any sockets in use, it ensures that the consumer can alert the coordinator about its departure from the group.

More tidbits:  
* Consumers always read from the leader
* turn off auto-commits - autocommit.enable = false  
* Manually commit offsets after the message data is processed or persisted.  
* fetch.message.max.bytes
  * The number of bytes of messages to attempt to fetch for each topic-partition in each fetch request. These bytes will be read into memory for each partition, so this helps control the memory used by the consumer.
* Theres a [Confluent Consumer Usage Blog](https://www.confluent.io/blog/tutorial-getting-started-with-the-new-apache-kafka-0-9-consumer-client/) that really gives a good overview of how to make the best of consumer client.




#####  Common Configuration Scenarios #####
* Exactly once consumption

### Sources

[Kafka Documentation](https://kafka.apache.org/documentation/)  
[Confluent Documentation](http://docs.confluent.io/)   
[Confluent Blog](https://www.confluent.io/blog/)  
[Linkedin Performance Tuning](http://www.slideshare.net/JiangjieQin/producer-performance-tuning-for-apache-kafka-63147600?next_slideshow=2)  