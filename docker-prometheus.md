---
title: docker-prometheus
created: '2022-03-30T01:50:37.552Z'
modified: '2023-06-18T03:00:25.612Z'
---

# docker-prometheus

```shell
docker run -d \
	--name prometheus \
    -p 9090:9090 \
    -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus

docker run -d \
--name prometheus \
-p 9090:9090 \
-v /home/gary/docker/prometheus:/etc/prometheus \
prom/prometheus 

Custom image
FROM prom/prometheus
ADD prometheus.yml /etc/prometheus/

docker build -t my-prometheus .
docker run -p 9090:9090 my-prometheus

./prometheus --config.file=./prometheus.yml
```



```shell
docker run -d \
  --name="node-exporter" \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```





```yaml
global:
  scrape_interval:     60s
  evaluation_interval: 60s
 
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus
 
  - job_name: node
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: localhost

  - job_name: 'file_ds'
    file_sd_configs:
    - files:
      - targets.json
```



```json
  [
      {
          "targets": ["127.0.0.1:9091"],
          "labels": {
              "job": "shorturl-api",
              "app": "shorturl-api",
              "env": "test",
              "instance": "127.0.0.1:8888"
          }
      }
  ]
```



```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute. 
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.     
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.  
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]

  # 我们自己的商城项目配置
  - job_name: 'mall'
    static_configs:
      # 目标的采集地址
      - targets: ['golang:9080']
        labels:
          # 自定义标签
          app: 'user-api'
          env: 'test'

      - targets: ['golang:9090']
        labels:
          app: 'user-rpc'
          env: 'test'

      - targets: ['golang:9081']
        labels:
          app: 'product-api'
          env: 'test'

      - targets: ['golang:9091']
        labels:
          app: 'product-rpc'
          env: 'test'

      - targets: ['golang:9082']
        labels:
          app: 'order-api'
          env: 'test'

      - targets: ['golang:9092']
        labels:
          app: 'order-rpc'
          env: 'test'

      - targets: ['golang:9083']
        labels:
          app: 'pay-api'
          env: 'test'

      - targets: ['golang:9093']
        labels:
          app: 'pay-rpc'
          env: 'test'
```

```
https://prometheus.io/docs/guides/node-exporter/
https://prometheus.io/docs/instrumenting/exporters/
```



```shell
docker run \
--user root \
-d \
-p 3000:3000 \
--name=grafana \
grafana/grafana
```

```
https://grafana.com/grafana/dashboards/

可以import官方的dashboard模板，需要修改 设置》Variables 》instance
label_values(node_exporter_build_info{job="node"}, instance) 改为prometheus对应的job名称
```

