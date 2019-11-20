# Jprofiler

Download the agent

```
https://download-gcdn.ej-technologies.com/jprofiler/jprofiler_linux_11_0_2.tar.gz
tar -zxf jprofiler_linux_11_0_2.tar.gz
```

# Docker

Run kafka 

```
docker-compose up
docker-compose ps -d
```

Verify all is good

```
seq 10 | docker-compose exec kafka-client kafka-console-producer --broker-list kafka-1:9092 --topic dummy
docker-compose exec kafka-client kafka-console-consumer --bootstrap-server kafka-1:9092 --topic dummy --from-beginning
```

# Setup interesting topics

```
docker-compose exec kafka-client kafka-topics --zookeeper zookeeper:2181 --list
docker-compose exec kafka-client kafka-topics --zookeeper zookeeper:2181 --create --topic one-partition --replication-factor 1 --partitions 1
docker-compose exec kafka-client kafka-topics --zookeeper zookeeper:2181 --create --topic ten-partitions --replication-factor 1 --partitions 10
docker-compose exec kafka-client kafka-topics --zookeeper zookeeper:2181 --create --topic demo --replication-factor 3 --partitions 10 --config min.insync.replicas=2
```

# Bench

```
for compressionType in none snappy gzip lz4 zstd; do
    echo Benching with compression.type=$compressionType
    docker-compose exec kafka-client kafka-topics \
        --create \
        --zookeeper zookeeper:2181 \
        --replication-factor 1 \
        --partitions 1 \
        --topic benchmark-compression-${compressionType}

    docker-compose exec kafka-client kafka-producer-perf-test \
        --topic benchmark-compression-${compressionType} \
        --num-records 15000000 \
        --record-size 100 \
        --throughput 15000000 \
        --producer-props \
            acks=1 \
            linger.ms=100 \
            batch.size=2000000 \
            bootstrap.servers=kafka-1:9092,kafka-2:9092,kafka-3:9092 \
            compression.type=${compressionType}
done
```

# Latency

```
docker-compose exec kafka-client kafka-run-class kafka.tools.EndToEndLatency kafka-1:9092 demo 5000 all 100
docker-compose exec kafka-client kafka-run-class kafka.tools.EndToEndLatency kafka-1:9092 one-partition 5000 all 100
```

# Profiling

````

docker-compose exec kafka-client bash -c \
    "export KAFKA_OPTS=-agentpath:/tmp/jprofiler11.0.2/bin/linux-x64/libjprofilerti.so=port=20000 && \    
    kafka-producer-perf-test \
        --topic benchmark-compression-snappy \
        --num-records 15000000 \
        --record-size 100 \
        --throughput 15000000 \
        --producer-props \
            acks=1 \
            linger.ms=100 \
            batch.size=2000000 \
            bootstrap.servers=kafka-1:9092,kafka-2:9092,kafka-3:9092 \
            compression.type=snappy"
```

Look at the accumulator in the Kafka Producer
Look at the Sender
Look at the thread history

We accumulate, and batch/send
