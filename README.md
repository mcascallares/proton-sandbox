# proton-sandbox

This repo shows in action [timeplus-io/proton](https://github.com/timeplus-io/proton), as a streaming SQL engine.

## Requirements

- Docker engine must be in version 26.1.1 or above
- Docker Desktop 4.30.0 (149282)

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
SELECT *
    FROM frontend_events_1;
```

```
-- Get live count
SELECT count() 
    FROM frontend_events_1;
```

```
-- Filter events by JSON attributes
SELECT _tp_time, raw:ipAddress, raw:requestedUrl 
    FROM frontend_events_1 
    WHERE raw:method='POST';
```

```
-- Show a live ASCII bar chart
SELECT raw:method, count() as cnt, bar(cnt, 0, 40,5) as bar 
    FROM frontend_events_1
    GROUP BY raw:method 
    ORDER BY cnt desc 
    LIMIT 5 by emit_version();
```

```
-- Show the timestamp for each event
SELECT _tp_time 
    FROM frontend_events_1;
```

```
-- Set offset initial behavior 
SELECT * 
    FROM frontend_events_1 
    SETTINGS seek_to='earliest';
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

### Output a join to a new topic

```
-- Create a external stream
CREATE EXTERNAL STREAM join_output_topic(
    _tp_time datetime64(3), 
    ip1 string, 
    ip2 string,
    url1 string, 
    url2 string)
    SETTINGS type='kafka', 
             brokers='kafka-1:9092', 
             topic='join_output_topic', 
             data_format='JSONEachRow',
             one_message_per_row=true;
```

It will fail, we need to create the topic upfront and repeat the 
above command

```
docker compose exec -it kafka-1 bash
```

```
./kafka-topics.sh --create \
    --topic join_output_topic \
    --bootstrap-server kafka-1:9092
```

```
-- Let's create the materialized view that will populate the join
CREATE MATERIALIZED VIEW mv INTO join_output_topic AS 
    SELECT now64() AS _tp_time, 
           stream_1.raw:ipAddress AS ip1,
           stream_2.raw:ipAddress AS ip2,
           stream_1.raw:requestedUrl AS url1,
           stream_2.raw:requestedUrl AS url2
        FROM frontend_events_1 as stream_1
        INNER JOIN frontend_events_2 AS stream_2
        ON stream_1.raw:method = stream_2.raw:method
```

## Custom UDF Functions

```
-- Let's register the function
CREATE FUNCTION three_musketeers(value string)
    RETURNS string
    LANGUAGE JAVASCRIPT AS $$
        function three_musketeers(value) {
            var  musketeers = [ "tomas", "matias", "akhi" ];
            return value.map(v=>musketeers[v.length % 3]);
        }
$$;
```

```
-- Let's use it!
SELEXT raw:requestedUrl, three_musketeers(raw:requestedUrl)
    FROM frontend_events_1;
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

![DBVisualizer](https://github.com/mcascallares/proton-sandbox/blob/main/images/dbvis.png)


## HTTP API

```
-- Let's create a stream
CREATE STREAM my_http_stream(id int, name string);
```

```
SELECT *, _tp_append_time
    FROM my_http_stream
```

... and let's push data with the HTTP API

```
curl -s -X POST http://localhost:3218/proton/v1/ingest/streams/my_http_stream \
-d '{
  "columns": ["id","name"],
  "data": [
    [1,"hello"],
    [2,"world"]
  ]
}'
```

## Pushing data with Connect

WIP
