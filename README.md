# Searchable Snapshot
```
docker-compose up
```
### Access Minio

http://localhost:9000/minio/login

Access Key:
```
AKIAIOSFODNN7EXAMPLE
```
Secret Key:
```
wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

### Access Elasticsearch 

http://localhost:9200

### Access Kibana

http://localhost:5601

### Setup Elasticsearch cluster


#### Validate cluster

From Kibana dev tools : http://localhost:5601/app/dev_tools#/console

```
GET /_cat/nodes?v
GET _nodes?filter_path=nodes.*.name,nodes.*.roles
GET _cat/plugins
```


#### Cluster Settings
```bash
docker exec -it hot curl --data-binary @/usr/share/elasticsearch/config/docker_host/cluster.settings -H 'Content-Type: application/json' -XPUT http://hot:9200/_cluster/settings
```
Kibana:
```
PUT _cluster/settings
{
  "transient": {
    "indices.lifecycle.poll_interval": "1s"
  }
}
```

#### ILM Policy

Command line:
```bash
docker exec -it hot curl --data-binary @/usr/share/elasticsearch/config/docker_host/ilm.policy -H 'Content-Type: application/json' -XPUT http://hot:9200/_ilm/policy/hot-warm-cold 
```

Kibana:
```
PUT _ilm/policy/hot-warm-cold
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_docs": 1000
          }
        }
      },
      "warm": {
        "min_age": "1m",
        "actions": {}
      },
      "cold": {
        "min_age": "3m",
        "actions": {
          "searchable_snapshot" : {
            "snapshot_repository" : "my_minio_repository"
          }
        }
      },
      "delete": {
        "min_age": "10m",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

#### Index Template

```bash
docker exec -it hot curl --data-binary @/usr/share/elasticsearch/config/docker_host/index.template -H 'Content-Type: application/json' -XPUT http://hot:9200/_index_template/test_indexes
```

Kibana:
```
PUT _index_template/test_indexes
{
  "index_patterns": [
    "test*"
  ],
  "data_stream": {},
  "template": {
    "settings": {
      "index.lifecycle.name": "hot-warm-cold",
      "index.number_of_replicas" : 0
    }
  }
}
```

### Setup Repository

Find the internal IP address of the minio host
```
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' minio

```

Open Minio web UI at http://localhost:9000/minio/login and add a bucket "test", and set the permissions to read and write. 

```
PUT _snapshot/my_minio_repository
{
  "type": "s3",
  "settings": {
    "bucket": "test",
    "endpoint": "172.26.0.4:9000",
    "protocol" : "http"
  }
}
```

Should recieve:

```
{
  "acknowledged" : true
}
```


### Generate 1000 documents

ILM is configured to use 1000 per generation, i.e. roll over every 1000 documents. 

_from the same directory and docker-compose up_

Find the network ID of the docker-compose

```
docker network ls | grep docker-hot-warm-cold_esnet | cut -d ' ' -f 1
```

For example, network id is `b43fce182ec6`

Run:
```
docker run  --network=d88371f458a4 -v $PWD:/usr/share/logstash/config/docker_host docker.elastic.co/logstash/logstash:7.10.0 bin/logstash -f config/docker_host/ls.config 
```

### Watch the generations


```
GET /_cat/shards/.ds-test-*?v
GET .ds-test-*/_ilm/explain

GET .ds-test-*/_count
GET _data_stream/test
GET /_cat/indices?v

```

Repeat generation documents to create new generations. 

### See the snapshot in Minio

http://localhost:9000/minio/test/

```
GET restored-.ds-test-000001
GET test/_count
GET /_cat/shards/restored*?v 
```

What kind of magic is this ? 