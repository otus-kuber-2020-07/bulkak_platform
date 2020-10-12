## GitLab

 - зарегистрировал аккаунт в gitlab.com
 - создал проект microservices-demo
 - добавил туда helm-chart  из демо-репы

```
https://gitlab.com/welcome_to_hell_man/microservices-demo
```


## Continuous Integration | Задание со ⭐

 - по лекции по CI-CD начала пробовать составить конфиг .gitlab-ci.yml для билда и  пуша в докер-хаб
 - составил конфиг как в лекции, добавил в настройки проекта на гитлабе свои переменные переменные CONTAINER и DOCKER_TOKEN
 - создал на докер-хабе репозитории для сервисов
 - При создании тэга запустился pipline, но выдал ошибку
 ```
This job is stuck because you don't have any active runners online or available with any of these tags assigned to them: shell
Go to project CI settings

 ```
 - пошел устанавливать раннеры локально (https://docs.gitlab.com/runner/install/linux-manually.html)

```
curl -LJO https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb
sudo dpkg -i gitlab-runner_amd64.deb
sudo gitlab-runner register --url https://gitlab.com/ --registration-token xxx --executor shell
sudo gitlab-runner run
```

 - сделал тэг и возрадовался автоматизации в быту - билд и деплой прошли на ура, в докер-хабе имэджи с тэгами, сказка какая-то. 

## Подготовка Kubernetes кластера

 - включил istio  в настройках кластера

# GitOps

## Подготовка

 - Установим CRD, добавляющую в кластер новый ресурс - HelmRelease:

```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/helm-v3-dev/deploy/flux-helm-release-crd.yaml
```

 - Добавим официальный репозиторий Flux

```
helm repo add fluxcd https://charts.fluxcd.io
```

 - Произведем установку Flux в кластер, в namespace flux

 ```
kubectl create namespace flux
helm upgrade --install flux fluxcd/flux -f flux.values.yaml --namespace flux
 ```

  - Установим Helm operator:

```
helm upgrade --install helm-operator fluxcd/helm-operator -f helm-operator.values.yaml --namespace flux
```

  - установил fluxctl 
  - добавил ssh ключ в гитлаб

## Проверка

 - запушил папку с файлом конфига  ns. Самом собой просто запушить не достаточно если только не пушить сразу в мастер (а мы знаем что это моветон). Я как хороший мальчик запушил в свою супер-веточку r0.0.1 и вмержил её в мастер. И узрел появление неймспейса в кластере.
 - так же разрыл логи на поде flux
 ```
ts=2020-10-05T19:01:50.441768627Z caller=sync.go:61 component=daemon info="trying to sync git changes to the cluster" old=7a5b70eaf77020eae8ea68f3417a41fcaf8f974b new=d10726854ea9602d175c0bf8bf48356e098cbb9f
ts=2020-10-05T19:01:51.548541842Z caller=sync.go:540 method=Sync cmd=apply args= count=1
ts=2020-10-05T19:01:52.177551715Z caller=sync.go:606 method=Sync cmd="kubectl apply -f -" took=628.944969ms err=null output="namespace/microservices-demo created"
ts=2020-10-05T19:01:52.179446317Z caller=daemon.go:701 component=daemon event="Sync: 91c3d62, <cluster>:namespace/microservices-demo" logupstream=false
ts=2020-10-05T19:01:53.259246201Z caller=loop.go:236 component=sync-loop state="tag flux-sync" old=7a5b70eaf77020eae8ea68f3417a41fcaf8f974b new=d10726854ea9602d175c0bf8bf48356e098cbb9f
ts=2020-10-05T19:01:54.085628444Z caller=loop.go:134 component=sync-loop event=refreshed url=ssh://git@gitlab.com/welcome_to_hell_man/microservices-demo.git branch=master HEAD=d10726854ea9602d175c0bf8bf48356e098cbb9f
 ```

  - так же обнаружил что flux  сделал себе тэг flux-sync. Это прям скаже не культурно было с его стороны. Хорошо я раннеры погасил, а то билдилось бы там и билдилось ненужные мне образы с ненужными тэгами =(

## HelmRelease

 - релиз не накатывался, в логах ругался:

```
"ts=2020-10-05T20:03:10.100108527Z caller=release.go:316 component=release release=frontend targetNamespace=microservices-demo resource=microservices-demo:helmrelease/frontend helmVersion=v3 error="installation failed: unable to build kubernetes objects from release manifest: unable to recognize \"\": no matches for kind \"ServiceMonitor\" in version \"monitoring.coreos.com/v1\"" phase=install
```

 - ну я так понял ему нехватало CRD ServiceMonitor,  я решил не мелочитсья и поставил prometheus О_о (наверно можно было просто CRD  отдельно накатить, но чего мелочиться)

```
kubectl create ns monitoring
helm repo add choerodon https://openchart.choerodon.com.cn/choerodon/c7n
helm upgrade --install prometheus  choerodon/kube-prometheus --version 9.3.1 \
 -n microservices-demo \
 --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```
 - после манипуляций увидел долгожданное:

```
$ kubectl get helmrelease -n microservices-demo
NAME       RELEASE    STATUS     MESSAGE                                                                       AGE
frontend   frontend   deployed   Release was successful for Helm release 'frontend' in 'microservices-demo'.   26m
$ helm list -n microservices-demo 
NAME            NAMESPACE               REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
frontend        microservices-demo      1               2020-10-05 20:12:10.292369803 +0000 UTC deployed        frontend-0.21.0         1.16.0     
prometheus      microservices-demo      1               2020-10-05 23:12:10.863760611 +0300 MSK deployed        kube-prometheus-9.3.1   0.38.1 
```

  - запушил изменения в сервис frontend, создал в гитлабе тэг, запустилась сборка и пуш тэгированных образов в docker-hub, красота.
  - но почему-то автоматика не захотела следить за тэгами моего образа( 
  - Покумекал - нашел что в values  чарта  frontend указан  image из примера, не мой - поправил. Все равно не обновляет
  - Еще покумекал. пошел в документацию flux, сравнил с предоставленным конфигом, нашел отличия и странности (flux.weave.works/tag.chart-image - что это вообще, из прошлых версий наверно. вообще конечно вся домашка в неактуальных ссылках и конфигах, печаль), поправил.
  - Увидел в логах запись, похоже на то что нужно:
```
"ts=2020-10-06T19:37:30.03630873Z caller=warming.go:198 component=warmer info="refreshing image" image=bulkak/hipster-frontend tag_count=6 to_update=1 of_which_refresh=0 of_which_missing=1
```
```
"ts=2020-10-06T19:37:31.047485925Z caller=warming.go:206 component=warmer updated=bulkak/hipster-frontend successful=1 attempted=1
"
```
 - не все равно ничего не присходит. Пошел в логи:
```
$ kubectl get helmrelease -n microservices-demo 
NAME       RELEASE    STATUS     MESSAGE                                                               AGE
frontend   frontend   deployed   Release failed for Helm release 'frontend' in 'microservices-demo'.   24h
```
```
$ kubectl describe helmrelease -n microservices-demo
  Warning  FailedReleaseSync  74s (x25 over 25m)    helm-operator  synchronization of release 'frontend' in namespace 'microservices-demo' failed: dry-run upgrade failed: dry-run upgrade for comparison failed: error validating "": error validating data: ValidationError(Deployment.spec.template.spec.containers[0].image): invalid type for io.k8s.api.core.v1.Container.image: got "map", expected "string"
```
 - логи говорят накосячил в конфиге deployment. подправил
 - все равно ничего!

 -----

  - утро вечера мудреннее, надо было не выпендриваться с названием тэгов. Сделал просто 0.0.1 - все сработало как надо. (у меня были r.0.0.1, хотя странно, ведь в задании v0.0.1 - тоже не проходит по официальной регулярке semver https://regex101.com/r/vkijKf/1/ . НО! доке есть даже целый абзац про как раз этот 'v', пишут там что 'prefixing a semantic version with a “v” is a common way (in English) to indicate it is a version number' и норм им. а мой не очень common way 'r.' им не угодил. тьфу)

## Обновление Helm chart

 - поменял имя deployment
 - и видим в логах строчки:

 ```
ts=2020-10-07T21:24:49.999521819Z caller=helm.go:69 component=helm version=v3 info="creating upgraded release for frontend" targetNamespace=microservices-demo release=frontend

ts=2020-10-07T21:24:50.024272745Z caller=helm.go:69 component=helm version=v3 info="checking 5 resources for changes" targetNamespace=microservices-demo release=frontend

ts=2020-10-07T21:24:50.032615373Z caller=helm.go:69 component=helm version=v3 info="Looks like there are no changes for Service \"frontend\"" targetNamespace=microservices-demo release=frontend

ts=2020-10-07T21:24:50.056403503Z caller=helm.go:69 component=helm version=v3 info="Created a new Deployment called \"frontend-hipster\" in microservices-demo\n" targetNamespace=microservices-demo release=frontend

ts=2020-10-07T21:24:50.069796385Z caller=helm.go:69 component=helm version=v3 info="Looks like there are no changes for Gateway \"frontend-gateway\"" targetNamespace=microservices-demo release=frontend

ts=2020-10-07T21:24:50.103326263Z caller=helm.go:69 component=helm version=v3 info="Looks like there are no changes for ServiceMonitor \"frontend\"" targetNamespace=microservices-demo release=frontend

ts=2020-10-07T21:24:50.123198029Z caller=helm.go:69 component=helm version=v3 info="Looks like there are no changes for VirtualService \"frontend\"" targetNamespace=microservices-demo release=frontend

ts=2020-10-07T21:24:50.126636377Z caller=helm.go:69 component=helm version=v3 info="Deleting \"frontend\" in microservices-demo..." targetNamespace=microservices-demo release=frontend

ts=2020-10-07T21:24:50.155886101Z caller=helm.go:69 component=helm version=v3 info="updating status for upgraded release for frontend" targetNamespace=microservices-demo release=frontend
 ```

## Самостоятельное задание

- Добавил по образу и подобию первого манифесты HelmRelease для всех микросервисов входящих в состав HipsterShop
- Проверил - все развернулись хорошо, единственно loadgenerator хотел себе url магазина корректный, я не дал, нагрузка его мне не нужна, вообще удалил его из кластера.
```
helm delete loadgenerator -n microservices-demo
kubectl delete helmrelease loadgenerator -n microservices-demo
```

# Canary deployments с Flagger и Istio

## Установка Istio
 - в начале задания мы включили istio в настройках кластера, а щас еще будем его устанавливать, странно.
```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.6.12 TARGET_ARCH=x86_64 sh -
cd istio-1.6.12
sudo mv bin/istioctl /usr/bin/ 
istioctl install --set profile=remote 
```
 - при установке были проблемы, решил не обращать на это внимания - видимо конфликты с уже установленным в кластере 
 - в итоге к уже установленному добавилось только istiod
```
$ kubectl -n istio-system get deploy 
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
istio-citadel            1/1     1            1           5d19h
istio-galley             1/1     1            1           5d19h
istio-ingressgateway     1/1     1            1           5d19h
istio-pilot              1/1     1            1           5d19h
istio-policy             1/1     1            1           5d19h
istio-sidecar-injector   1/1     1            1           5d19h
istio-telemetry          1/1     1            1           5d19h
istiod                   1/1     1            1           11m
promsd                   1/1     1            1           5d19h
```
## Установка Flagger

 - Добавление helm-репозитория flagger:
```
helm repo add flagger https://flagger.app
```

 - Установка CRD для Flagger:
```
kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml
```

 - Установка flagger с указанием использовать Istio:

```
helm upgrade --install flagger flagger/flagger \
--namespace=istio-system \
--set crd.create=false \
--set meshProvider=istio \
--set metricsServer=http://prometheus:9090
```

## Istio | Sidecar Injection

 - взял текущий конфиг ns, добавил туда label istio-injection: enabled

```
kubectl get ns microservices-demo -o yaml
kubectl apply -f kubernetes-gitops/NS-microservices-demo.yaml
```
 - пр  это конфиг неймспейса постоянно возвращадся в исходное состояние. Если поразмыслить - оно и понятно, кластер не подчиняется!) его состояние контроллирует flux, поэтому правильный способ изменить конфиг ns  - в коде deploy/namespaces/ns.yaml, flux сам применит эти измения
 - пушим измения, смотрим лэйблы у ns
```
kubectl get ns microservices-demo --show-labels
```
 - Самый простой способ добавить sidecar контейнер в уже запущенные pod - удалить их:
```
kubectl delete pods --all -n microservices-demo
```

## Доступ к frontend

 - Создайте директорию deploy/istio и поместите в нее следующие
манифесты:
frontend-vs.yaml
frontend-gw.yaml

```
$ kubectl get gateway -n microservices-demo
NAME               AGE
frontend           19s
frontend-gateway   13m
```

 - Для доступа снаружи нам понадобится EXTERNAL-IP сервиса istioingressgateway

 ```
$ kubectl get svc istio-ingressgateway -n istio-system 
NAME                   TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)                                                                                                                                      AGE
istio-ingressgateway   LoadBalancer   10.1.4.47    34.71.154.233   15020:31159/TCP,80:32617/TCP,443:31917/TCP,31400:30846/TCP,15029:31119/TCP,15030:32312/TCP,15031:30522/TCP,15032:31162/TCP,15443:31997/TCP   5d19h
 ```

## Istio | Самостоятельное задание

 - заменил файлы virtualService.yaml gateway.yaml в чарте frontend, удалил istio, запушил

## Flagger | Canary

 - добавил в чарт (а точнее изменил имующуйся) canary.yaml (весь конфиг я так понимаю не влез в дз, что тоже очень мило конечно)
 - ресурс canary не появился =(
 - посмотрел вокруг - подумал походу надо версию чарта увеличить в Cart.yaml, увеличил, запушил - ожидаемо не в этом дело.
 - обсмотрел вывод команды
```
fluxctl list-workloads -a --k8s-fwd-ns flux 
```
 - как водится пришлось еще покумекать. Мы переименовывали deployment для frontend, подловили блин. Поправил name в targetRef, запушил.
 - не получилось( решил переименовать deployment  потом чекнул логи helm релиза
```
kubectl describe helmrelease frontend -n microservices-demo
```
  -  а там такая чепухня

```
Warning  FailedReleaseSync  3m43s (x143 over 152m)  helm-operator  synchronization of release 'frontend' in namespace 'microservices-demo' failed: dry-run upgrade failed: dry-run upgrade for comparison failed: rendered manifests contain a new resource that already exists. Unable to continue with update: existing resource conflict: namespace: microservices-demo, name: frontend, existing_kind: networking.istio.io/v1alpha3, Kind=Gateway, new_kind: networking.istio.io/v1alpha3, Kind=Gateway
```
 - удалил ручками старый gateway

```
kubectl delete Gateway frontend -n microservices-demo
```

 - получилось!) 

```
$ kubectl get canary -n microservices-demo 
NAME       STATUS        WEIGHT   LASTTRANSITIONTIME
frontend   Initialized   0        2020-10-11T17:09:22

$ kubectl get pods -n microservices-demo -l app=frontend-primary
NAME                                READY   STATUS    RESTARTS   AGE
frontend-primary-859cf9cf57-bbws5   2/2     Running   0          4m41s
```

 - Попробуем провести релиз (сделал новый тэг, образ запушился)

```
$ kubectl describe canary frontend -n microservices-demo 

  Warning  Synced  65s                flagger  Prometheus query failed: running query failed: request failed: Get "http://prometheus:9090/api/v1/query?query=+sum%28+rate%28+istio_requests_total%7B+reporter%3D%22destination%22%2C+destination_workload_namespace%3D%22microservices-demo%22%2C+destination_workload%3D~%22frontend%22%2C+response_code%21~%225.%2A%22+%7D%5B1m%5D+%29+%29+%2F+sum%28+rate%28+istio_requests_total%7B+reporter%3D%22destination%22%2C+destination_workload_namespace%3D%22microservices-demo%22%2C+destination_workload%3D~%22frontend%22+%7D%5B1m%5D+%29+%29+%2A+100": dial tcp: lookup prometheus on 10.1.0.10:53: no such host
  Warning  Synced  5s                 flagger  Rolling back frontend.microservices-demo failed checks threshold reached 1
  Warning  Synced  5s                 flagger  Canary failed! Scaling down frontend.microservices-demo
```

 - вернул prometheus, раз уж мы его указали как metricsServer для flagger
 - по примеру вывода describe из дз - следующая проблема в том что без нагрузки fragger не может определить успешность релиза, поэтому вернем loadgenerator и учтем что теперь у нас есть ingress (пропишем корректный хост с xip)
 - пришлось удалять кластер, денюжки закончились.

 - ну и понеслась с нуля:
```
gcloud init
gcloud container clusters get-credentials cluster-1
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/helm-v3-dev/deploy/flux-helm-release-crd.yaml
kubectl create namespace flux
helm upgrade --install flux fluxcd/flux -f kubernetes-gitops/flux.values.yaml --namespace flux
helm upgrade --install helm-operator fluxcd/helm-operator -f kubernetes-gitops/helm-operator.values.yaml --namespace flux

fluxctl identity --k8s-fwd-ns flux
#установил полученный ключ в gitlab

istioctl install --set profile=default
# если не включать в настрйоках кластера istio, то установка проходит без ошибок. И надо указывать profile=default а не remote как я сделал прошлый раз - он тут не подходит

kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml
helm upgrade --install flagger flagger/flagger \
--namespace=istio-system \
--set crd.create=false \
--set meshProvider=istio \
--set metricsServer=http://prometheus:9090

kubectl delete pods --all -n microservices-demo 
kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)                                                      AGE
istio-ingressgateway   LoadBalancer   10.8.5.179   35.222.38.154   15021:30135/TCP,80:32506/TCP,443:31792/TCP,15443:30046/TCP   3m34s
# узнали новый ip, пушим его в loadgenerator
```

 - все получилось, привожу вывод:
 
```
$ kubectl get canaries -n microservices-demo 
NAME       STATUS      WEIGHT   LASTTRANSITIONTIME
frontend   Succeeded   0        2020-10-12T21:41:50Z
```


```
$ kubectl describe canary frontend -n microservices-demo                              
Name:         frontend
Namespace:    microservices-demo
Labels:       <none>
Annotations:  helm.fluxcd.io/antecedent: microservices-demo:helmrelease/frontend
API Version:  flagger.app/v1beta1
Kind:         Canary
Metadata:
  Creation Timestamp:  2020-10-12T20:53:11Z
  Generation:          1
  Resource Version:    54025
  Self Link:           /apis/flagger.app/v1beta1/namespaces/microservices-demo/canaries/frontend
  UID:                 ccdc3c53-8285-4cff-8c14-8ae19fd24c79
Spec:
  Analysis:
    Interval:    1m
    Max Weight:  30
    Metrics:
      Interval:   1m
      Name:       request-success-rate
      Threshold:  99
    Step Weight:  10
    Threshold:    1
  Provider:       istio
  Service:
    Gateways:
      frontend
    Hosts:
      *
    Port:  80
    Retries:
      Attempts:         3
      Per Try Timeout:  1s
      Retry On:         gateway-error,connect-failure,refused-stream
    Target Port:        8080
    Traffic Policy:
      Tls:
        Mode:  DISABLE
  Target Ref:
    API Version:  apps/v1
    Kind:         Deployment
    Name:         frontend
Status:
  Canary Weight:  0
  Conditions:
    Last Transition Time:  2020-10-12T21:41:50Z
    Last Update Time:      2020-10-12T21:41:50Z
    Message:               Canary analysis completed successfully, promotion finished.
    Reason:                Succeeded
    Status:                True
    Type:                  Promoted
  Failed Checks:           0
  Iterations:              0
  Last Applied Spec:       66c4d6964
  Last Promoted Spec:      66c4d6964
  Last Transition Time:    2020-10-12T21:41:50Z
  Phase:                   Succeeded
  Tracked Configs:
Events:
  Type     Reason  Age                From     Message
  ----     ------  ----               ----     -------
  Warning  Synced  50m                flagger  frontend-primary.microservices-demo not ready: waiting for rollout to finish: observed deployment generation less then desired generation
  Warning  Synced  49m (x2 over 50m)  flagger  Error checking metric providers: prometheus not avaiable: running query failed: request failed: Get "http://prometheus:9090/api/v1/query?query=vector%281%29": dial tcp: lookup prometheus on 10.8.0.10:53: no such host
  Normal   Synced  49m                flagger  Initialization done! frontend.microservices-demo
  Normal   Synced  30m                flagger  New revision detected! Scaling up frontend.microservices-demo
  Normal   Synced  29m                flagger  Starting canary analysis for frontend.microservices-demo
  Normal   Synced  29m                flagger  Advance frontend.microservices-demo canary weight 10
  Warning  Synced  28m                flagger  Prometheus query failed: running query failed: request failed: Get "http://prometheus-kube-prometheus-prometheus:9090/api/v1/query?query=+sum%28+rate%28+istio_requests_total%7B+reporter%3D%22destination%22%2C+destination_workload_namespace%3D%22microservices-demo%22%2C+destination_workload%3D~%22frontend%22%2C+response_code%21~%225.%2A%22+%7D%5B1m%5D+%29+%29+%2F+sum%28+rate%28+istio_requests_total%7B+reporter%3D%22destination%22%2C+destination_workload_namespace%3D%22microservices-demo%22%2C+destination_workload%3D~%22frontend%22+%7D%5B1m%5D+%29+%29+%2A+100": dial tcp: lookup prometheus-kube-prometheus-prometheus on 10.8.0.10:53: no such host
  Warning  Synced  27m                flagger  Rolling back frontend.microservices-demo failed checks threshold reached 1
  Warning  Synced  27m                flagger  Canary failed! Scaling down frontend.microservices-demo
  Normal   Synced  8m12s              flagger  New revision detected! Scaling up frontend.microservices-demo
  Normal   Synced  7m12s              flagger  Starting canary analysis for frontend.microservices-demo
  Normal   Synced  7m12s              flagger  Advance frontend.microservices-demo canary weight 10
  Normal   Synced  6m12s              flagger  Advance frontend.microservices-demo canary weight 20
  Normal   Synced  5m12s              flagger  Advance frontend.microservices-demo canary weight 30
  Normal   Synced  4m12s              flagger  Copying frontend.microservices-demo template spec to frontend-primary.microservices-demo
  Normal   Synced  3m12s              flagger  Routing all traffic to primary
  Normal   Synced  2m12s              flagger  Promotion completed! Scaling down frontend.microservices-demo
```


