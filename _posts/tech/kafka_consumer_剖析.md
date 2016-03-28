## Kafka Consumer 剖析

### 微观剖析

#### 1. 主要的类机器作用

ConsumerConfig: 生产者的各种配置
ConsumerIterator: KafkaStream 迭代器。  
next 方法实现：

KafkaStream: 内部维护一个  ConsumerIterator 可以迭代 MessageAndMetadata。 提供两个方法 iterator 和 clear ， clear 主要是在consumer重分布时清除被迭代的队列，减少consumer接收到重复消息。  

ConsumerConnector： consumer的主接口。定义了一个trait和一个object。

1. createMessageStreams：为每个topic创建一组KafkaStream
2. createMessageStreams （支持指定KeyDeCoder和ValueDecoder）
3. createMessageStreamsByFilter：也是为给定的所有topic创建一组KafkaStream，只不过这个方法允许传递一个filter，允许黑白名单过滤
4. commitOffsets：向连接此consumer connector的所有broker分区执行提交位移操作
5. shutdown：关闭connector
而Consumer object定义了两个方法：
1. create：创建一个ConsumerConnector
2. createJavaConsumerConnector：创建一个java client使用的consumer connector

FetchedDataChunk：获取数据块。

PartitionAssignor：rebalance 类
AssignmentContext：rebalance 上下文信息，主要包含：  

1. partitionsForTopic：返回topic对应的分区集合
2. consumersForTopic：返回topic对应的consumers线程
3. consumers：返回consumers id的集合


TopicCount： 统计分析

ZookeeperConsumerConnector： 主要负责与 ZK 进行交互。
