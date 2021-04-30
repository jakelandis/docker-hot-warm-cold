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


Find the internal IP address of the minio host
```
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' minio
```

This IP is used below in the "endpoint" config. For example `172.19.0.2` -> ` "endpoint": "172.19.0.2:9000",`

#### Setup cluster - with Kibana



```
PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval": "1s"
  }
}

PUT _snapshot/my_minio_repository
{
  "type": "s3",
  "settings": {
    "bucket": "test",
    "endpoint": "172.19.0.2:9000",
    "protocol" : "http"
  }
}

PUT _ilm/policy/hot-warm-cold-frozen
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
        "min_age": "1m",
        "actions": {}
      },
      "frozen": {
        "min_age": "1m",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "my_minio_repository",
            "force_merge_index": true
          }
        }
      },
      "delete": {
        "min_age": "5m",
        "actions" : {
            "delete" : {
              "delete_searchable_snapshot" : false
            }
          }
      }
    }
  }
}

PUT _index_template/test_indexes
{
  "index_patterns": [
    "test*"
  ],
  "data_stream": {},
  "template": {
    "settings": {
      "index.lifecycle.name": "hot-warm-cold-frozen",
      "index.number_of_replicas" : 0
    }
  }
}

DELETE  /_slm/policy/every20m

PUT /_slm/policy/every15m
{
  "schedule": "0 */15 * * * ?", 
  "name": "<every15m-{now/d}>", 
  "repository": "my_minio_repository", 
  "config": { 
    "indices": [".ds*"] 
  },
  "retention": { 
    "expire_after": "30d", 
    "min_count": 5, 
    "max_count": 50 
  }
}


GET .ds-test-*/_ilm/explain

GET /_cluster/health

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
