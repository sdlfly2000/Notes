Detailed in Link[https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html]
# Elastic Search docker
memory > 4GB
```
sudo docker run -d --name elasticsearch --net elastic -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e "ELASTIC_PASSWORD=password" -e "ELASTIC_USERNAME=elastic" elasticsearch:8.11.3
```

# Kibana
```
sudo docker run -d --name kib01 --net elastic -p 5601:5601 docker.elastic.co/kibana/kibana:8.11.3
```
