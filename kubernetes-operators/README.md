# Что сделано

 - Создал первые CRD и CR, добавил валидацию. На моей версии кубера не хотела
  выполнятся валидация из приложенных
  к ДЗ gist. Подправил валидацию на apiVersion: apiextensions.k8s.io/v1
	(https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/),
	 получил ошибку валидации.
 - Добавил описание обязательных полей для spec
 - создал mysqloperator.py, заполнил папку templates, запустил
 ```
 kopf run mysql-operator.py
 ```
 - **Вопрос: почему объект создался, хотя мы создали CR, до того, как
запустили контроллер?**  

потому что контроллер спросил текущее состояние у API-сервера, который вернул
те события которые нужны для проверки с желаемым состоянием - событие создания.

 - Дописал mysql-operator.py для назначения всех созданных им объектов дочерними с CR,
 добавил объект, и удалил его
```
$ kubectl delete mysqls.otus.homework mysql-instance
mysql.otus.homework "mysql-instance" deleted
```
```
[2020-09-04 19:40:41,686] kopf.objects         [INFO    ] [default/mysql-instance] Handler 'mysql_on_create' succeeded.
[2020-09-04 19:40:41,686] kopf.objects         [INFO    ] [default/mysql-instance] All handlers succeeded for creation.
[2020-09-04 19:41:07,056] kopf.objects         [INFO    ] [default/mysql-instance] Handler 'delete_object_make_backup' succeeded.
[2020-09-04 19:41:07,056] kopf.objects         [INFO    ] [default/mysql-instance] All handlers succeeded for deletion.
```
 - дописал в контроллер код для бэкапов при удалении и восстановления, попробовал (пришлось отключить плагин minikube https://github.com/kubernetes/minikube/issues/1239)
 ```
minikube addons disable default-storageclass
 ```
 в оcтальном все отработало штатно после удаления ресурса остался pvc бэкапа,
  при создании ресурса данные восстановились оттуда.

- забилдил запушил image контейнера с контроллером в docker hub
- применил деплой с ролями в кластер, проверил работоспособность.
  - **Добавьте в README вывод комманды kubectl get jobs**
  ```
  NAME                         COMPLETIONS   DURATION   AGE
  backup-mysql-instance-job    1/1           3s         37m
  restore-mysql-instance-job   1/1           49s        37m
  ```
 - **Приложетие вывод при запущенном MySQL**
`export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="
{.items[*].metadata.name}")
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otusdatabase`

    ```
  mysql: [Warning] Using a password on the command line interface can be insecure.
   +----+-------------+
   | id | name        |
   +----+-------------+
   |  1 | some data   |
   |  2 | some data-2 |
   +----+-------------+
    ```

##  Задание со ⭐ (1)

 - добавил с crd объект статуса

 ```
 status:
   type: object
   x-kubernetes-preserve-unknown-fields: true
 ```

 - там же указал что туда печатать

 ```
 additionalPrinterColumns:
 \- name: Message
   type: string
   priority: 0
   jsonPath: .status.create_fn.message
   description: As returned from the handler (sometimes).
 ```

 - Из функции возвращаю json вида: {"message": message}, содержимое message
 определяю в коде по косвенным признакам (наличие эксепшена при создании pvc)

 - при первом создании вывод describe:

 ```
 Status:
  mysql_on_create:
    Message:  mysql-instance created without restore-job
 ```

 - после удаления и создания заново:

 ```
 Status:
  mysql_on_create:
    Message:  mysql-instance created with restore-job
 ```

 ##  Задание со ⭐ (2)

 - дописал функцию для запуска при изменении поля пароля. Сделал шаблон Job по смене пароля в mysql. В коде сначала запускаю эту job, потом обновляю данные в deployment, который уже обновляет инстанс mysql.

 ```
 @kopf.on.field('otus.homework', 'v1', 'mysqls', field='spec.password')
 def update_object_change_password(old, new, status, namespace, body, logger, **kwargs):
     deployment_name = status['mysql_on_create']['deployment-name']
     name = body['metadata']['name']
     image = body['spec']['image']
     database = body['spec']['database']
     storage_size = body['spec']['storage_size']

     delete_success_jobs(name)

     if old is None:
         old = body['spec']['password']

     # Cоздаем change-pass job:
     api = kubernetes.client.BatchV1Api()
     change_pass_job = render_template('change-pass-job.yml.j2', {
         'name': name,
         'new_password': new,
         'password': old,
         'database': database})
     api.create_namespaced_job('default', change_pass_job)
     wait_until_job_end(f"change-pass-{name}-job")

     deployment_patch = render_template('mysql-deployment.yml.j2', {
         'name': name,
         'image': image,
         'password': new,
         'database': database})

     # Создаем mysql Deployment:
     api = kubernetes.client.AppsV1Api()
     api.patch_namespaced_deployment(
         namespace='default',
         name=deployment_name,
         body=deployment_patch,
     )
     return {
         "message": "mysql password changed!",
     }
 ```
