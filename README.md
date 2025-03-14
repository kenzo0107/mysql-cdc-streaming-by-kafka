# Enable MySQL binlog and stream CDC to Kafka.

```mermaid
graph LR

MySQL-->debezium-->kafka

zookeeper-->kafka
kafka-->zookeeper
```

## Initial Setup

```console
$ docker-compose up -d
$ curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json"  http://localhost:8083/connectors
-d @connector.json
```

## Usage

- Show list of topics

```console
$ docker-compose exec kafka kafka-topics.sh --list --bootstrap-server local-kafka:9092

__consumer_offsets
_kafka_connect_configs
_kafka_connect_offsets
_kafka_connect_statuses
schema-changes.test
```

- Insert data

```console
$ docker-compose exec mysql mysql -u root -ppassword test -e "INSERT INTO user(name) VALUES ('dummy');"
```

- Show list of topics again
  You can find new topic `debezium_cdc_topic.test.user`.

```console
$ docker-compose exec kafka kafka-topics.sh --list --bootstrap-server local-kafka:9092

__consumer_offsets
_kafka_connect_configs
_kafka_connect_offsets
_kafka_connect_statuses
debezium_cdc_topic.test.user <-- New
schema-changes.test
```

- Subscribe to a Kafka topic

```console
$ docker-compose exec kafka kafka-console-consumer.sh --bootstrap-server local-kafka:9092 --from-beginning --topic debezium_cdc_topic.test.user

{"before":null,"after":{"id":1,"name":"dummy"},"source":{"version":"2.0.1.Final","connector":"mysql","name":"debezium_cdc_topic","ts_ms":1741931017000,"snapshot":"false","db":"test","sequence":null,"table":"user","server_id":1,"gtid":null,"file":"mysql-bin.000003","pos":369,"row":0,"thread":12,"query":null},"op":"c","ts_ms":1741931017534,"transaction":null}
```

- Insert new data

```console
$ docker-compose exec mysql mysql -u root -ppassword test -e "INSERT INTO user(name) VALUES ('dummy2');"
```

subscribing a kafka topic:

```json
{
  "before": null,
  "after": {
    "id": 2,
    "name": "dummy2"
  },
  "source": {
    "version": "2.0.1.Final",
    "connector": "mysql",
    "name": "debezium_cdc_topic",
    "ts_ms": 1741931384000,
    "snapshot": "false",
    "db": "test",
    "sequence": null,
    "table": "user",
    "server_id": 1,
    "gtid": null,
    "file": "mysql-bin.000003",
    "pos": 658,
    "row": 0,
    "thread": 13,
    "query": null
  },
  "op": "c",
  "ts_ms": 1741931384706,
  "transaction": null
}
```

- Update id=2

```console
$ docker-compose exec mysql mysql -u root -ppassword test -e "UPDATE user SET name = 'dummy2-updated' WHERE id = 2;"
```

subscribing a kafka topic:

```json
{
  "before": {
    "id": 2,
    "name": "dummy2"
  },
  "after": {
    "id": 2,
    "name": "dummy2-updated"
  },
  "source": {
    "version": "2.0.1.Final",
    "connector": "mysql",
    "name": "debezium_cdc_topic",
    "ts_ms": 1741931429000,
    "snapshot": "false",
    "db": "test",
    "sequence": null,
    "table": "user",
    "server_id": 1,
    "gtid": null,
    "file": "mysql-bin.000003",
    "pos": 957,
    "row": 0,
    "thread": 14,
    "query": null
  },
  "op": "u",
  "ts_ms": 1741931429389,
  "transaction": null
}
```

- Delete id=2

```console
$ docker-compose exec mysql mysql -u root -ppassword test -e "DELETE FROM user WHERE id = 2;"
```

subscribing a kafka topic:

```json
{
  "before": {
    "id": 2,
    "name": "dummy2-updated"
  },
  "after": null,
  "source": {
    "version": "2.0.1.Final",
    "connector": "mysql",
    "name": "debezium_cdc_topic",
    "ts_ms": 1741931453000,
    "snapshot": "false",
    "db": "test",
    "sequence": null,
    "table": "user",
    "server_id": 1,
    "gtid": null,
    "file": "mysql-bin.000003",
    "pos": 1268,
    "row": 0,
    "thread": 15,
    "query": null
  },
  "op": "d",
  "ts_ms": 1741931453248,
  "transaction": null
}
```


### Subscribe topic for schema-changes

```console
$ docker-compose exec kafka kafka-console-consumer.sh --bootstrap-server local-kafka:9092 --from-beginning --topic schema-changes.test
```

```json
{
  "source" : {
    "server" : "debezium_cdc_topic"
  },
  "position" : {
    "ts_sec" : 1741930871,
    "file" : "mysql-bin.000003",
    "pos" : 157,
    "snapshot" : true
  },
  "ts_ms" : 1741930872124,
  "databaseName" : "",
  "ddl" : "SET character_set_server=utf8mb4, collation_server=utf8mb4_0900_ai_ci",
  "tableChanges" : [ ]
}
{
  "source" : {
    "server" : "debezium_cdc_topic"
  },
  "position" : {
    "ts_sec" : 1741930872,
    "file" : "mysql-bin.000003",
    "pos" : 157,
    "snapshot" : true
  },
  "ts_ms" : 1741930872133,
  "databaseName" : "test",
  "ddl" : "DROP TABLE IF EXISTS `test`.`user`",
  "tableChanges" : [ ]
}
{
  "source" : {
    "server" : "debezium_cdc_topic"
  },
  "position" : {
    "ts_sec" : 1741930872,
    "file" : "mysql-bin.000003",
    "pos" : 157,
    "snapshot" : true
  },
  "ts_ms" : 1741930872136,
  "databaseName" : "test",
  "ddl" : "DROP DATABASE IF EXISTS `test`",
  "tableChanges" : [ ]
}
{
  "source" : {
    "server" : "debezium_cdc_topic"
  },
  "position" : {
    "ts_sec" : 1741930872,
    "file" : "mysql-bin.000003",
    "pos" : 157,
    "snapshot" : true
  },
  "ts_ms" : 1741930872138,
  "databaseName" : "test",
  "ddl" : "CREATE DATABASE `test` CHARSET utf8mb4 COLLATE utf8mb4_0900_ai_ci",
  "tableChanges" : [ ]
}
{
  "source" : {
    "server" : "debezium_cdc_topic"
  },
  "position" : {
    "ts_sec" : 1741930872,
    "file" : "mysql-bin.000003",
    "pos" : 157,
    "snapshot" : true
  },
  "ts_ms" : 1741930872139,
  "databaseName" : "test",
  "ddl" : "USE `test`",
  "tableChanges" : [ ]
}
{
  "source" : {
    "server" : "debezium_cdc_topic"
  },
  "position" : {
    "ts_sec" : 1741930872,
    "file" : "mysql-bin.000003",
    "pos" : 157,
    "snapshot" : true
  },
  "ts_ms" : 1741930872157,
  "databaseName" : "test",
  "ddl" : "CREATE TABLE `user` (\n  `id` int NOT NULL AUTO_INCREMENT,\n  `name` varchar(25) NOT NULL,\n  PRIMARY KEY (`id`)\n) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci",
  "tableChanges" : [ {
    "type" : "CREATE",
    "id" : "\"test\".\"user\"",
    "table" : {
      "defaultCharsetName" : "utf8mb4",
      "primaryKeyColumnNames" : [ "id" ],
      "columns" : [ {
        "name" : "id",
        "jdbcType" : 4,
        "typeName" : "INT",
        "typeExpression" : "INT",
        "charsetName" : null,
        "position" : 1,
        "optional" : false,
        "autoIncremented" : true,
        "generated" : true,
        "comment" : null,
        "hasDefaultValue" : false,
        "enumValues" : [ ]
      }, {
        "name" : "name",
        "jdbcType" : 12,
        "typeName" : "VARCHAR",
        "typeExpression" : "VARCHAR",
        "charsetName" : "utf8mb4",
        "length" : 25,
        "position" : 2,
        "optional" : false,
        "autoIncremented" : false,
        "generated" : false,
        "comment" : null,
        "hasDefaultValue" : false,
        "enumValues" : [ ]
      } ],
      "attributes" : [ ]
    },
    "comment" : null
  } ]
}
```
