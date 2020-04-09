# Flume and kafka

## Data Streaming from Log file into HDFS with flume and Kafka


## Table of Contents


## Create Kafka Topic

```
$KAFKA_HOME/bin/kafka-topics.sh --zookeeper xxx.xxx.xx.xx:2181 -list
$KAFKA_HOME/bin/kafka-topics.sh  --create --zookeeper xxx.xxx.xx.xx:2181  --replication-factor 1 --partitions 1 --topic EDB-Sales-log

#Producer
$KAFKA_HOME/bin/kafka-console-producer.sh --broker-list 129.154.72.81:9093  --topic EDB-Sales-log

```

## Flume

This configuration flume is to write from kafka source into hdfs

```
# EDB-Sales-Log-Kafka.conf
target_agent.sources = kafkaSource  
target_agent.channels = memoryChannel  
target_agent.sinks = hdfsSink 

target_agent.sources.kafkaSource.type = org.apache.flume.source.kafka.KafkaSource  
target_agent.sources.kafkaSource.zookeeperConnect = 129.154.72.52:2181  
target_agent.sources.kafkaSource.topic =  EDB-Sales-Avro
#target_agent.sources.kafkaSource.batchSize = 500000
#target_agent.sources.kafkaSource.batchDurationMillis = 200  
target_agent.sources.kafkaSource.channels = memoryChannel

# http://flume.apache.org/FlumeUserGuide.html#memory-channel  
target_agent.channels.memoryChannel.type = memory  
target_agent.channels.memoryChannel.capacity = 200000 
target_agent.channels.memoryChannel.transactionCapacity = 100000
target_agent.channels.memoryChannel.keep-alive = 60

## Write to HDFS  
target_agent.sinks.hdfsSink.type = hdfs  
target_agent.sinks.hdfsSink.channel = memoryChannel  
target_agent.sinks.hdfsSink.hdfs.path = /dobelbonus/EDB-Sales-Avro-log
target_agent.sinks.hdfsSink.hdfs.fileType = DataStream  
target_agent.sinks.hdfsSink.hdfs.fileSuffix = .avro 
target_agent.sinks.hdfsSink.hdfs.writeFormat=Text 
# Roll files in HDFS every 5 min(roll interval) or at 255MB; roll based on number of records
# We roll at 255MB because our block size is 128MB, we want 2 full blocks without going over
target_agent.sinks.hdfsSink.hdfs.rollSize = 267386880 
target_agent.sinks.hdfsSink.hdfs.rollCount = 0
target_agent.sinks.hdfsSink.hdfs.rollInterval = 300 
target_agent.sinks.hdfsSink.hdfs.batchSize = 500
#target_agent.sinks.hdfsSink.hdfs.callTimeout = 60000
target_agent.sinks.hdfsSink.hdfs.round =true
target_agent.sinks.hdfsSink.hdfs.roundValue = 10
target_agent.sinks.hdfsSink.hdfs.roundUnit = minute
target_agent.sinks.hdfsSink.hdfs.threadsPoolSize = 10
target_agent.sinks.hdfsSink.hdfs.rollTimerPoolSize = 10

```
```
==========================================
#Flume Producer - Produce data to Kafka
==========================================
$FLUME_HOME/bin/flume-ng agent --conf-file $FLUME_HOME/conf/EDB-Sales-Log-Kafka.conf  -Dflume.root.logger=DEBUG,console --name a1 -Xmx1024m -Xms1024m
```


## This config flume to write data from kafka to Hive
```
flumeagent1.sources = source_from_kafka
flumeagent1.channels = mem_channel
flumeagent1.sinks = hive_sink
# Define / Configure source
flumeagent1.sources.source_from_kafka.type = org.apache.flume.source.kafka.KafkaSource
flumeagent1.sources.source_from_kafka.zookeeperConnect = 129.154.72.52:2181
flumeagent1.sources.source_from_kafka.topic = SalesDBTransactions
flumeagent1.sources.source_from_kafka.groupID = flume
flumeagent1.sources.source_from_kafka.channels = mem_channel
flumeagent1.sources.source_from_kafka.interceptors = i1
flumeagent1.sources.source_from_kafka.interceptors.i1.type = timestamp
flumeagent1.sources.source_from_kafka.consumer.timeout.ms = 1000
  
# Hive Sink
flumeagent1.sinks.hive_sink.type = hive
flumeagent1.sinks.hive_sink.hive.metastore = thrift://129.154.72.81:9083
flumeagent1.sinks.hive_sink.hive.database = dobelbonus
flumeagent1.sinks.hive_sink.hive.table = customers
flumeagent1.sinks.hive_sink.useLocalTimeStamp=true
flumeagent1.sinks.hive_sink.hive.txnsPerBatchAsk = 2
flumeagent1.sinks.hive_sink.hive.partition = %y-%m-%d-%H-%M
flumeagent1.sinks.hive_sink.batchSize = 10
flumeagent1.sinks.hive_sink.rollInterval=600
flumeagent1.sinks.hive_sink.rollCount=10000
flumeagent1.sinks.hive_sink.rollSize=0
flumeagent1.sinks.hive_sink.serializer = DELIMITED
flumeagent1.sinks.hive_sink.serializer.delimiter = ,
flumeagent1.sinks.hive_sink.serializer.fieldnames = id,name,email,street_address,company
# Use a channel which buffers events in memory
flumeagent1.channels.mem_channel.type = memory
flumeagent1.channels.mem_channel.capacity = 100000
flumeagent1.channels.mem_channel.transactionCapacity = 100000 
# Bind the source and sink to the channel
flumeagent1.sources.source_from_kafka.channels = mem_channel
flumeagent1.sinks.hive_sink.channel = mem_channel

```

# Flume with regex
```
target_agent.sources = kafkaSource  
target_agent.channels = memoryChannel  
target_agent.sinks = hdfsSink 

target_agent.sources.kafkaSource.type = org.apache.flume.source.kafka.KafkaSource  
target_agent.sources.kafkaSource.zookeeperConnect = 129.154.72.52:2181  
target_agent.sources.kafkaSource.topic =  EDB-Sales-log 
target_agent.sources.kafkaSource.batchSize = 5  
target_agent.sources.kafkaSource.batchDurationMillis = 200  
target_agent.sources.kafkaSource.channels = memoryChannel

## Describe regex_filter interceptor and configure exclude events attribute
target_agent.sources.kafkaSource.interceptors = i1 i2 i3 i4 i5 i6 i7
target_agent.sources.kafkaSource.interceptors.i1.type = regex_filter
target_agent.sources.kafkaSource.interceptors.i1.regex = Worker
target_agent.sources.kafkaSource.interceptors.i1.excludeEvents = true

target_agent.sources.kafkaSource.interceptors.i2.type = regex_filter
target_agent.sources.kafkaSource.interceptors.i2.regex = Preparing 
target_agent.sources.kafkaSource.interceptors.i2.excludeEvents = true

target_agent.sources.kafkaSource.interceptors.i3.type = regex_filter
target_agent.sources.kafkaSource.interceptors.i3.regex = Wrote 
target_agent.sources.kafkaSource.interceptors.i3.excludeEvents = true

target_agent.sources.kafkaSource.interceptors.i4.type = regex_filter 
target_agent.sources.kafkaSource.interceptors.i4.regex = Done 
target_agent.sources.kafkaSource.interceptors.i4.excludeEvents = true

target_agent.sources.kafkaSource.interceptors.i5.type = regex_filter
target_agent.sources.kafkaSource.interceptors.i5.regex = Nothing
target_agent.sources.kafkaSource.interceptors.i5.excludeEvents = true

# [x] Received '8561|880031988955|628|1|570|201802142002|YOMARTHQ|5|5'
# ^.*Received \B(')|\b(')
# ^.*Receivied ==> find String 
# \B(')==> find first and \b(')==find lastt '
# https://regex101.com/r/xA9kG3/1

target_agent.sources.kafkaSource.interceptors.i6.type = search_replace
#target_agent.sources.kafkaSource.interceptors.i6.searchPattern = ^.*Received \B(')|\b(') 
target_agent.sources.kafkaSource.interceptors.i6.searchPattern = ^.*Received \\B(')|\\b(')
target_agent.sources.kafkaSource.interceptors.i6.replaceString= 

# Matches any space, tab or newline character.
target_agent.sources.kafkaSource.interceptors.i7.type = regex_filter
target_agent.sources.kafkaSource.interceptors.i7.regex = ^\\s
target_agent.sources.kafkaSource.interceptors.i7.excludeEvents = true

# http://flume.apache.org/FlumeUserGuide.html#memory-channel  
target_agent.channels.memoryChannel.type = memory  
target_agent.channels.memoryChannel.capacity = 1000
target_agent.channels.memoryChannel.keep-alive = 60

## Write to HDFS  
#http://flume.apache.org/FlumeUserGuide.html#hdfs-sink  
target_agent.sinks.hdfsSink.type = hdfs  
target_agent.sinks.hdfsSink.channel = memoryChannel  
target_agent.sinks.hdfsSink.hdfs.path = /dobelbonus/EDB-Sales-log
target_agent.sinks.hdfsSink.hdfs.fileType = DataStream  
target_agent.sinks.hdfsSink.hdfs.writeFormat = Text  
target_agent.sinks.hdfsSink.hdfs.rollSize = 0  
target_agent.sinks.hdfsSink.hdfs.rollCount = 10000  
target_agent.sinks.hdfsSink.hdfs.rollInterval = 600  


```
