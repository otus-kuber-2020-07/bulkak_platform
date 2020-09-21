gcloud init
gcloud container clusters get-credentials cluster-1
kubectl create ns microservices-demo
kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Logging/microservices-demo-without-resources.yaml -n microservices-demo

helm repo add elastic https://helm.elastic.co
kubectl create ns observability
helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability
helm upgrade --install kibana elastic/kibana --namespace observability

helm repo add stable https://kubernetes-charts.storage.googleapis.com

helm upgrade --install fluent-bit stable/fluent-bit --namespace observability

helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability -f kubernetes-logging/elasticsearch.values.yaml

## Установка nginx-ingress

по совету из слака в nginx-ingress.values.yaml были добавлен формат логов в json

kubectl create ns nginx-ingress
helm upgrade --install nginx-ingress stable/nginx-ingress --namespace=nginx-ingress --version=1.41.3 -f nginx-ingress.values.yaml

##  Kibana

helm upgrade --install kibana elastic/kibana --namespace observability -f kubernetes-logging/kibana.values.yaml


helm upgrade --install fluent-bit stable/fluent-bit --namespace observability -f kubernetes-logging/fluentbit.values.yaml

11 583,65


