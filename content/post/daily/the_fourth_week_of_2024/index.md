+++
author = "Quang Nguyen"
title = "The fourth week of 2024: Kafka and local k8s cluster with minikube"
date = "2024-01-12"
description = "I've learned a lot in this week... From a local setup with podman-compose.yaml to k8s with minikube and helm chart"
tags = [
    "technical",
    "programming",
    "flink",
    "data-streaming",
    "kafka",
    "k8s"
]
categories = [
    "work",
    "programming",
    "life"
]
series = ["Daily"]
toc = false
+++

This is my `podman-compose.yml` file:

```yaml
version: '2'
services:

  kafka-ui:
    container_name: kafka-ui
    image: provectuslabs/kafka-ui:latest
    ports:
      - 8090:8080
    depends_on:
      - broker1
      - schema-registry
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: broker1:29092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
      KAFKA_CLUSTERS_0_JMXPORT: 9101

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.3
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - broker1
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'PLAINTEXT://broker1:29092'
      SCHEMA_REGISTRY_KAFKASTORE_SECURITY_PROTOCOL: PLAINTEXT
      SCHEMA_REGISTRY_SCHEMA_REGISTRY_INTER_INSTANCE_PROTOCOL: "http"
      SCHEMA_REGISTRY_LISTENERS: http://schema-registry:8081
    volumes:
      - schema-registry-secrets:/etc/schema-registry/secrets
  
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.3
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-log:/var/lib/zookeeper/log
      - zookeeper-secrets:/etc/zookeeper/secrets
      - zookeeper-other:/var/lib/zookeeper

  broker1:
    image: confluentinc/cp-kafka:7.5.3
    hostname: broker1
    container_name: broker1
    depends_on:
      - zookeeper
    ports:
      - "29092:29092"
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker1:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: zookeeper
    volumes:
      - broker1-data:/var/lib/kafka/data
      - broker1-secrets:/etc/kafka/secrets

  postgresql:
    image: postgres:16.1-alpine
    hostname: postgresql
    container_name: postgresql
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: myStrongP@sswordInCamelCase
      POSTGRES_DB: postgres
    volumes:
      - postgresql-data:/var/lib/postgresql/data

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2019-CU24-ubuntu-20.04
    hostname: sqlserver
    container_name: sqlserver
    ports:
      - "1433:1433"
    environment:
      ACCEPT_EULA: Y
      MSSQL_SA_PASSWORD: yourStrong(!)Password
    volumes:
      - sqlserver-data:/var/opt/mssql/data

volumes:
  schema-registry-secrets:
  broker1-data:
  broker1-secrets:
  postgresql-data:
  sqlserver-data:
  zookeeper-data:
  zookeeper-secrets:
  zookeeper-log:
  zookeeper-other:
```
This is a good local setup with
- kafka
- kafka-ui
- schema-registry
- zookeeper
- sqlserver
- postgresql

and all the important path are mounted to a volume so that I do not lose data when these containers are stopped.

However, I need to test the horizontal up and down scaling of a service that consumed data from kafka and persist them into postgresql server.
So I've to learn something about k8s.
At the beginning, I've tried to converted the podman-compose.yml file into k8s templates and apply them with 'kubectl update...'
And it is not easy as I thought... Then I found out about `helm chart`...
Here is my setup note

```sh
# start k8s locally with minikube
minikube stop && minikube delete
minikube start && \
minikube addons enable metrics-server && \
minikube addons enable ingress


minikube dashboard

minikube ip
# install kafka with helm chart, expose the necessary port to the host machine so that I can dev easily
helm install kafka \
  oci://registry-1.docker.io/bitnamicharts/kafka \
  --set fullnameOverride="kafka-release" \
  --set replicaCount=3 \
  --set persistence.enabled=true \
  --set persistence.mountPath=/bitnami/kafka \
  --set externalAccess.enabled=true \
  --set externalAccess.controller.service.type=ClusterIP \
  --set externalAccess.controller.service.ports.external=9094 \
  --set externalAccess.controller.service.domain=192.168.49.2 \
  --set externalAccess.broker.service.type=ClusterIP \
  --set externalAccess.broker.service.ports.external=9094 \
  --set externalAccess.broker.service.domain=192.168.49.2 \
  --set metrics.kafka.enabled=true
```
And I have to create these below yaml files:
```yaml
# ingress-nginx-controller-patch.yaml
spec:
  template:
    spec:
      containers:
        - name: controller
          ports:
            - containerPort: 9094
              hostPort: 9094
            - containerPort: 9095
              hostPort: 9095
            - containerPort: 9096
              hostPort: 9096
```

```yaml
# config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  9094: "default/kafka-release-controller-0-external:9094"
  9095: "default/kafka-release-controller-1-external:9094"
  9096: "default/kafka-release-controller-2-external:9094"
```
Then apply them so that I can access kafka from the host machine

```sh
# for exposing brokers to host machine
kubectl apply -f config-map.yaml -n ingress-nginx

kubectl patch deployment ingress-nginx-controller --patch "$(cat ingress-nginx-controller-patch.yaml)" -n ingress-nginx
```

and these commands for install `postgresql` with `helm`

```sh
helm install postgresql oci://registry-1.docker.io/bitnamicharts/postgresql \
--set fullnameOverride="postgresql-release" \
--set auth.postgresPassword="myAdminStrongPassword" \
--set auth.database="dev_database" \
--set metrics.enable=true 

# for exposing postgresql to host machine
kubectl port-forward --namespace default svc/postgresql-release 5432:5432
```

And that's enough for my environment for doing some cooling stuff.
That's all for this note.