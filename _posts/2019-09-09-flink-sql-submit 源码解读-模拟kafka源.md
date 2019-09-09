---
layout:     post
title:      apache flink
subtitle:   flink 1.9:Apache flink 使用 SQL 读取 Kafka 并写入 MySQL（源码解读-模拟kafka源）
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
>* [flink-sql-submit 源码解读-模拟kafka源](https://lianxiaobao.github.io/2019/09/09/flink-sql-submit-%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB-%E6%A8%A1%E6%8B%9Fkafka%E6%BA%90/)



# flink-sql-submit 源码解读

## 启动停止kafka
### `env.sh`
* 记录kafka、flink的安装目录

```bash
FLINK_DIR=/Users/lianxiaobao/lianxiaobao/flink/flink-1.9.0

KAFKA_DIR=/Users/lianxiaobao/lianxiaobao/kafka/kafka_2.11-2.2.0
```

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
* 启动 kafka 集群

```bash
# $(dirname "$0") 当前项目的相对路径
source "$(dirname "$0")"/kafka-common.sh

# prepare Kafka
echo "Preparing Kafka..."

start_kafka_cluster
```

### `stop-kafka.sh`
* 停止 kafka 集群

```bash
source "$(dirname "$0")"/kafka-common.sh

# prepare Kafka
echo "Stop Kafka..."

rm -rf /tmp/zookeeper
rm -rf /tmp/kafka-logs
stop_kafka_cluster
```

### `source-generator.sh`
* 开始模拟 kafka 源

```bash
source "$(dirname "$0")"/kafka-common.sh

# prepare Kafka
echo "Generating sources..."

create_kafka_topic 1 1 user_behavior
# 将输出结果 管道的形式输入到kafka topic中
java -cp target/flink-sql-submit.jar com.github.wuchong.sqlsubmit.SourceGenerator 1000 | $KAFKA_DIR/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic user_behavior
```

### 模拟kafka生产数据 `SourceGenerator.java`

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;

/*source-generator.sh 脚本会自动读取 user_behavior.log 的数据并以默认每毫秒1条的速率灌到 Kafka 的 user_behavior topic 中
 *  ···
 *source "$(dirname "$0")"/kafka-common.sh
 * # prepare Kafka
 * echo "Generating sources..."
 * create_kafka_topic 1 1 user_behavior
 * # 将输出结果 管道的形式输入到kafka topic中
 * java -cp target/flink-sql-submit.jar com.github.wuchong.sqlsubmit.SourceGenerator 1000 | $KAFKA_DIR/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic user_behavior
 * ···
 *
 * user_behavior.log
 * {"user_id": "543462", "item_id":"1715", "category_id": "1464116", "behavior": "pv", "ts": "2017-11-26T01:00:00Z"}
 * {"user_id": "662867", "item_id":"2244074", "category_id": "1575622", "behavior": "pv", "ts": "2017-11-26T01:00:00Z"}
 * ...
 *
 */
public class SourceGenerator {

    private static final long SPEED = 1000; // 每秒1000条

    public static void main(String[] args) {
        long speed = SPEED;
        if (args.length > 0) {
            speed = Long.valueOf(args[0]);
        }
        long delay = 1000_000 / speed; // 每条耗时多少微秒

        try (InputStream inputStream = SourceGenerator.class.getClassLoader().getResourceAsStream("user_behavior.log")) {
            BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
            long start = System.nanoTime();
            while (reader.ready()) {
                String line = reader.readLine();
                System.out.println(line);

                long end = System.nanoTime();
                long diff = end - start;
                //当时间小于1ms sleep 1ms
                while (diff < (delay*1000)) {
                    Thread.sleep(1);
                    end = System.nanoTime();
                    diff = end - start;
                }
                start = end;
            }
            reader.close();
        } catch (IOException e) {
            throw new RuntimeException(e);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```