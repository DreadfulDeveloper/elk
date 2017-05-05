# Docker Compose for ELK stack
A docker compose to set up an ELK stack from Elastic's latest images (currently 5.3.2).

## Elasticsearch
- Ports 9200 and 9300 are exposed to the docker host so data can be shipped to elastic from outside docker.

```
 elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:5.3.2
    command: elasticsearch -E network.host=0.0.0.0 -E discovery.zen.minimum_master_nodes=1
    ports:
      - "9200:9200"
      - "9300:9300"
```
- A sample template mapping is also included

````
{
  "template" : "logs-*",
  "order" : 0,
  "settings" : {
    "index.refresh_interval" : "5s"
  },
  "mappings" : {
    "_default_" : {
      "dynamic_templates" : [ {
        "message_field" : {
          "mapping" : {
            "omit_norms" : true,
            "type" : "keyword"
          },
          "match_mapping_type" : "keyword",
          "match" : "message"
        }
      }, {
        "keyword_fields" : {
          "mapping" : {
            "omit_norms" : true,
            "type" : "keyword"
          },
          "match_mapping_type" : "keyword",
          "match" : "*"
        }
      } ],
      "properties" : {
        "@version" : {
          "index" : "not_analyzed",
          "type" : "keyword"
        }
      },
      "_all" : {
        "enabled" : true
      }
    }
  },
  "aliases" : { }
}
```


## Logstash
- The included config file sets logstash up to receive logs from Elastic's Beats framework and send logs to elasticsearch

```
input {
  beats {
    port => 5044
  }
}

output {
  if [@metadata][beat] {
    elasticsearch {
      hosts => "elasticsearch:9200"
      manage_template => false
      index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
      document_type => "%{[@metadata][type]}"
      user => elastic
      password => changeme
    }
  }
}

```

- Port 5000 and 5044 are exposed to the docker host so logs can be shipped from outside docker.

```
logstash:
  image: docker.elastic.co/logstash/logstash:5.3.2
  command: logstash -f /etc/logstash/conf.d/logstash.conf
  volumes:
    - ./logstash-docker.conf:/etc/logstash/conf.d/logstash.conf
    - ./logs-template.json:/etc/logstash/templates/logs-template.json
  ports:
    - "5000:5000"
    - "5044:5044"
  links:
    - elasticsearch
```


## Kibana
- Port 5601 is exposed to the docker host so Kibana can be accessed from outside docker

```
kibana:
  image: docker.elastic.co/kibana/kibana:5.3.2
  ports:
    - "5601:5601"
  links:
    - elasticsearch
```
