## Создан кластер GCP

## Переклоючаем контекст на gcloud
```
gcloud init
gcloud container clusters get-credentials cluster-1
```
## Установка EFK
```
kubectl create ns microservices-demo
kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Logging/microservices-demo-without-resources.yaml -n microservices-demo
helm repo add elastic https://helm.elastic.co
kubectl create ns observability
helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability
helm upgrade --install kibana elastic/kibana --namespace observability
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm upgrade --install fluent-bit stable/fluent-bit --namespace observability
helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability -f kubernetes-logging/elasticsearch.values.yaml
```

## Установка nginx-ingress

по совету из слака в nginx-ingress.values.yaml были добавлен формат логов в json
```
kubectl create ns nginx-ingress
helm upgrade --install nginx-ingress stable/nginx-ingress --namespace=nginx-ingress --version=1.41.3 -f kubernetes-logging/nginx-ingress.values.yaml
```

##  Kibana

Добавляем хост (ip  - external ip service/nginx-ingress-controller):
```
kubectl get services -n nginx-ingress
helm upgrade --install kibana elastic/kibana --namespace observability -f kubernetes-logging/kibana.values.yaml
```
удаляем лишние ключ времени (time, @timestamp оставил):
```
helm upgrade --install fluent-bit stable/fluent-bit --namespace observability -f kubernetes-logging/fluentbit.values.yaml
```

## Prometheus exporter 
Установил prometheus-operator

```
helm repo add choerodon https://openchart.choerodon.com.cn/choerodon/c7n
helm upgrade --install prometheus  choerodon/kube-prometheus --version 9.3.1 \
 -n observability \
 --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false -f kubernetes-logging/prometeus-grafana.values.yaml
helm upgrade --install elasticsearch-exporter stable/elasticsearch-exporter --set es.uri=http://elasticsearch-master:9200 --set serviceMonitor.enabled=true --namespace=observability
```
admin
prom-operator

## Мониторинг ElasticSearch

 - Импортровал через id dashboard ElasticSearch (4358)
 - Поигрался вс нодами (удалил, восстановил)

## EFK | nginx ingress

Добавил в nginx-ingress values:
```
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      namespace: "nginx-ingress"
      namespaceSelector: {}
      scrapeInterval: 30s
  podAnnotations: {
    fluentbit.io/parser: k8s-nginx-ingress
  }
```

Find requests that contain the number 200, in any field
200
Find 200 in the status field
status:200
Find all status codes between 400-499
status:[400 TO 499]
Find status codes 400-499 with the extension php
status:[400 TO 499] AND extension:PHP
Find status codes 400-499 with the extension php or html
status:[400 TO 499] AND (extension:php OR extension:html)
 
















