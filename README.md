# skytaker_platform
skytaker Platform repository
Домашнее задание #1
Задание 1:

Разберитесь почему все pod в namespace kube-system восстановились после удаления.

Все pods в namespace kube-system кроме сoredns востановились потому что их пересоздал kubelet systemd демон работающий на самом хосте. А pod coredns перезапустился потому что есть deployment с параметром replicas 1, то есть, существует replicaset для coredns который его пересоздает если его нет. Для kube-proxy есть daemonset который запускает его в одном экземпляре на каждой ноде.

Задание 2:

Написать валиадный манифест kubernetes-intro/web-pod.yaml

done

Задание 3:

Написать рабочий манифест kubernetes-intro/frontend-pod-healthy.yaml

done


Домашнее задание #2
## Homework 2. Kubernetes controllers.ReplicaSet, Deployment, DaemonSet

### 2.1 ReplicaSet

Создадим манифест frontend-replicaset.yaml и запустим одну реплику микросервиса frontend.  
Получаем ошибку  

```console
error: error validating "frontend-replicaset.yaml": error validating data: ValidationError(ReplicaSet.spec): missing required field "selector" in io.k8s.api.apps.v1.ReplicaSetSpec; if you choose to ignore these errors, turn validation off with --validate=false
```

Добавим selector в манифест frontend-replicaset.yaml :

```YAML
  selector:
    matchLabels:
      app: frontend
```

Применим манифест и выведем реплики микросервиса frontend : 

```console
$ kubectl get pods -l app=frontend
NAME             READY   STATUS             RESTARTS   AGE
frontend-z2spz   0/1     CrashLoopBackOff   3          74s
```

Error status потому что не указны переменные :

```console
$ kubectl logs frontend-q2spz
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set

```

Добавим переменые из предыдущего урока:

```YAML
    env:
    - name: PORT
      value: "8080"
    - name: PRODUCT_CATALOG_SERVICE_ADDR
      value: "productcatalogservice:3550"
    - name: CURRENCY_SERVICE_ADDR
      value: "currencyservice:7000"
    - name: CART_SERVICE_ADDR
      value: "cartservice:7070"
    - name: RECOMMENDATION_SERVICE_ADDR
      value: "recommendationservice:8080"
    - name: SHIPPING_SERVICE_ADDR
      value: "shippingservice:50051"
    - name: CHECKOUT_SERVICE_ADDR
      value: "checkoutservice:5050"
    - name: AD_SERVICE_ADDR
      value: "adservice:9555"
```

Выведем реплики микросервиса frontend:

```console
NAME             READY   STATUS    RESTARTS   AGE
frontend-l5gcj   1/1     Running   0          15s
```

Увеличим количество реплик сервиса ad-hoc командой:

```console
$ kubectl scale replicaset frontend --replicas=3
replicaset.apps/frontend scaled

$ kubectl get rs frontend
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       31m
```

Проверим, что благодаря контроллеру pod'ы действительно восстанавливаются после их ручного удаления:

```console
$ kubectl delete pods -l app=frontend | kubectl get pods -l app=frontend -w
NAME             READY   STATUS    RESTARTS   AGE
frontend-cbbwx   1/1     Running   0          64s
frontend-qrw7l   1/1     Running   0          64s
frontend-rtc86   1/1     Running   0          64s
frontend-cbbwx   1/1     Terminating   0          64s
frontend-qrw7l   1/1     Terminating   0          64s
frontend-fcrz2   0/1     Pending       0          0s
frontend-fcrz2   0/1     Pending       0          0s
frontend-rtc86   1/1     Terminating   0          64s
frontend-w6hd8   0/1     Pending       0          0s
frontend-b94jq   0/1     Pending       0          0s
frontend-w6hd8   0/1     Pending       0          0s
frontend-fcrz2   0/1     ContainerCreating   0          0s
frontend-b94jq   0/1     Pending             0          0s
frontend-b94jq   0/1     ContainerCreating   0          0s
frontend-w6hd8   0/1     ContainerCreating   0          0s
frontend-rtc86   0/1     Terminating         0          64s
frontend-cbbwx   0/1     Terminating         0          64s
frontend-qrw7l   0/1     Terminating         0          64s
frontend-b94jq   1/1     Running             0          1s
frontend-w6hd8   1/1     Running             0          1s
frontend-fcrz2   1/1     Running             0          1s
```

Повторно применим манифест frontend-replicaset.yaml. Количество реплик вновь уменьшилось до одной :

```console
$ kubectl apply -f frontend-replicaset.yaml
replicaset.apps/frontend configured

$ kubectl get pods -l app=frontend
NAME             READY   STATUS    RESTARTS   AGE
frontend-w6hd8   1/1     Running   0          9m39s
```  

Изменим манифест, чтобы сразу разворачивались три реплики сервиса :

```YAML
  replicas: 3
```

Вновь применим его. Поднялись три реплики сервиса :

```console
$ kubectl apply -f frontend-replicaset.yaml
replicaset.apps/frontend configured

$ kubectl get pods -l app=frontend
NAME             READY   STATUS    RESTARTS   AGE
frontend-5xzfk   1/1     Running   0          9m24s
frontend-w6hd8   1/1     Running   0          32m
frontend-w6ssf   1/1     Running   0          9m24s
```

### 2.2 Обновление ReplicaSet

Перетегируем старый образ, добавим его в манифест и применим :

```console
$ kubectl apply -f frontend-replicaset.yaml | kubectl get pods -l app=frontend -w
NAME             READY   STATUS    RESTARTS   AGE
frontend-5xzfk   1/1     Running   0          24m
frontend-w6hd8   1/1     Running   0          47m
frontend-w6ssf   1/1     Running   0          24m
```

Проверим образ, указанный в ReplicaSet:

```console
$ kubectl get replicaset frontend -o=jsonpath='{.spec.template.spec.containers[0].image}'
devopscourses/hipster-shop:v0.0.2
```

И образ из которого сейчас запущены pod:

```console
$ kubectl get pods -l app=frontend -o=jsonpath='{.items[0:3].spec.containers[0].image}'
devopscourses/hipster-frontend:v0.0.1 devopscourses/hipster-frontend:v0.0.1 devopscourses/hipster-frontend:v0.0.1
```

Удалим все запущеные pod, проверим из какого образа перезапустились :

```console
$ kubectl get pods -l app=frontend -o=jsonpath='{.items[0:3].spec.containers[0].image}'
devopscourses/hipster-frontend:v0.0.2 devopscourses/hipster-frontend:v0.0.2 devopscourses/hipster-frontend:v0.0.2
```

Как показали примеры выше, ReplicaSet проверяет только количество запущеных pod.  
ReplicaSet не умеет рестартовать pod.    

### 2.3 Deployment

Cоберем 2 образa paymentservice с тегамиv0.0.1 и v0.0.2.  
Создадим манифест paymentservice-replicaset.yaml с тремя репликами, разворачивающими из образа версии v0.0.1:  

```console
$ kubectl get pods -l app=paymentservice
NAME                   READY   STATUS    RESTARTS   AGE
paymentservice-4ghd5   1/1     Running   0          3m33s
paymentservice-8xlw6   1/1     Running   0          3m33s
paymentservice-tcwgq   1/1     Running   0          3m33s

$ kubectl get pods -l app=paymentservice -o=jsonpath='{.items[0:3].spec.containers[0].image}'
newbradburyfan/paymentservice:v0.0.1 devopscourses/paymentservice:v0.0.1 devopscourses/paymentservice:v0.0.1
```
Создадим манифест paymentservice-deployment.yaml с kind Deployment вместо Replicaset. 
Применим манифест,  предварительно удалив существующую реплику :

```console
$ kubectl delete rs paymentservice
replicaset.apps "paymentservice" deleted

$ kubectl get pods -l app=paymentservice
No resources found in default namespace.

$ kubectl apply -f paymentservice-deployment.yaml
deployment.apps/paymentservice created

$ kubectl get pods -l app=paymentservice
NAME                             READY   STATUS    RESTARTS   AGE
paymentservice-cb846dcc7-7hpvx   1/1     Running   0          58s
paymentservice-cb846dcc7-8f7ps   1/1     Running   0          58s
paymentservice-cb846dcc7-cwh7j   1/1     Running   0          58s
```

Также, помимо 3 pod у нас появился Deployment и новый ReplicaSet 

```console
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
paymentservice   3/3     3            3           2m7s
$ kubectl get rs
NAME                       DESIRED   CURRENT   READY   AGE
paymentservice-cb846dcc7   3         3         3       2m13s
```

### 2.4 Обновление Deployment

Обновим Deployment на версию образа v0.0.2 и применим манифест.  
Все новые pod развернуты из образа v0.0.2: 

```console
$ kubectl get pods -l app=paymentservice -o=jsonpath='{.items[0:3].spec.containers[0].image}'
devopscourses/paymentservice:v0.0.2 devopscourses/paymentservice:v0.0.2 devopscourses/paymentservice:v0.0.2
```

Также создано два ReplicaSet: 

```console
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
paymentservice-75f846bb64   3         3         3       7m44s
paymentservice-cb846dcc7    0         0         0       15m

```
Посмотрим историю версий нашего Deployment:  

```console
$ kubectl rollout history deployment paymentservice
deployment.apps/paymentservice 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

### 2.5 Deployment | Rollback

Представим, что обновление прошло неудачно и сделаем откат:

```console
$ kubectl rollout undo deployment paymentservice --to-revision=1 | kubectl get rs -l app=paymentservice -w
NAME                        DESIRED   CURRENT   READY   AGE
paymentservice-75f846bb64   0         0         0       14m
paymentservice-cb846dcc7    3         3         3       18m
paymentservice-75f846bb64   0         0         0       14m
paymentservice-75f846bb64   1         0         0       14m


```
Podы вернулись на первую версию образа :

```console
$ kubectl get pods -l app=paymentservice -o=jsonpath='{.items[0:3].spec.containers[0].image}'
devopscourses/paymentservice:v0.0.1 devopscourses/paymentservice:v0.0.1 devopscourses/paymentservice:v0.0.1
```

### 2.6 Deployment | Задание со⭐

Добавим стратегию развертывания в манифест paymentservice-deployment-bg.yaml :

```YAML
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 100%
      maxUnavailable: 0
```

Как видим сначала создаются три новых контейнера, а потом удаляются старые :

```console
$ kubectl apply -f paymentservice-deployment-bg.yaml | kubectl get pods -l app=paymentservice -w
NAME                             READY   STATUS    RESTARTS   AGE
paymentservice-cb846dcc7-d698f   1/1     Running   0          33m
paymentservice-cb846dcc7-rvkh8   1/1     Running   0          33m
paymentservice-cb846dcc7-vwjw4   1/1     Running   0          33m
paymentservice-75f846bb64-q9dr9   0/1     Pending   0          0s
paymentservice-75f846bb64-psrxv   0/1     Pending   0          0s
paymentservice-75f846bb64-q9dr9   0/1     Pending   0          0s
paymentservice-75f846bb64-jqcvw   0/1     Pending   0          0s
paymentservice-75f846bb64-psrxv   0/1     Pending   0          0s
paymentservice-75f846bb64-jqcvw   0/1     Pending   0          0s
paymentservice-75f846bb64-q9dr9   0/1     ContainerCreating   0          0s
paymentservice-75f846bb64-psrxv   0/1     ContainerCreating   0          0s
paymentservice-75f846bb64-jqcvw   0/1     ContainerCreating   0          0s
paymentservice-75f846bb64-jqcvw   1/1     Running             0          0s
paymentservice-cb846dcc7-rvkh8    1/1     Terminating         0          33m
paymentservice-75f846bb64-psrxv   1/1     Running             0          0s
paymentservice-75f846bb64-q9dr9   1/1     Running             0          0s
paymentservice-cb846dcc7-d698f    1/1     Terminating         0          33m
paymentservice-cb846dcc7-vwjw4    1/1     Terminating         0          33m
paymentservice-cb846dcc7-d698f    0/1     Terminating         0          33m
paymentservice-cb846dcc7-vwjw4    0/1     Terminating         0          33m
```

Реализуем другую стратегию развертывания в манифесте paymentservice-deployment-reverse.yaml :

```YAML
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

По этой стратегии удаляется один старый pod и создается один новый:  

```console
$ kubectl apply -f paymentservice-deployment-reverse.yaml | kubectl get pods -l app=paymentservice -w
NAME                              READY   STATUS    RESTARTS   AGE
paymentservice-75f846bb64-jqcvw   1/1     Running   0          14m
paymentservice-75f846bb64-psrxv   1/1     Running   0          14m
paymentservice-75f846bb64-q9dr9   1/1     Running   0          14m
paymentservice-cb846dcc7-kjpzw    0/1     Pending   0          0s
paymentservice-cb846dcc7-kjpzw    0/1     Pending   0          0s
paymentservice-75f846bb64-jqcvw   1/1     Terminating   0          14m
paymentservice-cb846dcc7-kjpzw    0/1     ContainerCreating   0          0s
paymentservice-cb846dcc7-wggvf    0/1     Pending             0          0s
paymentservice-cb846dcc7-wggvf    0/1     Pending             0          0s
paymentservice-cb846dcc7-wggvf    0/1     ContainerCreating   0          0s
paymentservice-cb846dcc7-kjpzw    1/1     Running             0          0s
paymentservice-75f846bb64-psrxv   1/1     Terminating         0          14m
paymentservice-cb846dcc7-hgnzd    0/1     Pending             0          0s
paymentservice-cb846dcc7-hgnzd    0/1     Pending             0          1s
paymentservice-cb846dcc7-hgnzd    0/1     ContainerCreating   0          1s
paymentservice-cb846dcc7-wggvf    1/1     Running             0          1s
paymentservice-75f846bb64-q9dr9   1/1     Terminating         0          14m
paymentservice-cb846dcc7-hgnzd    1/1     Running             0          1s
paymentservice-75f846bb64-jqcvw   0/1     Terminating         0          15m
paymentservice-75f846bb64-q9dr9   0/1     Terminating         0          15m
```

### 2.7 Probes 

Создадим манифест frontend-deployment.yaml и добавим туда описание readinessProbe :

```YAML
          ports:
          - containerPort: 8080
          readinessProbe:
            initialDelaySeconds: 10
            httpGet:
              path: "/_healthz"
              port: 8080
              httpHeaders:
              - name: "Cookie"
                value: "shop_session-id=x-readiness-probe"
```

Види три запущенных pod c указанием readinessProbe :

```console
$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
frontend-8494458dd7-6tbdz   1/1     Running   0          2m46s
frontend-8494458dd7-gq8pf   1/1     Running   0          2m46s
frontend-8494458dd7-tsjt4   1/1     Running   0          2m46s

$ kubectl describe pod frontend-8494458dd7-6tbdz

Readiness:      http-get http://:8080/_healthz delay=10s timeout=1s period=10s #success=1 #failure=3
```

Cымитируем некорректную работу приложения, изменив название readinessProbe :

```console
NAME                        READY   STATUS    RESTARTS   AGE
frontend-55c64dc7d-2zlbw    0/1     Running   0          14m

Warning  Unhealthy  35s (x89 over 15m)  kubelet, kind-worker  Readiness probe failed: HTTP probe failed with statuscode: 404
```

### 2.8 DaemonSet | Задание со ⭐

Скопипастим манифест node-exporter-daemonset.yaml и применим его.  

Затестим как отдаются метрики :

```console
$ kubectl port-forward node-exporter-4pjr8 9100:9100
Forwarding from 127.0.0.1:9100 -> 9100
Forwarding from [::1]:9100 -> 9100

$ curl localhost:9100/metrics
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
# HELP go_goroutines Number of goroutines that currently exist.
```





Домашнее задание #3
## Security

### task01

- Создадим Service Account **bob** и дади ему роль **admin** в рамках всего кластера

01-serviceAccount.yaml:

```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bob
```

02-clusterRoleBinding.yaml:

```yml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: bob
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: bob
  namespace: default
```

- Создадим Service Account **dave** без доступа к кластеру

03-serviceAccount.yaml:

```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dave
```

### task02

- Создадим Namespace prometheus

01-namespace.yaml:

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: prometheus
```

- Создадим Service Account **carol** в этом Namespace

02-serviceAccount.yaml

```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: carol
  namespace: prometheus
```

- Дадим всем Service Account в Namespace prometheus возможность делать **get, list, watch** в отношении Pods всего кластера

03-clusterRole.yaml

```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  verbs: ["get", "list", "watch"]
  resources: ["pods"]
```

04-clusterRoleBinding.yaml

```yml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: prometheus
roleRef:
  kind: ClusterRole
  name: prometheus
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: system:serviceaccounts:prometheus
  apiGroup: rbac.authorization.k8s.io
```

### task03

- Создадим Namespace **dev**

01-namespace.yaml

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

- Создадим Service Account **jane** в Namespace **dev**

02-serviceAccount.yaml

```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jane
  namespace: dev
```

- Дадим **jane** роль **admin** в рамках Namespace **dev**

03-RoleBinding.yaml
```yml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jane
  namespace: dev
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: jane
  namespace: dev
```

- Создади Service Account **ken** в Namespace **dev**

04-serviceAccount.yaml

```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ken
  namespace: dev
```

- Дадим **ken** роль **view** в рамках Namespace **dev**


05-RoleBinding.yaml

```yml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ken
  namespace: dev
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: ken
  namespace: dev
```


