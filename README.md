# Kafka Cheat sheet

## Create or Update Topics

Topics are logical containers that will contain logs coming from the same business use case or sharing a same format. You may have a topic to centralize access logs from your Apache servers and another one to store your application logs.

### Create a Topic

The most important details when creating a topic are its name (obviously), the number of partitions and the replication factor.

The number of partitions will determine how much your application may hope to parallelize reads and writes to a topic. If you found out that your application can read up to "n" messages per second from a topic with a single partition, you could reasonably assume that this number could be doubled by adding a second partition. Think of a partition as a FIFO (First-In, First-Out) meaning that message ordering is **NOT** guaranteed unless the topic is made of a single partition.

The replication factor will determine the number of replicas to create for a topic. "1" is the minimal allow value for the replication factor, it means that you do not want replication (a bit misleading right?). By setting a replication factor greater than one, Kafka will replicate all your topic partitions to the other nodes.

```sh
kafka-topics \
--zookeeper zookeeper:2181 \
--create \
--topic a_topic \
--partitions 1 \
--replication-factor 1 \
--if-not-exists
```

Output:

```sh
Created topic "a_topic".
```

### List the Topics

```sh
kafka-topics \
--zookeeper zookeeper:2181 \
--list
```

Output:

```sh
a_topic
```

### Describe the Topics

Describing topics give extra informations about your cluster state. You will find the overall configuration of the topic and state informations for each partition of the topic.

When a topic is first created, Kafka automatically assign a leader for every partition part of the topic. A partition leader is a broker assigned with the task of receiving all produce and fetch requests for a partition. In case of a single node cluster all partitions of a topic will share the same leader. If a cluster was made of 3 nodes when a 3 partitions topic was first created, each partition will eventually have a different leader id.

The replicas is a comma separated list of brokers ids that contain a replicated version of a topic partition. The replicas list is statically determined at topic creation. An interesting property of this list is that the first broker id is the **preferred partition leader**. It means that Kafka will do its best to ensure that the current partition leader will always be this one.

Finally the ISR is the list of in-sync replicas, i.e. the brokers that managed to keep their topic partition replica up-to-date with the leader topic partition.

Ideally the current leader for a topic partition should match the first id of the replicas list and the ISR list should be equal to the replicas list.

```sh
kafka-topics \
--zookeeper zookeeper:2181 \
--describe
```

Output:

```sh
Topic:a_topic  PartitionCount:1        ReplicationFactor:1     Configs:
        Topic: a_topic Partition: 0    Leader: 1       Replicas: 1     Isr: 1
```

### Alter a Topic

```sh
kafka-topics \
--zookeeper zookeeper:2181 \
--alter \
--topic a_topic \
--partitions 2
```

Output:

```sh
WARNING: If partitions are increased for a topic that has a key, the partition logic or ordering of the messages will be affected
Adding partitions succeeded!
```

### Delete a Topic

It is possible to delete a Kafka topic iff:

- no more producers on this topic
- no more consumers on this topic
- brokers are configured to allow topic deletion (`delete.topic.enable=true`)

```sh
kafka-topics \
--zookeeper zookeeper:2181 \
--delete \
--topic a_topic
```

Output:

```sh
Topic a_topic is marked for deletion.
```

## Produce from or Consume to a Topic

### Produce to a Topic from the Console

```sh
kafka-console-producer \
--broker-list localhost:9092 \
--topic a_topic
```

Output:

```sh
> Hello World!
> ^C
```

### Consume a Topic from the First Offset

```sh
kafka-console-consumer \
--bootstrap-server kafka:9092 \
--topic a_topic \
--from-beginning
```

Output:

```sh
Hello World!
^CProcessed a total of 1 messages
```

### Consume 1 Message from a Topic

```sh
kafka-console-consumer \
--bootstrap-server kafka:9092 \
--topic a_topic \
--from-beginning \
--max-messages 1
```

Output:

```sh
Hello World!
```

### Consume a Topic with a Consumer Group

Usually topics are consumed using a consumer group but it is not mandatory. Consumer groups allow to group a pack of processes as a single application and split reads and writes to a topic among theme. If you wish to start multiple threads or deploying multiple processes to consume a topic in parallelized fashion, you want to use a consumer group.

By using a consumer group Kafka will be able to track down the last message you have consumed for every partition of a topic. An offset is the index of a message in a topic partition. Offsets are auto incremented and used to know what messages have not be read yet by a consumer. Messages which offsets have already been committed by a consumer will not be read again even if the consumer reboots.

Kafka store the offsets in the `__consumer_offsets` topic.

```sh
kafka-console-consumer \
--bootstrap-server kafka:9092 \
--topic a_topic \
--group a_group \
--from-beginning
```

Output:

```sh
Hello World!
^CProcessed a total of 1 messages
```

## Manage the Consumer Groups

### Describe a Consumer Group

Consumer groups can be described to get useful informations on how fast a consumer group reads messages from a topic.

The current offset information is pretty simple, it represents the last committed offset for a topic partition for the consumer group. The log end offset is the last produced message offset for a partition. The lag statistic is very interesting, it is calculated by subtracting the current offset to the log end offset:

```sh
For a partition k,
Lag(k) = EndOffset(k) - CurrentOffset(k)
```

The lag indicates the number of messages, for a given partition, a consumer has yet to consume to be up-to-date at a single point in time. Ideally you'd want your application to have a lag around 0.

```sh
kafka-consumer-groups \
--bootstrap-server kafka:9092 \
--describe \
--group a_group
```

Output:

```sh
GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                             HOST            CLIENT-ID
a_group         a_topic         0          1               1               0               consumer-a_group-1-c40d5ee8-5d05-4e29-9245-a527a5144ba9 /172.18.0.3     consumer-a_group-1
```

### Reset our Current Offset to Earliest

```sh
kafka-consumer-groups \
--bootstrap-server kafka:9092 \
--reset-offsets \
--topic a_topic \
--group a_group \
--execute \
--to-earliest
```

Output:

```sh
TOPIC                          PARTITION  NEW-OFFSET
a_topic                        0          0
```

### Reset our Current Offset to a Timestamp

```sh
kafka-consumer-groups \
--bootstrap-server kafka:9092 \
--reset-offsets \
--topic a_topic \
--group a_group \
--execute \
--to-datetime 2020-01-01T00:00:00.000
```

Output:

```sh
TOPIC                          PARTITION  NEW-OFFSET
a_topic                        0          0
```

## Browse Zookeeper Content

Zookeeper is the brain of a Kafka cluster, it provides primitives to manage synchronization operations such as controller election as well as a configuration storage for topic definitions or security policies.

### Connect to a Zookeeper Node

```sh
zookeeper-shell localhost:2181
```

Output:

```sh
Connecting to localhost:2181
Welcome to ZooKeeper!
JLine support is enabled

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: localhost:2181(CONNECTED) 0]
```

### List the Z-nodes

```sh
ls /
```

Output:

```sh
[admin, brokers, cluster, config, consumers, controller, controller_epoch, isr_change_notification, latest_producer_id_block, log_dir_event_notification, zookeeper]
```

### Read a Z-Node Content

```sh
get /brokers/ids/1
```

Output:

```sh
{"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT","PLAINTEXT_HOST":"PLAINTEXT"},"endpoints":["PLAINTEXT://kafka:9092","PLAINTEXT_HOST://localhost:29092"],"jmx_port":-1,"host":"kafka","timestamp":"1581430101909","port":9092,"version":4}
```

### Close the Zookeeper Console

```sh
quit
```

Output:

```sh
WATCHER::

WatchedEvent state:Closed type:None path:null
```

## Debug a Kakfa Cluster

### Find the Current Controller

The controller is the broker in charge of electing partition leaders in addition to the usual work of a Kafka broker. A cluster should have one and only one controller at all times but it may not always be the same.

Sometimes you may need to find out which node is the current controller. This piece of information is stored in an ephemeral z-node that will self-destruct if the controller does not regularly send heartbeats to the Zookeeper cluster.

```sh
get /controller
```

Output:

```sh
{"version":1,"brokerid":1,"timestamp":"1581430102118"}
```

### List the Registered Brokers IDs

Another very important piece of information you wish to easily retrieve is the list of brokers registered under Zookeeper. At startup each broker register itself to its configured Zookeeper cluster, Zookeeper is the man providing all Kafka needs to form a resilient cluster.

Brokers ids are also stored in ephemerals z-nodes:

```sh
ls /brokers/ids
```

Output:

```sh
[1]
```

### Read a Segment Content

Kafka topics are split into partitions, a partition acts like a FIFO on its own and allow to parallelize reads and writes to achieve high message throughput.

A Kafka topic is persisted on disk under the location set by the `kafka.logs.dir` parameter. Therefore if you navigate to this directory, you should see one or several subfolder(s) named after the topic name and appended with the partition numbers:

```sh
$ ls -l /var/lib/kafka/data
> ...
> a_topic-0
> a_topic-1
```

Over time a partition can grow quite large, any operation such as logs compaction or deletion can become expensive. That is why partitions are split into smaller chunks of data called segments. Depending on the broker configuration, segments that are too big will be purged or compacted to create more manageable segments. A message produced to a topic ends up in a segment somewhere in the cluster:

```sh
$ ls -l /var/lib/kafka/data/a_topic-0
> -rw-r--r-- 1 root root 10485760 Feb 11 14:10 00000000000000000000.index
> -rw-r--r-- 1 root root       80 Feb 11 14:11 00000000000000000000.log
> -rw-r--r-- 1 root root 10485756 Feb 11 14:10 00000000000000000000.timeindex
> -rw-r--r-- 1 root root        8 Feb 11 14:10 leader-epoch-checkpoint
```

Sometimes you may need dump the content of a segment, remember that a single segment may contain gigabytes of data:

```sh
kafka-run-class kafka.tools.DumpLogSegments \
--print-data-log \
--files 00000000000000000000.log
```

Expected:

```sh
Dumping 00000000000000000000.log
Starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 0 CreateTime: 1581430298436 size: 80 magic: 2 compresscodec: NONE crc: 208563306 isvalid: true
| offset: 0 CreateTime: 1581430298436 keysize: -1 valuesize: 12 sequence: -1 headerKeys: [] payload: Hello World!
```
