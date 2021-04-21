Kafka Connector - Elastic Search Sink POC
-----------------------------------------

Setup
=========

**Kafkacat install**  
```
apt-get install kafkacat
```

KC in action playbook
=====================

```
$ docker exec -it ksqldb-cli ksql http://ksqldb-server:8088 

CREATE STREAM TEST01 (ROWKEY VARCHAR KEY, COL1 INT, COL2 VARCHAR) WITH (KAFKA_TOPIC='test01', PARTITIONS=1, VALUE_FORMAT='AVRO');


show streams;
describe TEST01;

INSERT INTO TEST01 (ROWKEY, COL1, COL2) VALUES ('X', 1, 'FOO');
INSERT INTO TEST01 (ROWKEY, COL1, COL2) VALUES ('Y', 2, 'BAR');
INSERT INTO TEST01 (ROWKEY, COL1, COL2) VALUES ('Z', 3, 'FOO BAR');


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
  | jq .


curl -XPUT -s http://elastisearch:9200/_cluster/settings \
  -H 'content-type: application/json' \
  -d '{"transient": { "cluster.routing.allocation.disk.threshold_enabled": false }}' \
  | jq .


## issue on create sink connect
## https://github.com/confluentinc/ksql/issues/4127
## included this in kafkadb server config: KSQL_KSQL_CONNECT_URL: http://kafka-connect-540:8083



```




Reference
=========

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

