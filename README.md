# KafkaPOC


1. Start Zookeeper docker:

```
docker run -d \
    --net=host \
    --name=zookeeper \
    -e ZOOKEEPER_CLIENT_PORT=32181 \
    -e ZOOKEEPER_TICK_TIME=2000 \
    confluentinc/cp-zookeeper:3.2.0
```

2. Start Kafka docker:

```
docker run -d \
    --net=host \
    --name=kafka \
    -e KAFKA_ZOOKEEPER_CONNECT=localhost:32181 \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:29092 \
    confluentinc/cp-kafka:3.2.0
```

3.  Start the Schema Registry docker:

```
docker run -d \
  --net=host \
  --name=schema-registry \
  -e SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=localhost:32181 \
  -e SCHEMA_REGISTRY_HOST_NAME=localhost \
  -e SCHEMA_REGISTRY_LISTENERS=http://localhost:8081 \
  confluentinc/cp-schema-registry:3.2.0
```

4. Now let’s start up Kafka Connect. Connect stores config, status, and offsets of the connectors in Kafka topics. We will create these topics now using the Kafka broker we created.

```
docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:3.2.0 \
  kafka-topics --create --topic quickstart-avro-offsets --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
```

```
  docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:3.2.0 \
  kafka-topics --create --topic quickstart-avro-config --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
```

```
docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:3.2.0 \
  kafka-topics --create --topic quickstart-avro-status --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
```

5. Create a folder named jars in /tmp/quickstart

```
mkdir -p /tmp/quickstart/jars 
```

6. Download the Microsoft JDBC Driver for SQL Server and place the jar in tmp/quickstart/jars

Link: https://www.microsoft.com/en-us/download/details.aspx?id=57175

7. Start a connect worker with Avro support.

```
docker run -d \
  --name=kafka-connect-avro \
  --net=host \
  -e CONNECT_BOOTSTRAP_SERVERS=localhost:29092 \
  -e CONNECT_REST_PORT=28083 \
  -e CONNECT_GROUP_ID="quickstart-avro" \
  -e CONNECT_CONFIG_STORAGE_TOPIC="quickstart-avro-config" \
  -e CONNECT_OFFSET_STORAGE_TOPIC="quickstart-avro-offsets" \
  -e CONNECT_STATUS_STORAGE_TOPIC="quickstart-avro-status" \
  -e CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_KEY_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_VALUE_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL="http://localhost:8081" \
  -e CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL="http://localhost:8081" \
  -e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_REST_ADVERTISED_HOST_NAME="localhost" \
  -e CONNECT_LOG4J_ROOT_LOGLEVEL=DEBUG \
  -e CONNECT_PLUGIN_PATH="/usr/share/java/" \
  -v /tmp/quickstart/file:/tmp/quickstart \
  -v /tmp/quickstart/jars:/etc/kafka-connect/jars \
  confluentinc/cp-kafka-connect:3.2.0
```

8. Make sure that the connect worker is healthy.

```
docker logs kafka-connect-avro | grep started
```

9. You should see the following output in your terminal window:

```
[2016-08-25 19:18:38,517] INFO Kafka Connect started (org.apache.kafka.connect.runtime.Connect)
[2016-08-25 19:18:38,557] INFO Herder started (org.apache.kafka.connect.runtime.distributed.DistributedHerder)
```

10. Open MS SQL Server and create a database named `kafka-ms-sql` and a table named `kafka_test`

11. Set the CONNECT_HOST. 

```
export CONNECT_HOST=localhost
```

12. Create the JDBC Source connector.

```
curl -X POST \
  -H "Content-Type: application/json" \
  --data '{ "name": "quickstart-jdbc-source", "config": { "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector", "tasks.max": 1, "connection.url": "jdbc:sqlserver://kafka-ms-sql-server.database.windows.net:1433;database=kafka-ms-sql;user=kafka@kafka-ms-sql-server;password=Pakistan@123;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30", "mode": "incrementing", "incrementing.column.name": "id", "timestamp.column.name": "modified", "topic.prefix": "quickstart-jdbc-", "poll.interval.ms": 1000 } }' http://$CONNECT_HOST:28083/connectors
```

13. Output of above command should be like this.
```
{"name":"quickstart-jdbc-source","config":{"connector.class":"io.confluent.connect.jdbc.JdbcSourceConnector","tasks.max":"1","connection.url":"jdbc:sqlserver://kafka-ms-sql-server.database.windows.net:1433;database=kafka-ms-sql;user=kafka@kafka-ms-sql-server;password=Pakistan@123;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30","mode":"incrementing","incrementing.column.name":"id","timestamp.column.name":"modified","topic.prefix":"quickstart-jdbc-","poll.interval.ms":"1000","name":"quickstart-jdbc-source"},"tasks":[],"type":null}
```

14. Check the status of the connector using curl as follows:
```
curl -s -X GET http://$CONNECT_HOST:28083/connectors/quickstart-jdbc-source/status
```
15. The JDBC sink create intermediate topics for storing data. We should see a `quickstart-jdbc-kafka_test` topic.
```
docker run \
   --net=host \
   --rm \
   confluentinc/cp-kafka:3.2.0 \
   kafka-topics --describe --zookeeper localhost:32181
```
16. Now we will read from the `quickstart-jdbc-kafka_test topic` to check if the connector works.
```
docker run \
 --net=host \
 --rm \
 confluentinc/cp-schema-registry:latest \
 kafka-avro-console-consumer --bootstrap-server localhost:29092 --topic quickstart-jdbc-kafka_test --from-beginning --max-messages 10
```

### References:

https://docs.confluent.io/3.2.0/cp-docker-images/docs/tutorials/connect-avro-jdbc.html

https://github.com/confluentinc/cp-docker-images/issues/346

https://github.com/confluentinc/kafka-connect-jdbc/issues/344
