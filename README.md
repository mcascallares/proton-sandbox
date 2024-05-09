# proton-sandbox

This repo shows in action [timeplus-io/proton](https://github.com/timeplus-io/proton), as a streaming SQL engine.

## Components

- Proton engine
- Single-node Kafka cluster called kafka-1 (to show interactions with multiple clusters)
- Single-node Kafka cluster called kafka-2
- [owl-shop sample app](https://github.com/cloudhut/owl-shop) producing data to cluster kafka-1
- [owl-shop sample app](https://github.com/cloudhut/owl-shop) producing data to cluster kafka-2

## Running it
```
docker compose up -d
```


