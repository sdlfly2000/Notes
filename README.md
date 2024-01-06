# Elastic Search docker
memory > 4GB
```
sudo docker run -d --name elasticsearch --net elastic -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e "ELASTIC_PASSWORD=password" -e "ELASTIC_USERNAME=elastic" elasticsearch:8.11.3
```
