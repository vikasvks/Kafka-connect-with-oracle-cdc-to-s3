# Kafka-connect-with-oracle-cdc-to-s3
# Data ingestion from Oracle to  aws S3 using Confluent’s Oracle CDC Source Connector in Docker 

It is a new way to Capture data that has been added to, updated, or deleted from Oracle RDBMS with Change Data Capture (CDC).
### Confluent’s `Oracle CDC Source connector`
Confluent’s Oracle CDC Source Connector is a plug-in for Kafka Connect, which (surprise) connects Oracle as a source into Kafka as a destination.

⚠️ Some of these components are paid offerings 💰 for production use. Both the Oracle and Confluent license grant you free licence to play with this stuff as a developer for 30 days .Be sure to review the license and download the zip file for the Confluent Oracle CDC Source Connector.

This uses Docker Compose to run the Kafka Connect worker and other kafka dependency.
### Prerequisites
*In docker-compose.yml,I already added the both confluent oracle cdc source and s3 sink connectors (change version if you needed)*
1. Install Docker (for kafka)
2. Running oracle and its details `Hostname`,`username`,`password`,`port`,`sid`,`database` and other details.
3. Create the S3 bucket, make a note of the region
4. Obtain your access key pair
5. set `environment` variable for `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` 

```bash
#In linux
export AWS_ACCESS_KEY_ID=xxxxxxxxxxxxxxxxxxx
export AWS_SECRET_ACCESS_KEY=yyyyyyyyyyyyyyyyyyyyyyy
```
## Getting Started
1. Clone this repo and Bring the Docker Compose up

```bash
docker-compose up -d
```

2. Make sure everything is up and running

```bash
$ docker-compose ps
     Name                  Command               State                    Ports
---------------------------------------------------------------------------------------------
broker            /etc/confluent/docker/run   Up             0.0.0.0:9092->9092/tcp
kafka-connect     bash -c #                   Up (healthy)   0.0.0.0:8083->8083/tcp, 9092/tcp
                  echo "Installing ...
schema-registry   /etc/confluent/docker/run   Up             0.0.0.0:8081->8081/tcp
zookeeper         /etc/confluent/docker/run   Up             2181/tcp, 2888/tcp, 3888/tcp
ksqldb            /usr/bin/docker/run         Up             0.0.0.0:8088->8088/tcp

```

3. Create the oracle cdc Source connector

```javascript
curl -d '{
"name": "oracle-12",
"config": {
"tasks.max":1,
"connector.class": "io.confluent.connect.oracle.cdc.OracleCdcSourceConnector",
"value.converter": "io.confluent.connect.avro.AvroConverter",
"errors.tolerance": "all",
"topic.creation.groups": "redo",
"oracle.server": "<hostname>",
"oracle.port": "1521",
"oracle.sid": "<sid>",
"oracle.username": "<username>",
"oracle.password": "<password>",
"start.from": "snapshot",
"redo.log.poll.interval.ms": "1000",
"redo.log.consumer.bootstrap.servers": "localhost:9092",
"value.converter.schema.registry.url": "http://localhost:8081"
"redo.log.consumer.fetch.min.bytes": "1",
"numeric.mapping": "best_fit",
"table.inclusion.regex": "<databaseName>\\.<username>\\.<tableName>",
"table.topic.name.template": "${connectorName}-${tableName}",
"connection.pool.max.size": "20",
"confluent.topic.bootstrap.servers": "localhost:9092",
"confluent.topic.replication.factor": "1",
"topic.creation.default.partitions": "1",
"topic.creation.redo.partitions": "1",
"topic.creation.redo.include": "redo-log-topic",
"topic.creation.default.replication.factor": "1",
"topic.creation.redo.retention.ms": "1209600000",
"topic.creation.redo.replication.factor": "1",
"topic.creation.default.cleanup.policy": "delete",
"topic.creation.redo.cleanup.policy": "compect",
}
}' -H 'Content-Type: application/json' http://0.0.0.0:8083/connectors/
```
**customise  the connector for your environment** for more information about [oracle cdc source connector](https://docs.confluent.io/kafka-connect-oracle-cdc/current/configuration-properties.html)

4. Check that required connectors are loaded successfully
  (oracle-12 is connector name)
  run this line:
   
         curl http://0.0.0.0:8083/connectors/oracle-12/status | jq .
5. Now Check oracle data in kafka topic

    Run ksqldb cli:
   
       docker exec -it ksqldb ksql http://ksqldb:8088

*  For show connector list `SHOW CONNECTORS;`
*  For show topics list `SHOW TOPICS;`
*  For show connector list `PRINT "ftp-source-topic" FROM BEGINNING;`
for more details of [ksqldb](https://docs.ksqldb.io/en/latest/developer-guide/ksqldb-reference/create-connector/) 
   
6. Now we make s3 sink connector(Kafka topic to s3)

```javascript
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/sink-s3/config \
    -d '
 {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max": "1",
    "topics": "<connectorName-tableName>",
    "s3.region": "us-east-2",
    "s3.bucket.name": "kafkatestdata",
    "flush.size": "3",
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
    "schema.generator.class": "io.confluent.connect.storage.hive.schema.DefaultSchemaGenerator",
    "schema.compatibility": "NONE"
  }'
```


**Things to customise for your environment:**

* `topics` :  the source topic(s) you want to send to S3
* `key.converter` : match the serialisation of your source data [link](see https://www.confluent.io/blog/kafka-connect-deep-dive-converters-serialization-explained/)
* `value.converter` : match the serialisation of your source data [link](see https://www.confluent.io/blog/kafka-connect-deep-dive-converters-serialization-explained/)
* And many more


7. Bravo 🎉🎉,All done now check data into S3.

References

* https://hub.confluent.io [Confluent Hub]
* https://docs.confluent.io/current/connect/kafka-connect-s3/index.html#connect-s3 [S3 Sink connector docs]
* https://docs.confluent.io/kafka-connect-oracle-cdc/current/configuration-properties.html [oracle cdc source connector]
