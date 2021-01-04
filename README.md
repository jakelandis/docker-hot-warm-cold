# Hot Warm Cold
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

#### Setup repository


Open Minio web UI at http://localhost:9000/minio/login and add a bucket "test", and set the permissions to read and write. 

#### Setup cluster - command line 

```bash
docker exec -it hot curl --data-binary @/usr/share/elasticsearch/config/docker_host/cluster.settings -H 'Content-Type: application/json' -XPUT http://hot:9200/_cluster/settings && \
docker exec -it hot curl --data-binary @/usr/share/elasticsearch/config/docker_host/ilm.policy -H 'Content-Type: application/json' -XPUT http://hot:9200/_ilm/policy/hot-warm-cold && \
docker exec -it hot curl --data-binary @/usr/share/elasticsearch/config/docker_host/index.template -H 'Content-Type: application/json' -XPUT http://hot:9200/_index_template/test_indexes && \
docker exec -it hot curl --data "{\"type\":\"s3\",\"settings\":{\"bucket\":\"test\",\"endpoint\":\"$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' minio):9000\",\"protocol\":\"http\"}}" -H 'Content-Type: application/json' -XPUT http://hot:9200/_snapshot/my_minio_repository

```

#### Setup cluster - with Kibana

No need to do this if done with command line.

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

Find the internal IP address of the minio host
```
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' minio

```

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


### Generate documents

ILM is configured to use 1000 per generation, i.e. roll over every 1000 documents. 

_from the same directory and docker-compose up_

Change THREADS and DOCS_PER_THREAD to change how many docs to generate. 

Run:
```
docker run  -e DOCS_PER_THREAD=6000 -e THREADS=1 --network=$(docker network ls | grep docker-hot-warm-cold_esnet | cut -d ' ' -f 1) -v $PWD:/usr/share/logstash/config/docker_host docker.elastic.co/logstash/logstash:7.10.0 bin/logstash -f config/docker_host/ls.config --pipeline.workers=1
```

### Watch the generations - command line

Requires curl and jq installed.

```bash
watch -n .5 ' echo "* all docs in standard index* "; curl -s localhost:9200/.ds-test-*/_count | jq; echo "* all docs including those in searchable snapshot *"; curl -s localhost:9200/test/_count | jq; curl -s localhost:9200/_cat/shards?v'
```


### Watch the generations - Kibana line

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

Command line:

Requires curl and jq installed.

```bash
watch -n .5  ' echo "* all docs in standard index* "; curl -s localhost:9200/.ds-test-*/_count | jq; echo "* all docs including those in searchable snapshot *"; curl -s localhost:9200/test/_count | jq; curl -s localhost:9200/_cat/shards?v'
```

Kibana: 
```
GET restored-.ds-test-000001
GET test/_count
GET /_cat/shards/restored*?v 
```

What kind of magic is this ? 