---
layout:     post
title:      apache flink
subtitle:   flink 1.9:Apache flink 使用 SQL 读取 Kafka 并写入 MySQL（源码解读）
date:       2019-09-09
author:     BY xiaobao(微信：Bao1697047283)
header-img: img/post-bg-mi1.jpg
catalog: true
tags:
    - blog
    - 数据开发
    - kafka
    - flink
    
---


#### 目录
>* [使用 SQL 读取 Kafka 并写入 MySQL](https://lianxiaobao.github.io/2019/09/06/Apache-flink%E4%BD%BF%E7%94%A8SQL%E8%AF%BB%E5%8F%96Kafka%E5%B9%B6%E5%86%99%E5%85%A5MySQL/)
>* [flink-sql-submit 源码解读](https://lianxiaobao.github.io/2019/09/06/Apache-flink%E4%BD%BF%E7%94%A8SQL%E8%AF%BB%E5%8F%96Kafka%E5%B9%B6%E5%86%99%E5%85%A5MySQL/)



# flink-sql-submit 源码解读

## 启动停止kafka
### `kafka-common.sh`
* 该工具包很好的做到了启动、停止`kafka`启群以及`topic`的创建、删除可以作为日后的工具包使用

```bash
# $(dirname "$0") 当前项目的相对路径
source "$(dirname "$0")"/env.sh

function create_kafka_json_source {
    topicName="$1"
    create_kafka_topic 1 1 $topicName

    # put JSON data into Kafka
    echo "Sending messages to Kafka..."

    send_messages_to_kafka '{"rowtime": "2018-03-12T08:00:00Z", "user_name": "Alice", "event": { "message_type": "WARNING", "message": "This is a warning."}}' $topicName
    send_messages_to_kafka '{"rowtime": "2018-03-12T08:10:00Z", "user_name": "Alice", "event": { "message_type": "WARNING", "message": "This is a warning."}}' $topicName
    send_messages_to_kafka '{"rowtime": "2018-03-12T09:00:00Z", "user_name": "Bob", "event": { "message_type": "WARNING", "message": "This is another warning."}}' $topicName
    send_messages_to_kafka '{"rowtime": "2018-03-12T09:10:00Z", "user_name": "Alice", "event": { "message_type": "INFO", "message": "This is a info."}}' $topicName
    send_messages_to_kafka '{"rowtime": "2018-03-12T09:20:00Z", "user_name": "Steve", "event": { "message_type": "INFO", "message": "This is another info."}}' $topicName
    send_messages_to_kafka '{"rowtime": "2018-03-12T09:30:00Z", "user_name": "Steve", "event": { "message_type": "INFO", "message": "This is another info."}}' $topicName
    send_messages_to_kafka '{"rowtime": "2018-03-12T09:30:00Z", "user_name": null, "event": { "message_type": "WARNING", "message": "This is a bad message because the user is missing."}}' $topicName
    send_messages_to_kafka '{"rowtime": "2018-03-12T10:40:00Z", "user_name": "Bob", "event": { "message_type": "ERROR", "message": "This is an error."}}' $topicName
}

function create_kafka_topic {
    $KAFKA_DIR/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor $1 --partitions $2 --topic $3
}

function drop_kafka_topic {
    $KAFKA_DIR/bin/kafka-topics.sh --delete --zookeeper localhost:2181 --topic $1
}

function send_messages_to_kafka {
    echo -e $1 | $KAFKA_DIR/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic $2
}

function start_kafka_cluster {
  if [[ -z $KAFKA_DIR ]]; then
    echo "Must run setup kafka dist dir before attempting to start Kafka cluster"
    exit 1
  fi

  $KAFKA_DIR/bin/zookeeper-server-start.sh -daemon $KAFKA_DIR/config/zookeeper.properties
  $KAFKA_DIR/bin/kafka-server-start.sh -daemon $KAFKA_DIR/config/server.properties

  # zookeeper outputs the "Node does not exist" bit to stderr
  while [[ $($KAFKA_DIR/bin/zookeeper-shell.sh localhost:2181 get /brokers/ids/0 2>&1) =~ .*Node\ does\ not\ exist.* ]]; do
    echo "Waiting for broker..."
    sleep 1
  done
}

function stop_kafka_cluster {
  $KAFKA_DIR/bin/kafka-server-stop.sh
  $KAFKA_DIR/bin/zookeeper-server-stop.sh

  # Terminate Kafka process if it still exists
  PIDS=$(jps -vl | grep -i 'kafka\.Kafka' | grep java | grep -v grep | awk '{print $1}'|| echo "")

  if [ ! -z "$PIDS" ]; then
    kill -s TERM $PIDS || true
  fi

  # Terminate QuorumPeerMain process if it still exists
  PIDS=$(jps -vl | grep java | grep -i QuorumPeerMain | grep -v grep | awk '{print $1}'|| echo "")

  if [ ! -z "$PIDS" ]; then
    kill -s TERM $PIDS || true
  fi
}
```



### `start-kafka.sh`

```bash
# $(dirname "$0") 当前项目的相对路径
source "$(dirname "$0")"/kafka-common.sh

# prepare Kafka
echo "Preparing Kafka..."

start_kafka_cluster
```

### `stop-kafka.sh`

```bash
source "$(dirname "$0")"/kafka-common.sh

# prepare Kafka
echo "Stop Kafka..."

rm -rf /tmp/zookeeper
rm -rf /tmp/kafka-logs
stop_kafka_cluster
```