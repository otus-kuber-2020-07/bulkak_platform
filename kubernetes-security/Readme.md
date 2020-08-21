#task-01

 - создал SA bob
 - привязал к bob дефолтную роль admin
 - создал SA dave
  - проверил:
```
kubectl auth can-i get pods --as system:serviceaccount:default:bob
yes
kubectl auth can-i get pods --as system:serviceaccount:default:dave
no
```

#task-02

 - создал Namespace prometheus
 - создал sa carol
 - создал ClusterRole pod-reader с правами на  get, list, watch в отношении Pods
 - Привязал роль pod-reader к группе sa с namespace=prometheus: system:serviceaccounts:prometheus
 - проверил:
```
kubectl auth can-i list pods --as system:serviceaccount:prometheus:carol
yes
kubectl auth can-i edit pods --as system:serviceaccount:prometheus:carol
no
```

#task-03
 - создал Namespace dev
 - создал sa jane
 - привязал ей ClusterRole admin, но с указанием namespace в subjects
 - создал sa ken
 - привязал ему ClusterRole view, но с указанием namespace в subjects
 - проверил
```
kubectl auth can-i delete pods --as system:serviceaccount:dev:jane
no
kubectl auth can-i delete pods --as system:serviceaccount:dev:jane --namespace dev
yes
kubectl auth can-i list pods --as system:serviceaccount:dev:ken --namespace dev
yes
kubectl auth can-i delete pods --as system:serviceaccount:dev:ken --namespace dev
no
kubectl auth can-i list pods --as system:serviceaccount:dev:ken
no
```