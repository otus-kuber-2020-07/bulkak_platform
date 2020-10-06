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

  - запушил изменения в сервис frontend, создал в гитлаюе тэг, запустилась сборка и пуш тэгированных образов в docker-hub, красота.
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
 - логи горят накосячил в конфиге deployment. подправил
 - все равно ничего!



