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
docker exec -it elasticsearch curl --data-binary @/usr/share/elasticsearch/config/docker_host/cluster.settings -H 'Content-Type: application/json' -XPUT http://localhost:9200/_cluster/settings
```

#### ILM Policy

```
docker exec -it elasticsearch curl --data-binary @/usr/share/elasticsearch/config/docker_host/ilm.policy -H 'Content-Type: application/json' -XPUT http://localhost:9200/_ilm/policy/hot-warm-cold 
```

#### Index Template

```
docker exec -it elasticsearch curl --data-binary @/usr/share/elasticsearch/config/docker_host/index.template -H 'Content-Type: application/json' -XPUT http://localhost:9200/_index_template/test_indexes
```

### Generate 1000 documents

ILM is configured to use 1000 per generation, i.e. roll over every 1000 documents. 

_from the same directory and docker-compose up_

Find the network ID of the docker-compose

```
docker network ls | grep searchable_snapshot | cut -d ' ' -f 1
```

For example, network id is `b43fce182ec6`

Run:
```
docker run  --network=b43fce182ec6 -v $PWD:/usr/share/logstash/config/docker_host docker.elastic.co/logstash/logstash:7.10.0 bin/logstash -f config/docker_host/ls.config 
```

### Review lifecycle 

From Kibana dev tools : http://localhost:5601/app/dev_tools#/console

```
GET .ds-test-*/_ilm/explain
```