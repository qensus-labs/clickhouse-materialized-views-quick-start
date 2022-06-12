# clickhouse-materialized-views-quick-start

## Introduction

Formatting incoming data streams can be challenging. Sometimes you just can't adjust the incoming data provider their ingestion model, especially when receiving sensor data. To overcome this ClickHouse has a great feature to create Materialized views.

This quick start guide helps you to get an understanding of

- Short introduction in materialized views
- The benefits of using materialized views
- Example  how to parse logs with ClickHouse

## Short introduction in materialized views

![overview clickhkouse](https://raw.github.com/qensus-labs/clickhouse-materialized-views-quick-start/main/clickhouse.jpg)

Compared with a normal view is that materialized views actually store transformed data,  instead of only returning the result. You can even process data that is written to a Null table, since materialized views store the transformed data stream. This is very helpful when you only want to store the aggregated data, for example receiving data from a Kafka topic or log collector like FluentD, Logstash or Vector. In ClickHouse a materialized view act more as insert trigger. Here you can find more information about the [ClickHouse implementation](https://clickhouse.com/docs/en/sql-reference/statements/create/view/#materialized-view).

## Benefits of using materialized views

Benefits of materialized views are that they are not only providing denormalization, but also helps to

- Increase data productivity.
- Only store the necessary data set.
- Improve query performance.

This way you can easily make your queries faster. Especially when you are using views or have complex queries that contain too much joins that require multiple tables to be scanned.

# Parsing logs with ClickHouse example

It's time to start our small scenario parsing applications logs. We will use to simple Spring boot logs that are inserted into a single column called 'message'. For simplicity we only use a small data set, see [springboot_logs](sample_data/springboot_logs).

Of course I do encourage you to use your own data set.

## Start docker container
```
docker run -d --name clickhouse-playground --ulimit nofile=262144:262144 -p 9000:9000 -p 9009 -p 8123 clickhouse/clickhouse-server:22.5.1
```
## Startup the clickhouse-client
```
docker run -it --rm --link clickhouse-playground:clickhouse-server clickhouse/clickhouse-client:22.5.1 --host clickhouse-server
```

## Create database
```sql
CREATE DATABASE IF NOT EXISTS demodb
```

## Show database
```sql
SHOW database
```

## Create table
```sql
CREATE TABLE IF NOT EXISTS  demodb.logs (
    message String
) 
ENGINE = MergeTree()
ORDER BY tuple()
```

## Connect to database and see logs table
```sql
USE demodb
```

## Show tables
```sql
SHOW tables
```

## Import sample logs
```sql
INSERT INTO demodb.logs (message) FROM infile '/sample_data/springboot_logs' format LineAsString
```

## Show logs
```sql
select * from demodb.logs
```

## Create materialized view
```sql
SELECT splitByWhitespace('2022-05-17 22:37:41 [main] ERROR c.e.logger.LoggerQensusDemoApp - This is an error')
```

## Regex msg 
```sql
SELECT splitByRegexp('\s-\s', '2022-05-17 22:37:41 [main] ERROR c.e.logger.LoggerQensusDemoApp - This is an error')
```

## Regex timestamp
```sql
SELECT splitByRegexp('\s\[', '2022-05-17 22:37:41 [main] ERROR c.e.logger.LoggerQensusDemoApp - This is an error')
```

## Remove garbage
```sql
SELECT trim(BOTH '[]'FROM '[main]')
```

## Create MR 
```sql
CREATE MATERIALIZED VIEW demodb.logs_view
(
    Timestamp DateTime,
    ThreadName String,
    LogLevel String,
    LoggerName String,
    LogMessage String
)
ENGINE = MergeTree()
ORDER BY LoggerName
POPULATE AS
WITH 
    splitByWhitespace(message) as split,
    splitByRegexp('\s\[', message) as ts,
    splitByRegexp('\s-\s', message) as msg
SELECT
    parseDateTimeBestEffort(ts[1]) AS Timestamp,
    trim(BOTH '[]'FROM split[3]) AS ThreadName,
    split[4] AS LogLevel,
    split[5] AS LoggerName,
    msg[2] AS LogMessage
FROM 
    (SELECT message FROM demodb.logs)
```

## View data from materialized view (MV)
```sql
select * from demodb.logs_view
```

## Insert additional record
```sql
INSERT INTO demodb.logs (message) values ('2022-06-10 23:48:42 [main] INFO  c.e.l.L.LoggerQensusDemoApp - Hello World')
```

## View data from materialized view (MV) again
```sql
select * from demodb.logs_view
```
