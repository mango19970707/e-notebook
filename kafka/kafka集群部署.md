### kafka集群部署

---

在3个服务器上部署3节点的kafka集群。
```yaml
version: "3"

volumes:
  kafka_data1:
  kafka_data2:
  kafka_data3:

services:
  kafka1:
    image: 'bitnami/kafka:latest'
    container_name: kafka1
    environment:
      - KAFKA_HEAP_OPTS=-Xmx1024m -Xms1024m
      - KAFKA_ENABLE_KRAFT=yes
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://192.168.110.6:19092  # 传递回客户端的元数据，填写宿主机IP地址
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@127.0.0.1:9093
      - KAFKA_BROKER_ID=1
      - KAFKA_CFG_NODE_ID=1
      - KAFKA_KRAFT_CLUSTER_ID=jkUlhzQmQkic54LMxrB1oV
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      - ALLOW_PLAINTEXT_LISTENER=yes
    volumes:
      - "kafka_data1:/bitnami"
    ports:
      - "19092:9092"
    networks:
      kafka:
        aliases:
          - kafka

  kafka2:
    image: 'bitnami/kafka:latest'
    container_name: kafka2
    environment:
      - KAFKA_HEAP_OPTS=-Xmx1024m -Xms1024m
      - KAFKA_ENABLE_KRAFT=yes
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://192.168.110.6:29092  # 传递回客户端的元数据，填写宿主机IP地址
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@127.0.0.1:9093
      - KAFKA_BROKER_ID=2
      - KAFKA_CFG_NODE_ID=2
      - KAFKA_KRAFT_CLUSTER_ID=jkUlhzQmQkic54LMxrB1oV
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      - ALLOW_PLAINTEXT_LISTENER=yes
    volumes:
      - "kafka_data2:/bitnami"
    ports:
      - "29092:9092"
    networks:
      kafka:
        aliases:
          - kafka

  kafka3:
    image: 'bitnami/kafka:latest'
    container_name: kafka3
    environment:
      - KAFKA_HEAP_OPTS=-Xmx1024m -Xms1024m
      - KAFKA_ENABLE_KRAFT=yes
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://192.168.110.6:39092  # 传递回客户端的元数据，填写宿主机IP地址
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@127.0.0.1:9093
      - KAFKA_BROKER_ID=3
      - KAFKA_CFG_NODE_ID=3
      - KAFKA_KRAFT_CLUSTER_ID=jkUlhzQmQkic54LMxrB1oV
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      - ALLOW_PLAINTEXT_LISTENER=yes
    volumes:
      - "kafka_data3:/bitnami"
    ports:
      - "39092:9092"
    networks:
      kafka:
        aliases:
          - kafka

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "38080:8080"
    restart: always
    environment:
      - KAFKA_CLUSTERS_0_NAME=local
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka1:9092,kafka2:9092,kafka3:9092
      - KAFKA_CLUSTERS_0_READONLY=true
    depends_on:
      - kafka1
      - kafka2
      - kafka3
    networks:
      kafka:
        aliases:
          - kafka-ui

  console:
    container_name: console
    image: docker.redpanda.com/redpandadata/console:latest
    networks:
      kafka:
        aliases:
          - kafka
    entrypoint: /bin/sh
    command: -c 'echo "$$CONSOLE_CONFIG_FILE" > /tmp/config.yml; /app/console'
    environment:
      CONFIG_FILEPATH: /tmp/config.yml
      CONSOLE_CONFIG_FILE: |
        kafka:
          brokers: ["kafka1:9092","kafka2:9092","kafka3:9092"]
          sasl:
            #enabled: true
            mechanism: PLAIN
            username: bob
            password: Sw@123456
    ports:
      - 10020:8080

networks:
  kafka:
    driver: bridge
    ipam:
      config:
        - subnet: 172.31.16.0/24
```