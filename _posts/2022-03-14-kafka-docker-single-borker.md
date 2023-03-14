---
title: "Docker를 활용한 Single Broker Kafka 구성"
categories:
  - Kafka
tag:
  - Docker
  - Kafka
toc: true
toc_sticky: true
---

이번에는 Docker Compose없이 Docker를 활용한 kafka Single Broker를 구성해보겠습니다. 

Kraft가 아닌 Zookeeper를 사용해서 구현 할 예정입니다. 

사용할 이미지는 다음과 같습니다.

- zookeeper
- bitnami/kafka

# Network 생성

Container들이 서로 통신 할 수 있도록 network를 생성합니다. Docker 내부의 다른 Container와 통신을 해야하므로 driver는 bridge로 설정합니다.

```shell
docker network create kafka --driver bridge
```

[참고](https://docs.docker.com/network/)

# Zookeeper Container 생성

zookeeper container를 생성합니다.

```shell
docker run -p 2181:2181 -d --network kafka --name zookeeper zookeeper
```

# Broker Container 생성

broker container를 생성합니다.

```shell
docker run -p 9092:9092 -d --network kafka --name kafka-server-1 \
        -e ALLOW_PLAINTEXT_LISTENER=yes \
        -e KAFKA_CFG_ZOOKEEPER_CONNECT=$zookeeper_server \
        bitnami/kafka
```

# Broker Container 생성 확인

log 명령어를 통해서 broker 생성 여부를 확인합니다.

```shell
docker log kafka-server-1
```

# Producer 생성

이제 Broker로 들어가서 간단히 Console로 Message를 보낼 Producer를 생성합니다. bootstrap-server에 연결하고 메시지를 보냅니다.

```shell
docker exec -it kafka-server-2 /bin/bash

kafka-topics.sh --create --bootstrap-server kafka-server-1:9092 --topic test-topic

kafka-console-producer.sh --bootstrap-server kafka-server-1:9092 --topic test-topic
> Hello World!
```

# Consumer 생성

간단한 Consumer를 생성합니다.

```shell
docker exec -it kafka-server-2 /bin/bash

kafka-console-consumer.sh --bootstrap-server kafka-server-1:9092 --topic test-topic
Hello World!
```

# shell script 생성

# Make Kafka Container

Kafka에 필요한 멀티 컨테이너 환경을 최대한 편하게 구축하기 위해 Shell Script를 작성해봅시다.

```shell
#!/bin/bash
port=2181
port_kafka_server_1=9092
name_kafka_server_1="kafka-server-1"
name="zookeeper"
zookeeper_server="${name}:${port}"
network="kafka"

# Must Be Configure network And Installed bitnami/kafka And zookeeper

echo " Config Zookeeper         => port : ${port} network : ${network} "
echo " Config kafka-server1 => port : ${port_kafka_server_1} network : ${name_kafka_server_1} , zookeeper : ${zookeeper_server}"

## MAKE ZOOKEEPER CONTAINER
echo " => Make Zookeeper Container"
docker run -p $port:$port -d --network ${network} --name ${name} zookeeper

## MAKE KAFKA-BROKER-1 CONTAINER
echo " => Make kafka Broker Container"
docker run -p $port_kafka_server_1:$port_kafka_server_1 -d --network $network --name $name_kafka_server_1 \
        -e ALLOW_PLAINTEXT_LISTENER=yes \
        -e KAFKA_CFG_ZOOKEEPER_CONNECT=$zookeeper_server \
        bitnami/kafka
```

```shell
#!/bin/bash
# stop container

docker stop zookeeper kafka-server-1

# rm container
docker rm zookeeper kafka-server-1
```



이제 shell script가 있으므로 귀찮은 명령어 입력을 스킵해도 됩니다.

하지만 이 방법은 여러 개의 컨테이너를 관리하는 매력적인 방법은 아닙니다.

따라서 Docker는 이런 여러 개의 컨테이너를 한꺼번에 관리할 수 있도록 docker compose를 지원합니다.



Docker Compose를 사용하면, 여러 개의 독립적인 서비스를 쉽게 띄울 수 있고, 개발 및 테스트를 빠르게 수행할 수 있습니다. 또한, Docker Compose 파일을 이용하면, 서버, 데이터베이스, 캐싱, 로깅 등 다양한 서비스를 한번에 관리할 수 있으며, 배포 환경에서도 유용하게 사용될 수 있습니다.

