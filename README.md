# OwnTracks

| Overview | [Phone Setup](/docs/phone.md) | [MQTT Server](/docs/mqtt_server.md) |[MQTT to Kafka](/docs/mqtt_kafka.md) |
|---|----|----|-----|


OwnTracks for displaying participant progress and location - Kafka, KSQL, Kibana and MQTT based integration.

## Architectural overview

![Architecture](/docs/owntracks-arch.png)

Race mapper for displaying participant progress and location - Kafka, KSQL, Kibana and MQTT based integration

![Kibana](/docs/kibana-capture.png)



## Get started

### Prerequisites
- clone this repo!
- Having k8s cluster or deploy each component of project with docker
- enjoy :)

### Setup

#### Install MQTT Broker

We use EMQX.

#### Phone Setup
If you want to have a play with demonstration data (and not bother with phone setup and MQTT server) skip to the next section.

If you want to setup the entire project, you will need a to start with [Phone Setup](/docs/phone.md) 

#### Install Kafka & related software
- Kafka
- Kafka Connect
- KSQL
- Kafka UI

#### MQTT to Kafka
We use this config to start MQTT Connector:

```bash
curl -X POST   -H "Content-Type: application/json"   --data '{ "name": "mqtt", "config":
{
    "connector.class" : "io.confluent.connect.mqtt.MqttSourceConnector",
    "mqtt.server.uri" : "tcp://external.abriment.com:30733",
    "mqtt.topics" : "owntracks/#",
    "kafka.topic" : "data_mqtt",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
    "tasks.max" : "1",
    "confluent.topic.bootstrap.servers" : "kafka-60046646:9092",
    "confluent.topic.replication.factor" : "1"
  }}' http://localhost:8083/connectors
```

The Kafka Connect MQTT connector is used to integrate with existing MQTT servers. The connector is compatible with Confluent Platform, version 4.1 and later. Prior versions do not work with this connector. The Kafka Connect MQTT Source Connector connects to a MQTT broker and subscribes to the specified topics. SSL is supported. For information on how to create SSL keys and certificates, see Creating SSL Keys and Certificates. For the relevant configuration properties, see the MQTT Source Connector configuration reference.

You can use this connector for a 30-day trial period without a license key.

#### Create streams and tables with KSQL
First we setup ksqldb-server and ksqldb-cli then create streams and tables with KSQL:

```bash
$ ksql http://ksql-server:8088

create stream data_demo_stream  (who varchar KEY, batt INTEGER, lon DOUBLE, lat DOUBLE, tst BIGINT, alt INTEGER, vel DOUBLE)
with (kafka_topic = 'data_mqtt', value_format='JSON', KEY_FORMAT = 'KAFKA');

CREATE table runner_status with (value_format='JSON') AS 
select who
, min(vel) as min_speed
, max(vel) as max_speed
, min(GEO_DISTANCE(lat, lon, -33.87014, 151.211945, 'km')) as dist_to_finish
, count(*) as num_events 
from data_demo_stream WINDOW TUMBLING (size 5 minute) 
group by who;


create stream runner_location with (value_format='JSON') as
select who
, tst as event_time
, concat(concat(CAST(LAT AS STRING), ','), CAST(LON AS STRING)) as LOCATION
, alt
, batt
, vel
from data_demo_stream ;
```

#### Kafka to Elasticsearch
First we setup elastic and kibana then create index template with `kafkaconnect` name:

```
curl -XPUT "http://localhost:9200/_template/kafkaconnect/" -H 'Content-Type: application/json' -d'
{
  "template": {
    "settings": {
      "index": {
        "number_of_shards": "1",
        "number_of_replicas": "0"
      }
    },
    "mappings": {
      "dynamic": "false",
      "dynamic_templates": [],
      "properties": {
        "ALT": {
          "type": "integer"
        },
        "BATT": {
          "type": "integer"
        },
        "DIST_TO_FINISH": {
          "type": "double"
        },
        "EVENT_TIME": {
          "type": "date",
          "format": "strict_date_optional_time||epoch_millis||epoch_second"
        },
        "LOCATION": {
          "type": "geo_point",
          "ignore_malformed": false,
          "ignore_z_value": true
        },
        "MAX_SPEED": {
          "type": "integer"
        },
        "MIN_SPEED": {
          "type": "integer"
        },
        "NUM_EVENTS": {
          "type": "integer"
        },
        "VEL": {
          "type": "double"
        }
      }
    },
    "aliases": {}
  }
}'
```

We also use Elasticsearch Sink connector to tranfer data with this config:

```bash
  curl -X "POST" "localhost:8083/connectors/" \
     -H "Content-Type: application/json" \
     -d $'{
  "name": "es_sink_runner_location",
  "config": {
    "schema.ignore": "true",
    "topics": "RUNNER_LOCATION",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter.schemas.enable": false,
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "key.ignore": "true",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "type.name": "kafkaconnect",
    "topic.index.map": "RUNNER_LOCATION:runner_location",
    "connection.url": "http://elastic-81592013:9200",
    "transforms": "ExtractTimestamp",
    "transforms.ExtractTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.ExtractTimestamp.timestamp.field" : "EVENT_TIME"
  }
}' 


  curl -X "POST" "localhost:8083/connectors/" \
     -H "Content-Type: application/json" \
     -d $'{
  "name": "es_sink_runner_status",
  "config": {
    "schema.ignore": "true",
    "topics": "RUNNER_STATUS",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter.schemas.enable": false,
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "key.ignore": "true",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "type.name": "kafkaconnect",
    "topic.index.map": "RUNNER_STATUS:runner_status",
    "connection.url": "http://elastic-81592013:9200",
    "transforms": "ExtractTimestamp",
    "transforms.ExtractTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.ExtractTimestamp.timestamp.field" : "EVENT_TIME"
  }
}'
```

#### Kibana
We can see indexes in kibana then for check data we should create index patterns (runner_location and runner_status)
Also we can import dashboard in this way: 

`Management > Stack Management > Kibana > Saved Objects > Import`

You can import [Dashboard](./scripts/kibana-dashboard.json)

You can see dashboard from `Analytics > Dashboards`

## Contributing

If you want to contribute to the OwnTracks, feel free to fork the repository and submit pull requests. We welcome any contributions, including bug fixes, feature enhancements, and documentation improvements.

