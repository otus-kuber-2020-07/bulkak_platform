# Что делалось

 - установил локальный клиент  - gcloud
 - создал кластер
```
gcloud container clusters create otus-bulkak --num-nodes=1
```
 - установил helm
 - добавил в кластер nginx-ingress 
 - добавил в кластер cert-manager. Изучил документацию по установке, установил что нужен хотябы один Issuer. Сделал Issuer (self-signed), но этого не хватило при создании дальше chartmuseum (к тому же нужно отдельно для каждого ns создавать такой, что не удобно). Вернулся сюда с подсказкой в виде /cluster-issuer: "letsencrypt-production". Сделал ClusterIssuer Let’s Encrypt по туториалу https://cert-manager.io/docs/tutorials/acme/ingress/#step-7-deploy-a-tls-ingress-resource по защите nginx-ingress. Пока варился production сертификат пошел дальше
 - поставил harbor (он пошел уже гораздо легче) - пошарился по values.yaml, принимая во внимание типсы. Накатил chat нужной версии, проследил что сертификат запросился, страницу https://harbor.34.107.103.100.nip.io/ открыл, авторизовался. К этому времени сварился серт для chartmuseum, сертификат хром показывает доверенный - гуд.

https://chartmuseum.34.107.103.100.nip.io/


https://harbor.34.107.103.100.nip.io/

 - запустил chat хипстер-шопа с приложенным template. На облаке не хватило ресурсов (создал кластер из 1 ноды и 1 пула узлов) - 
 пришлось увеличивать. пробросил порт на сервис NodePort командой из gcp на http://127.0.0.1:8080/ - магазин запустился
 - Убрал frontend из хипстер-шопа, запилил отдельный чат с ingress на http://shop.34.107.103.100.nip.io заапгрейдил релизы 
 хипстер-шопа и фронтенд отдельно, шоп по адресу открылся.
 - Параметризовал чат frontend, почитал доку по Helm по части параметризации шаблонов, параметризовал как просят в домашке, 
 объявил frontend зависимостью хипстершопа, подтянул как зависимость, накатил релиз магазина - все работает.
  Поигрался с разными вариантам условий на вывод парметров, установкой переменных через --set
  - Установил helm-secret. Зашифровал файлик с данными секрета, задеплоил через helm шаблон секрета для frontend  - 
  значение подставилось из зашифрованного файла при апгрейде, в секретах кластера данные увидел.
  -  **Предложите способ использования плагина helm-secrets в CI/CD** Думаю можно в гит уже зашифрованные например пароли 
  доступа к бд - они будут расшифровываться уже при деплое плагином - удобно.
  - **Про что необходимо помнить, если используем helm-secrets (например,
как обезопасить себя от коммита файлов с секретами, которые забыл
зашифровать)?**  Не знаю( Варианты кот. приходят на ум: 1) Помнить, и определить flow работы что файлы секретов добавляются 
в гит только после шифрования. 2) Определить правило именования файлов которые должны быть зашифрованы и включить в тест 
сборки кейс на проверку наличия полей шифрования в таких файлах.
 - **Если вы попробовали использовать helm-secrets с KMS - опишите
результаты своей работы** Не пробовал
 - запаковал чаты шопа и фронтенда, предварительно поправив зависимости шопа в соответствии с новым расположением чата 
 frontend. Загрузил в свой Harbor. Удалил релиз шопа из облака. Создал файл repo.sh, выполнил, запустил установку из
  своего репозитория templating/hipster-shop. Проверил  - шоп открывается.
  - переместил деплой и сервисы  paymentservice && shippingservice. накатил релиз без них - добавление в корзину словмалось.
  - поставил kubecfg
```
$ kubecfg version
kubecfg version: v0.16.0
jsonnet version: v0.15.0
client-go version: v0.0.0-master+$Format:%h$
```
 - почитал доку по jsonnet, проследил чего откуда берется в родительскких объектах деплоя и сервиса, подправил
  приложенный вариант конфига services.jsonnet. Задеплоил сервисы с помощью kubecfg, шоп починился.
 - установил kustomize, сделал минимальную конфигурацию, заэплаил в кластер обе кастомизации (сервис recommendationservice).
 - пошел делать PR чтобы прогнать тесты на задания без звездочек.

##  chartmuseum | Задание со ⭐

   - Installation
```
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/linux/amd64/chartmuseum
```
 - configuration
 Создаем секрет с кредами сервис-аккаунта гугла
 ```
 kubectl create secret generic chartmuseum-secret --from-file=credentials.json="my-project-45e35d85a593.json" -n chartmuseum
 ```
 в GC создал хранилище (долго разбирался что нужно чтобы гугл не ругался на права доступа к bucket, оказалось что
  его сначала создать надо), его имя и имя секрета с файлом сервис-аккаунта гугла подпихнул в values.yaml, заново
   накатил релиз chartmuseum (подсмотрел в доке здесь https://github.com/helm/charts/tree/master/stable/chartmuseum#using-with-google-cloud-storage-and-a-google-service-account )
 Запустил проверочку:
 ```
chartmuseum --debug \
  --storage="google" \
  --storage-google-bucket="bulkak-gcs-bucket"

2020-09-04T00:22:40.677+0300	DEBUG	Fetching chart list from storage	{"repo": ""}
2020-09-04T00:22:40.790+0300	DEBUG	No change detected between cache and storage	{"repo": ""}
2020-09-04T00:22:40.790+0300	INFO	Starting ChartMuseum	{"port": 8080}
```
наконец отработала. Дальше попробовал разместить там файл пакета чата. 
```
curl --data-binary "@hipster-shop-0.1.0.tgz" http://localhost:8080/api/charts
```
ответ:
```
{"saved":true}
```

дальше добавил репозиторий в список helm:
```
helm repo add chartmuseum http://localhost:8080
```
и проверил что он нашел загруженный чат:
```
$ helm search repo chartmuseum/ 
NAME                    	CHART VERSION	APP VERSION	DESCRIPTION                
chartmuseum/hipster-shop	0.1.0        	1.16.0     	A Helm chart for Kubernetes
```

##  Используем helmfile | Задание со ⭐

 - Набросал примерно структуру файлов для установки nginx-ingress, cert-manager и harbor, большая часть есть в лекции, 
 от себя добавил усановку CRD и ClusterIssuer для cert-manager в зависимости от environment и добавил для Harbor annotations
  которые добавлял у себя при установке.
 - проверять не стал - время поджимает уже(
