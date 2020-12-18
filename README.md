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

### Setup

#### Cluster Settings
```
docker exec -it hot curl --data-binary @/usr/share/elasticsearch/config/docker_host/cluster.settings -H 'Content-Type: application/json' -XPUT http://hot:9200/_cluster/settings
```

#### ILM Policy

```
docker exec -it hot curl --data-binary @/usr/share/elasticsearch/config/docker_host/ilm.policy -H 'Content-Type: application/json' -XPUT http://hot:9200/_ilm/policy/hot-warm-cold 
```

#### Index Template

```
docker exec -it hot curl --data-binary @/usr/share/elasticsearch/config/docker_host/index.template -H 'Content-Type: application/json' -XPUT http://hot:9200/_index_template/test_indexes
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

### Review lifecycle 

From Kibana dev tools : http://localhost:5601/app/dev_tools#/console

Relevant commands

```
GET /_cat/nodes?v
GET _nodes?filter_path=nodes.*.name,nodes.*.roles
GET /_cat/indices?v
GET .ds-test-*/_ilm/explain


```
