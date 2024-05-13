# proton-sandbox

This repo shows in action [timeplus-io/proton](https://github.com/timeplus-io/proton), as a streaming SQL engine.

## Components

- Proton engine
- Single-node Kafka cluster called kafka-1 (to show interactions with multiple clusters)
- Single-node Kafka cluster called kafka-2
- [owl-shop sample app](https://github.com/cloudhut/owl-shop) producing data to cluster kafka-1
- [owl-shop sample app](https://github.com/cloudhut/owl-shop) producing data to cluster kafka-2


## Launch the environment
```
docker compose up -d
```


## Launch the SQL Client
```
docker compose exec -it proton proton-client
```


## SQL Queries

### Analytics over a single stream

```
-- Create external stream to read data from Kafka
CREATE EXTERNAL STREAM frontend_events_1(raw string)
SETTINGS type='kafka', 
         brokers='kafka-1:9092',
         topic='owlshop-frontend-events';
```

```
-- Scan incoming events
select * from frontend_events_1;
```

```
-- Get live count
select count() from frontend_events_1;
```

```
-- Filter events by JSON attributes
select _tp_time, raw:ipAddress, raw:requestedUrl from frontend_events_1 where raw:method='POST';
```

```
-- Show a live ASCII bar chart
select raw:method, count() as cnt, bar(cnt, 0, 40,5) as bar from frontend_events_1
group by raw:method order by cnt desc limit 5 by emit_version();
```


### Analytics over multiple streams

```
-- Create a second stream, agains the second cluster
CREATE EXTERNAL STREAM frontend_events_2(raw string)
SETTINGS type='kafka', 
         brokers='kafka-2:9093',
         topic='owlshop-frontend-events';
```

```
-- Let's join both streams, from different Kafka clusters
SELECT stream_1.raw:ipAddress, stream_1.raw:method, stream_1.raw:requestedUrl 
    FROM frontend_events_1 as stream_1
INNER JOIN frontend_events_2 AS stream_2
ON stream_1.raw:method = stream_2.raw:method
```

## JDBC Access

### Requirements

- [DBVisualizer CLI](https://www.dbvis.com/download/)
- [Proton JDBC driver](https://github.com/timeplus-io/proton-java-driver/releases) (testing with 0.6.0)

### Instructions

- Add the JDBC Driver to the DBVisualizer
- Configure the URL like `jdbc:proton://localhost:8123`
- By default Proton's query behavior is streaming SQL, looking for new data in the 
  future and never ends. This can be considered as hang for JDBC client. You have 2 options:
    - Use the 8123 port. In this mode, all SQL are ran in batch mode. So select .. from car_live_data will read all existing data.
    - Use 3218 port. In this mode, by default all SQL are ran in streaming mode. Please use select .. from .. LIMIT 100 to stop the query at 100 events. Or use the table function to query historical data, such as select .. from table(car_live_data)..


## Pushing data with Connect

WIP