+++
author = "Quang Nguyen"
title = "Flink Checkpoint Mechanism and How to Setup Flink Cluster on Local Environment"
date = "2024-01-12"
description = "An interesting story about how the Chandy–Lamport algorithm (a snapshot algorithm) is invented, as my valuable lesson when setup the Flink locally"
tags = [
    "technical",
    "programming",
    "flink",
    "data-streaming",
]
categories = [
    "work",
    "programming",
    "life"
]
series = ["Daily"]
toc = false
+++

## Chandy–Lamport algorithm 

>  “The distributed snapshot algorithm described here came about when I visited Chandy, who was then at the University of Texas in Austin. He posed the problem to me over dinner, but we had both had too much wine to think about it right then. The next morning, in the shower, I came up with the solution. When I arrived at Chandy's office, he was waiting for me with the same solution.”

Source: I copied the text from [Wikipedia](https://en.wikipedia.org/wiki/Chandy%E2%80%93Lamport_algorithm).

So I think wine is the actual author of this algorithm. LOL.

And below are some useful links when learning about flink checkpoint mechanism

- [Apache Flink Series on Medium](https://medium.com/@akash.d.goel/apache-flink-series-part-6-4ef9ad38e051)
- [Flink Docs on Fault Tolerance](https://nightlies.apache.org/flink/flink-docs-master/docs/learn-flink/fault_tolerance/)
- [Rundown of Flink's Checkpoints - Flink Forward - Youtube](https://youtu.be/hoLeQjoGBkQ)

## Setup Flink Cluster Locally

I observed that people (that I knew) exploited the [aws](https://aws.amazon.com/managed-service-apache-flink/?trk=1463b60c-38e9-4b74-84e7-397e78d91cc7&sc_channel=el) envirionment for developing purpose. From my point of view, it was totally wasteful since we (or the company) had to pay a lot for the cost of running many developing data-streaming pipelines on the cloud.

My mistake is to try to set up flink using [docker](https://hub.docker.com/_/flink/) image and stuff like `docker-compose`. Actually, `docker` image is one more layer of `abstraction`, hiding stuff from me as a developer. I found out that it is not convinient for me to pack the flink app as a jar and submit it to `flink job manager` for executing.

If I add `flink-runtime-web` as a flink job dependency, then when I start the flink job using `Intelij`, I can have `Flink UI` with the url `localhost:8081`. However, the disadvantage here is that it's hard to configure stuff like [`State Backends`](https://nightlies.apache.org/flink/flink-docs-master/docs/ops/state/state_backends/).

```groovy
testImplementation 'org.apache.flink:flink-runtime-web:1.15.2'
```

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(new Configuration());
```

So I decided to download the flink binary (actually `docker` image abstract this stuff from us) - [link](https://archive.apache.org/dist/flink/flink-1.15.2/)

Just edit the configuration - file `flink-conf.yaml` - then I can easily configure to use `RocksDB` for checkpoint storage - and I can specify using just my `filesystem` with `file://` protocol instead of relying on complicated stuff like `hadoop`
```yml
jobmanager.rpc.address: localhost
jobmanager.rpc.port: 6123
jobmanager.bind-host: localhost
jobmanager.memory.process.size: 2g
taskmanager.bind-host: localhost
taskmanager.host: localhost
taskmanager.memory.process.size: 4gb
taskmanager.numberOfTaskSlots: 4
parallelism.default: 4

execution.checkpointing.interval: 3min

execution.checkpointing.max-concurrent-checkpoints: 1
state.backend: rocksdb

state.checkpoints.dir: file:///home/CLUSTER/STATE/flink-checkpoints/


state.savepoints.dir: file:///home/CLUSTER/STATE/flink-savepoints/

state.backend.incremental: true

jobmanager.execution.failover-strategy: region

rest.port: 8081
rest.address: localhost
rest.bind-address: localhost
```
Useful commands

```bash
./bin/start-cluster.sh
./bin/stop-cluster.sh
# For faking a failure -> trigger restore from checkpoint state
./bin/stop-taskmanager.sh && ./bin/start-taskmanger.sh
# log can be found in the `log` directory
```

That's all.



