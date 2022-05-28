For full details see the [demo script](demo_build-a-streaming-pipeline.adoc)

🗒Slides: https://talks.rmoff.net/JS3Heu/apache-kafka-and-ksqldb-in-action-lets-build-a-streaming-data-pipeline

🎥 Recording: https://www.youtube.com/watch?v=Z8_O0wEIafw

# How to Demo CDC Mysql Confluent Platform

## Download CP

Clean and prepare env
```bash
rm -rf /tmp/confluent.*
export CONFLUENT_HOME=/home/ansible/confluent/confluent-7.1.1
export PATH=$PATH:$CONFLUENT_HOME/bin
confluent local services --help
```

Install connector and start CP
```bash
confluent-hub install --no-prompt debezium/debezium-connector-mysql:1.6.0
confluent-hub install --no-prompt confluentinc/kafka-connect-datagen:0.5.0
confluent-hub install --no-prompt confluentinc/kafka-connect-elasticsearch:11.1.1
confluent local services start
```

Open Control-center
```bash
http://localhost:9021/clusters
```

Make sure that the Elasticsearch, Debezium, and DataGen connectors are available:
```bash
curl -s localhost:8083/connector-plugins|jq '.[].class'|egrep 'DatagenConnector|MySqlConnector|ElasticsearchSinkConnector'
```

## Start Docker Mysql and Elastic

Start service
```bash
cd /home/ansible/confluent/demo-scene/build-a-streaming-pipeline
docker-compose up -d
```

Load
```bash
http://localhost:5601/app/kibana#/dashboard/mysql-ksql-kafka-es
```

## Intro to KSQL

Access ksqlDB editor by Control-center


Access ksqlDB editor by command-line:
```bash
ksql http://localhost:8088
SET 'auto.offset.reset' = 'earliest';
```

### KSQL Kafka Connect for streaming data

check C:\Windows\System32\drivers\etc\hosts file to include elasticsearch
```bash
127.0.0.1       elasticsearch
```

```bash
SHOW TOPICS;
```

```bash
PRINT ratings;
```

```bash
CREATE SINK CONNECTOR SINK_ES_RATINGS WITH (
    'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
    'topics'          = 'ratings',
    'connection.url'  = 'http://elasticsearch:9200',
    'type.name'       = '_doc',
    'key.ignore'      = 'false',
    'schema.ignore'   = 'true',
    'transforms'= 'ExtractTimestamp',
    'transforms.ExtractTimestamp.type'= 'org.apache.kafka.connect.transforms.InsertField$Value',
    'transforms.ExtractTimestamp.timestamp.field' = 'RATING_TS'
);
```

Show in Kibana
http://localhost:5601/app/management/kibana/indexPatterns

## Part 1 - ksqlDB for filtering streams

Inspect topics
```bash
SHOW TOPICS;
```

Inspect ratings & define stream
```bash
CREATE STREAM RATINGS WITH (KAFKA_TOPIC='ratings',VALUE_FORMAT='AVRO');
```

Select columns from live stream of data
```bash
SELECT USER_ID, STARS, CHANNEL, MESSAGE FROM RATINGS EMIT CHANGES;
```

Filter live stream of data
```bash
SELECT USER_ID, STARS, CHANNEL, MESSAGE FROM RATINGS WHERE LCASE(CHANNEL) NOT LIKE '%test%' EMIT CHANGES;
```

Create a derived stream
```bash
CREATE STREAM RATINGS_LIVE AS
SELECT * FROM RATINGS WHERE LCASE(CHANNEL) NOT LIKE '%test%' EMIT CHANGES;

CREATE STREAM RATINGS_TEST AS
SELECT * FROM RATINGS WHERE LCASE(CHANNEL) LIKE '%test%' EMIT CHANGES;
```

```bash
SELECT * FROM RATINGS_LIVE EMIT CHANGES LIMIT 5;
SELECT * FROM RATINGS_TEST EMIT CHANGES LIMIT 5;
```

```bash
DESCRIBE RATINGS_LIVE EXTENDED;
```

## Part 2 - ingesting state from a database as an event stream

Show MySQL table + contents

Launch the MySQL CLI:
```bash
docker exec -it mysql bash -c 'mysql -u $MYSQL_USER -p$MYSQL_PASSWORD demo'
```

```bash
SHOW TABLES;
```


```bash
+----------------+
| Tables_in_demo |
+----------------+
| CUSTOMERS      |
+----------------+
1 row in set (0.00 sec)

```


```bash
SELECT ID, FIRST_NAME, LAST_NAME, EMAIL, CLUB_STATUS FROM CUSTOMERS LIMIT 5;
```

```bash
+----+-------------+------------+------------------------+-------------+
| ID | FIRST_NAME  | LAST_NAME  | EMAIL                  | CLUB_STATUS |
+----+-------------+------------+------------------------+-------------+
|  1 | Rica        | Blaisdell  | rblaisdell0@rambler.ru | bronze      |
|  2 | Ruthie      | Brockherst | rbrockherst1@ow.ly     | platinum    |
|  3 | Mariejeanne | Cocci      | mcocci2@techcrunch.com | bronze      |
|  4 | Hashim      | Rumke      | hrumke3@sohu.com       | platinum    |
|  5 | Hansiain    | Coda       | hcoda4@senate.gov      | platinum    |
+----+-------------+------------+------------------------+-------------+
5 rows in set (0.00 sec)
```

### Ingest the data (plus any new changes) into Kafka

In ksqlDB:
```bash

CREATE SOURCE CONNECTOR SOURCE_MYSQL_01 WITH (
    'connector.class' = 'io.debezium.connector.mysql.MySqlConnector',
    'database.hostname' = 'mysql',
    'database.port' = '3306',
    'database.user' = 'debezium',
    'database.password' = 'dbz',
    'database.server.id' = '42',
    'database.server.name' = 'asgard',
    'table.whitelist' = 'demo.customers',
    'database.history.kafka.bootstrap.servers' = 'localhost:9092',
    'database.history.kafka.topic' = 'dbhistory.demo' ,
    'include.schema.changes' = 'false',
    'transforms'= 'unwrap,extractkey',
    'transforms.unwrap.type'= 'io.debezium.transforms.ExtractNewRecordState',
    'transforms.extractkey.type'= 'org.apache.kafka.connect.transforms.ExtractField$Key',
    'transforms.extractkey.field'= 'id',
    'key.converter'= 'org.apache.kafka.connect.storage.StringConverter',
    'value.converter'= 'io.confluent.connect.avro.AvroConverter',
    'value.converter.schema.registry.url'= 'http://localhost:8081'
    );

```

Check that it’s running:
```bash

ksql> SHOW CONNECTORS;

 Connector Name  | Type   | Class                                                         | Status
------------------------------------------------------------------------------------------------------------------------
 SOURCE_MYSQL_01 | SOURCE | io.debezium.connector.mysql.MySqlConnector                    | RUNNING (1/1 tasks RUNNING)
 source-datagen  | SOURCE | io.confluent.kafka.connect.datagen.DatagenConnector           | RUNNING (1/1 tasks RUNNING)
 SINK_ES_RATINGS | SINK   | io.confluent.connect.elasticsearch.ElasticsearchSinkConnector | RUNNING (1/1 tasks RUNNING)
------------------------------------------------------------------------------------------------------------------------
ksql>
```

### Show Kafka topic has been created & populated

```bash
SHOW TOPICS;
```

```bash
ksql> SHOW TOPICS;

 Kafka Topic                 | Partitions | Partition Replicas
---------------------------------------------------------------
 RATINGS_LIVE                | 1          | 1
 RATINGS_TEST                | 1          | 1
 asgard.demo.CUSTOMERS       | 1          | 1
 dbhistory.demo              | 1          | 1
 default_ksql_processing_log | 1          | 1
 mii-confluent-topic         | 1          | 1
 ratings                     | 1          | 1
---------------------------------------------------------------
ksql>
```

Show topic contents
```bash
PRINT 'asgard.demo.CUSTOMERS' FROM BEGINNING;
```

```bash
ksql> PRINT 'asgard.demo.CUSTOMERS' FROM BEGINNING;
Key format: JSON or KAFKA_STRING
Value format: AVRO or KAFKA_STRING
rowtime: 2022/05/28 09:53:04.476 Z, key: 1, value: {"id": 1, "first_name": "Rica", "last_name": "Blaisdell", "email": "rblaisdell0@rambler.ru", "gender": "Female", "club_status": "bronze", "comments": "Universal optimal hierarchy", "create_ts": "2022-05-28T08:46:36Z", "update_ts": "2022-05-28T08:46:36Z"}, partition: 0
rowtime: 2022/05/28 09:53:04.477 Z, key: 2, value: {"id": 2, "first_name": "Ruthie", "last_name": "Brockherst", "email": "rbrockherst1@ow.ly", "gender": "Female", "club_status": "platinum", "comments": "Reverse-engineered tangible interface", "create_ts": "2022-05-28T08:46:36Z", "update_ts": "2022-05-28T08:46:36Z"}, partition: 0
```

Create ksqlDB stream and table
```bash
CREATE TABLE CUSTOMERS (CUSTOMER_ID VARCHAR PRIMARY KEY)
  WITH (KAFKA_TOPIC='asgard.demo.CUSTOMERS', VALUE_FORMAT='AVRO');
```


Query the ksqlDB table:

```bash
SET 'auto.offset.reset' = 'earliest';
SELECT CUSTOMER_ID, FIRST_NAME, LAST_NAME, EMAIL, CLUB_STATUS FROM CUSTOMERS EMIT CHANGES LIMIT 5;
```

Make changes in MySQL, observe it in Kafka
MySQL terminal:

```bash
INSERT INTO CUSTOMERS (ID,FIRST_NAME,LAST_NAME) VALUES (42,'Rick','Astley');
```

```bash
UPDATE CUSTOMERS SET EMAIL = 'rick@example.com' where ID=42;
```

```bash
UPDATE CUSTOMERS SET CLUB_STATUS = 'bronze' where ID=42;
```

```bash
UPDATE CUSTOMERS SET CLUB_STATUS = 'platinum' where ID=42;
```

## Part 03 - ksqlDB for joining streams

Join live stream of ratings to customer data
```bash
SELECT R.RATING_ID, R.MESSAGE, R.CHANNEL,
       C.CUSTOMER_ID, C.FIRST_NAME + ' ' + C.LAST_NAME AS FULL_NAME,
       C.CLUB_STATUS
FROM   RATINGS_LIVE R
       LEFT JOIN CUSTOMERS C
         ON CAST(R.USER_ID AS STRING) = C.CUSTOMER_ID
WHERE  C.FIRST_NAME IS NOT NULL
EMIT CHANGES;
```


Persist this stream of data
```bash
SET 'auto.offset.reset' = 'earliest';
CREATE STREAM RATINGS_WITH_CUSTOMER_DATA
       WITH (KAFKA_TOPIC='ratings-enriched')
       AS
SELECT R.RATING_ID, R.MESSAGE, R.STARS, R.CHANNEL,
       C.CUSTOMER_ID, C.FIRST_NAME + ' ' + C.LAST_NAME AS FULL_NAME,
       C.CLUB_STATUS, C.EMAIL
FROM   RATINGS_LIVE R
       LEFT JOIN CUSTOMERS C
         ON CAST(R.USER_ID AS STRING) = C.CUSTOMER_ID
WHERE  C.FIRST_NAME IS NOT NULL
EMIT CHANGES;
```

### Create stream of unhappy VIPs

```bash
CREATE STREAM UNHAPPY_PLATINUM_CUSTOMERS AS
SELECT FULL_NAME, CLUB_STATUS, EMAIL, STARS, MESSAGE
FROM   RATINGS_WITH_CUSTOMER_DATA
WHERE  STARS < 3
  AND  CLUB_STATUS = 'platinum'
PARTITION BY FULL_NAME;
```

### Stream to Elasticsearch

```bash
CREATE SINK CONNECTOR SINK_ELASTIC_01 WITH (
  'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
  'connection.url' = 'http://elasticsearch:9200',
  'type.name' = '',
  'behavior.on.malformed.documents' = 'warn',
  'errors.tolerance' = 'all',
  'errors.log.enable' = 'true',
  'errors.log.include.messages' = 'true',
  'topics' = 'ratings-enriched,UNHAPPY_PLATINUM_CUSTOMERS',
  'key.ignore' = 'true',
  'schema.ignore' = 'true',
  'key.converter' = 'org.apache.kafka.connect.storage.StringConverter',
  'transforms'= 'ExtractTimestamp',
  'transforms.ExtractTimestamp.type'= 'org.apache.kafka.connect.transforms.InsertField$Value',
  'transforms.ExtractTimestamp.timestamp.field' = 'EXTRACT_TS'
);
```


Check status
```bash
ksql> SHOW CONNECTORS;

 Connector Name  | Type   | Class                                                         | Status
------------------------------------------------------------------------------------------------------------------------
 SOURCE_MYSQL_01 | SOURCE | io.debezium.connector.mysql.MySqlConnector                    | RUNNING (1/1 tasks RUNNING)
 SINK_ELASTIC_01 | SINK   | io.confluent.connect.elasticsearch.ElasticsearchSinkConnector | RUNNING (1/1 tasks RUNNING)
 source-datagen  | SOURCE | io.confluent.kafka.connect.datagen.DatagenConnector           | RUNNING (1/1 tasks RUNNING)
 SINK_ES_RATINGS | SINK   | io.confluent.connect.elasticsearch.ElasticsearchSinkConnector | RUNNING (1/1 tasks RUNNING)
------------------------------------------------------------------------------------------------------------------------
ksql>
```

Check status
```bash
curl -s "http://localhost:9200/_cat/indices/*?h=idx,docsCount"
```


### View in Kibana

http://localhost:5601/app/kibana#/dashboard/mysql-ksql-kafka-es