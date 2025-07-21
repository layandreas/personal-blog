+++
author = "Andreas Lay"
title = "Real-time Data Streaming with Kafka Connect"
date = "2025-07-21"
description = "Using Kafka Connect to Stream Data to S3"
tags = ["kafka", "kafka-connect", "docker", "s3"]
categories = ["Kafka", "Data Engineering", "Streaming"]
ShowToc = true
TocOpen = true
+++

## Why Kafka Connect?

While you can always write your own Kafka connector to write data from Kafka to S3 or a database using for example [confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python), this might be hard to maintain and error prone. [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) can help you to simplify this task.

In this post we will ...

- ... set up a local Kafka cluster, S3 storage & Kafka Connect with Docker Compose
- ... create a Kafka topic and publish messages to it
- ... use Kafka Connect to create an S3 sink connector and write the messages to S3

## Run the Example

You can run the [full example](https://gist.github.com/layandreas/ac955778ad97b55112301e5efaf07e60) which is described in this post by executing the following script: It will download the necessary files, start the containers, create a Kafka topic, publish messages, and create an S3 sink connector to write the data to S3:

```zsh
#!/usr/bin/env zsh

curl https://gist.githubusercontent.com/layandreas/ac955778ad97b55112301e5efaf07e60/raw/5d065594619bd998c7ebb39ce1c91518ef25272c/kafka-connect-self-contained-example.sh > kafka-connect-self-contained-example.sh

zsh kafka-connect-self-contained-example.sh
```

You can clean up the containers afterwards by running:

```zsh
#!/usr/bin/env zsh

cd kafka-connect-example
docker compose down -v
```

## Setup

### Prerequisite: Download S3 Sink Connector

First we need to [download the Confluent Kafka Connect S3 Sink Connector](https://www.confluent.io/hub/confluentinc/kafka-connect-s3):

```zsh
#!/usr/bin/env zsh

mkdir kafka-connect-example
cd kafka-connect-example
mkdir kafka-connect-plugins

curl -O https://hub-downloads.confluent.io/api/plugins/confluentinc/kafka-connect-s3/versions/10.6.7/confluentinc-kafka-connect-s3-10.6.7.zip

unzip confluentinc-kafka-connect-s3-10.6.7.zip -d ./kafka-connect-plugins/confluentinc-kafka-connect-jdbc;
```

This will be mounted into the Kafka Connect container so that it can use the S3 Sink Connector.

### Start Containers & Create S3 Bucket

Let's use our [docker-compose.yml](https://gist.github.com/layandreas/ffb57ec5f102ed0d285c1e8e40838e1a) to start the containers:

```zsh
curl https://gist.githubusercontent.com/layandreas/ffb57ec5f102ed0d285c1e8e40838e1a/raw/be4dd51730c931085c58f9c00c761840d188326b/docker-compose.yml > docker-compose.yml

docker compose up
```

In a separate terminal create the S3 bucket to which we will write our data:

```zsh
docker exec minio mc alias set local http://localhost:9000 minioadmin minioadmin && docker exec minio mc mb local/my-bucket --ignore-existing
```

### Kafka: Create Topic & Publish Messages

After our containers have started we first need to create a Kafka topic:

```zsh
#!/usr/bin/env zsh

# Create Kafka topic. This is were we will send our messages to
docker exec kafka kafka-topics --create \
  --topic kafka_message \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1
```

Now we're ready to publish some messages:

```zsh
#!/usr/bin/env zsh

# Publish messages to Kafka topic
for i in {1..10}; do
  echo "Sending message $i..."
  docker exec -i kafka kafka-console-producer --broker-list kafka:9092 --topic my-topic <<EOF
{"id":$i,"name":"Alice","email":"alice@example.com"}
EOF
done

echo "✅ Done sending 10 messages."
```

You can go to [http://localhost:8080/ui/clusters/local/all-topics/my-topic/messages](http://localhost:8080/ui/clusters/local/all-topics/my-topic/messages) to open the [provectus Kafka UI](https://github.com/provectus/kafka-ui). You should see the messages we ave just published:

![Minio Bucket](/personal-blog/kafka-topic.png)

### Kafka Connect: Create S3 Sink Connector

Next we will create an [S3 sink connector](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html):

```zsh
#!/usr/bin/env zsh

curl -X POST -H "Content-Type: application/json" --data '{
  "name": "s3-sink",
  "config": {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max": "1",
    "topics": "my-topic",
    "s3.bucket.name": "my-bucket",
    "s3.region": "us-east-1",
    "store.url": "http://minio:9000",
    "flush.size": "3",
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.json.JsonFormat",
    "schema.compatibility": "NONE",
    "aws.access.key.id": "minioadmin",
    "aws.secret.access.key": "minioadmin",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "key.converter.schemas.enable": "false"
  }
}' http://localhost:8083/connectors
```

The connector will read messages from the `my-topic` topic and write them to the minio S3 bucket `my-bucket` in JSON format. The connector will flush data to S3 every 3 messages.

## Check the Results

You can go to the minio web interface at [http://localhost:9001/browser/my-bucket](http://localhost:9001/browser/my-bucket) and log in with `minioadmin` as both username and password. You should see the bucket `my-bucket` with the files loaded into it by Kafka Connect:

![Minio Bucket](/personal-blog/kafka-minio.png)

Each file will contain 3 messages in [JSON Lines format](https://jsonlines.org/examples/):

```json
{"name":"Alice","id":70,"email":"alice@example.com"}
{"name":"Alice","id":71,"email":"alice@example.com"}
{"name":"Alice","id":72,"email":"alice@example.com"}
```

## Full Docker Compose File

**docker-compose.yml**:

```yaml
version: "3.8"

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper

  minio:
    image: minio/minio
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio-data:/data

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.4.0
    container_name: kafka-connect
    ports:
      - "8083:8083"
    environment:
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: "connect-cluster"
      CONNECT_CONFIG_STORAGE_TOPIC: "connect-configs"
      CONNECT_OFFSET_STORAGE_TOPIC: "connect-offsets"
      CONNECT_STATUS_STORAGE_TOPIC: "connect-status"
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_INTERNAL_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_INTERNAL_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_PLUGIN_PATH: /usr/share/java,/etc/kafka-connect/jars
      CONNECT_LOG4J_LOGGERS: org.apache.kafka.connect.runtime.rest=WARN,org.reflections=ERROR
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      # S3 / MinIO access config (can be passed to the connector config)
      AWS_ACCESS_KEY_ID: minioadmin
      AWS_SECRET_ACCESS_KEY: minioadmin
      AWS_ENDPOINT_OVERRIDE: http://minio:9000
      # This will help the S3 connector connect to MinIO properly
    volumes:
      - ./kafka-connect-plugins:/etc/kafka-connect/jars
    depends_on:
      - kafka
      - minio

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      - KAFKA_CLUSTERS_0_NAME=local
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092
      - KAFKA_CLUSTERS_0_ZOOKEEPER=zookeeper:2181
    depends_on:
      - kafka
      - zookeeper

volumes:
  minio-data:
```
