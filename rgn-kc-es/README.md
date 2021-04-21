Kafka Connector - Elastic Search Sink POC
=========================================

Setup
-----

### Kafkacat install

```bash
apt-get install kafkacat
```

Tutorial Playbooks
------------------

### Kafka Connect in Action: Elasticsearch

[Kafka Connect in Action: Elasticsearch - Big intro](https://www.youtube.com/watch?v=Cq-2eGxOCc8&t=336s)

```bash
$ docker exec -it ksqldb-cli ksql http://ksqldb-server:8088 

CREATE STREAM TEST01 (ROWKEY VARCHAR KEY, COL1 INT, COL2 VARCHAR) WITH (KAFKA_TOPIC='test01', PARTITIONS=1, VALUE_FORMAT='AVRO');


show streams;
describe TEST01;

INSERT INTO TEST01 (ROWKEY, COL1, COL2) VALUES ('X', 1, 'FOO');
INSERT INTO TEST01 (ROWKEY, COL1, COL2) VALUES ('Y', 2, 'BAR');


describe TEST01;
show topics;

print test01 from beginning;


CREATE SINK CONNECTOR SINK_ELASTIC_TEST_01 WITH (
    'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
    'connection.url' = 'http://elasticsearch:9200',
    'key.converter' = 'org.apache.kafka.connect.storage.StringConverter',
    'type.name' = '_doc',
    'topics' = 'test01',
    'key.ignore' = 'true',
    'schema.ignore' = 'false'
);

show connectors;


curl -s http://elasticsearch:9200/test01/_search \
  -H 'content-type: application/json' \
  -D '{"size": 42}' \
  | jq -c .


curl -s http://elasticsearch:9200/test01/_search \
  -H 'content-type: application/json' \
  -D '{"size": 42}' \
  | jq -c . '.hits.hits[]'



INSERT INTO TEST01 (ROWKEY, COL1, COL2) VALUES ('Z', 1, 'WOO');
-- new value for existing key 'Y' 
INSERT INTO TEST01 (ROWKEY, COL1, COL2) VALUES ('Y', 4, 'PFF');

SET 'auto.offset.reset' = 'earliest';



SELECT ROWKEY, COL1, COL2 FROM TEST01 EMIT CHANGES LIMIT 4;


DROP CONNECTOR SINK_ELASTIC_TEST_01;

docker exec elasticsearch curl -s -XDELETE "http://localhost:9200/test01"


CREATE SINK CONNECTOR SINK_ELASTIC_TEST_02 WITH (
    'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
    'connection.url' = 'http://elasticsearch:9200',
    'key.converter' = 'org.apache.kafka.connect.storage.StringConverter',
    'type.name' = '_doc',
    'topics' = 'test01',
    'key.ignore' = 'false',
    'schema.ignore' = 'false'
);

```

[Deleting documents in Elasticsearch with the sink connector](https://www.youtube.com/watch?v=Cq-2eGxOCc8&t=698s)

```bash
# tombstone  for deletion
# type Y:<enter> <ctrc+c>
kafkacat -b kafka:29092 \
   -P       -t test01 -Z -K:


# connector should handle tombstones: 'behavior.on.null.values' = 'delete'

DROP CONNECTOR SINK_ELASTIC_TEST_02;
docker exec elasticsearch curl -s -XDELETE "http://localhost:9200/test01"


CREATE SINK CONNECTOR SINK_ELASTIC_TEST_03 WITH (
    'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
    'connection.url' = 'http://elasticsearch:9200',
    'key.converter' = 'org.apache.kafka.connect.storage.StringConverter',
    'type.name' = '_doc',
    'topics' = 'test01',
    'key.ignore' = 'false',
    'schema.ignore' = 'false',
    'behavior.on.null.values' = 'delete'
);
```

[Sending data to Elasticsearch without a declared schema](https://www.youtube.com/watch?v=Cq-2eGxOCc8&t=1557s)

```bash
### (not) using schema
### -- override 'value.converter'  

CREATE STREAM TEST_JSON (ROWKEY VARCHAR KEY, COL1 INT, COL2 VARCHAR) WITH (KAFKA_TOPIC='TEST_JSON', PARTITIONS=1, VALUE_FORMAT='JSON');

INSERT INTO TEST_JSON (ROWKEY, COL1, COL2) VALUES ('X', 1, 'FOO');
INSERT INTO TEST_JSON (ROWKEY, COL1, COL2) VALUES ('Y', 2, 'BAR');

CREATE SINK CONNECTOR SINK_ELASTIC_TEST_JSON_C WITH (
    'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
    'connection.url' = 'http://elasticsearch:9200',
    'key.converter' = 'org.apache.kafka.connect.storage.StringConverter',
    'value.converter' = 'org.apache.kafka.connect.json.JsonConverter',
    'value.converter.schemas.enable' = 'false',
    'type.name' = '_doc',
    'topics' = 'TEST_JSON',
    'key.ignore' = 'false',
    'schema.ignore' = 'true'
);


curl -s http://elasticsearch:9200/test_json/_mapping | jq .
```

Quick troubleshoot

```bash
######


curl -XPUT -s http://elastisearch:9200/_cluster/settings \
  -H 'content-type: application/json' \
  -d '{"transient": { "cluster.routing.allocation.disk.threshold_enabled": false }}' \
  | jq .


## issue on create sink connect
## https://github.com/confluentinc/ksql/issues/4127
## included this in kafkadb server config: KSQL_KSQL_CONNECT_URL: http://kafka-connect-540:8083

```

Reference
---------

* [Kafka Connect and Elasticsearch](https://rmoff.net/2019/10/07/kafka-connect-and-elasticsearch/)
* [Kafka Connect Elasticsearch Connector in Action](https://www.confluent.io/blog/kafka-elasticsearch-connector-tutorial/)
* [Integrating Elasticsearch and ksqlDB for Powerful Data Enrichment and Analytics](https://www.confluent.io/blog/elasticsearch-ksqldb-integration-for-data-enrichment-and-analytics/)
* https://github.com/mitch-seymour/mastering-kafka-streams-and-ksqldb/blob/master/chapter-09/docker-compose.yml
* https://github.com/rmoff/kafka-elasticsearch/blob/master/docker-compose.yml
* [Kafka Connect Deep Dive – Converters and Serialization Explained](https://www.confluent.io/blog/kafka-connect-deep-dive-converters-serialization-explained/)
* [From Zero to Hero with Kafka Connect](https://talks.rmoff.net/QZ5nsS/from-zero-to-hero-with-kafka-connect#sDwoflR)
* [CREATE/SHOW CONNECTOR Failure
#4127](https://github.com/confluentinc/ksql/issues/4127)
* [](https://www.confluent.io/blog/kafka-elasticsearch-connector-tutorial/)
