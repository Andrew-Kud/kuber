# 1:

1.
Проверка статуса:
```
microk8s status --wait-ready
```

Провека ноды:
```
microk8s kubectl get nodes
```

Проверка всех подов в кластере:
```
microk8s kubectl get pods -A
```

2.
Создание namespace:
```
microk8s kubectl create namespace lesson1
```
проверь сразу:
```
microk8s kubectl get namespaces
```
должен быть `lesson1  Active  8s`

Ресурсы буду создавать в `lesson1`.

3.
Создание deployment:
`deployment.yaml`

Метки должны совпадать. Благодаря этому Deployment понимает, какие Pod относятся к нему:
```
selector:
  matchLabels:
    app: demo-web
```
```
template:
  metadata:
    labels:
      app: demo-web
```

`containerPort: 80` - описывает порт приложения для Кубера, но сам по себе не открывает порт, не изменяет настройки nginx и не резервирует его.

Пространство имён сначала создаётся, потом задаётся:
```
metadata:
  name: demo-web
  namespace: lesson1
```

4.
Запуск deployment:
```
microk8s kubectl apply -f deployment.yaml
```
`deployment.apps/demo-web created`

Проверка:
```
microk8s kubectl -n lesson1 get deployments
```
```
microk8s kubectl -n lesson1 get pods
```

# 1-1

5.
Поиск причины `Error`:
```
microk8s kubectl -n lesson1 describe pod demo-web-698ff69777-ccgmp
```
```
microk8s kubectl -n lesson1 logs demo-web-698ff69777-ccgmp -c nginx
```
```
microk8s kubectl -n lesson1 logs demo-web-698ff69777-ccgmp -c multitool --previous
```

Логи в реалтайме:
```
microk8s kubectl -n lesson1 logs demo-web-698ff69777-ccgmp -c nginx -f
```

Логи сразу двух контейнеров:
```
microk8s kubectl -n lesson1 logs demo-web-698ff69777-ccgmp --all-containers=true --prefix=true
```

Проблема - оба контейнера используют 80 tcp порт.
Два приложения не могут одновременно слушать один и тот же IP:порт при обычных настройках

# 1-2

6.
в `deployment.yml` заменить настройки контейнера для `multitool` для портов:
```
        - name: multitool
          image: wbitt/network-multitool
          env:
            - name: HTTP_PORT
              value: "8080"
            - name: HTTPS_PORT
              value: "8443"
          ports:
            - containerPort: 8080
              name: multitool-http
```

7.
Применить изменения:
```
microk8s kubectl apply -f deployment.yaml
```
`deployment.apps/demo-web configured`

Проверка:
```
microk8s kubectl -n lesson1 rollout status deployment/demo-web
```
`deployment "demo-web" successfully rolled out`

```
microk8s kubectl -n lesson1 get pods
```

`curl` из `miltitool` в `nginx`:
```
microk8s kubectl -n lesson1 exec deployment/demo-web -c multitool -- curl -sSI http://localhost:80
```

`curl` из `nginx` в `multitool`:
```
microk8s kubectl -n lesson1 exec deployment/demo-web -c multitool -- curl -sS http://localhost:8080
```

# 1-3

8.
Масштабирование до 2-ух реплик:

Проверка до масштабирования:
```
microk8s kubectl -n lesson1 get pods -l app=demo-web -o wide
```

Увличение реплик до 2-ух:
```
microk8s kubectl -n lesson1 scale deployment/demo-web --replicas=2
```
`deployment.apps/demo-web scaled`

```
microk8s kubectl -n lesson1 rollout status deployment/demo-web
```
`deployment "demo-web" successfully rolled out`

Проверка после масштабирования:
```
microk8s kubectl -n lesson1 get pods -l app=demo-web -o wide
```
```
microk8s kubectl -n lesson1 get deployment demo-web
```

команда scale изменила состояние в кластере, но в файле deployment.yaml осталось replicas: 1. Чтобы файл соответствовал фактическому состоянию, нужно поменять в `.yaml` файле `replicas: 1` -> `replicas: 2`. И снова выполнить `apply`, в противном случае повторное использование старого конфигу уменьшит число реплик до 1.


# 1-4

9.
Создание service.

ip pod не являются постоянными. Если pod будет пересоздан, его ip изменится. service это решает.

нужно создать `service.yaml`.

`type: ClusterIP` - service доступен внутри кластера.
`selector.app`    - ищет под с меткой `app: demo-web`.
`port`            - порт, по которому клиент обращается к service.
`targetPort`      - Порт приложения внутри выбранного под,

10.
Применить:
```
microk8s kubectl apply -f service.yaml
```
`service/demo-web created`

Проверить:
```
microk8s kubectl -n lesson1 get svc
```
```
microk8s kubectl -n lesson1 get endpointslices -l kubernetes.io/service-name=demo-web -o yaml
```

endpoints:
  - addresses:
    - показывает ip упрвляемых подов.

# 1-5

11.
Создание отдельного пода для проверки.

Нужно убедиться, что `dns` - `enabled`:
```
microk8s status
```

Создание нового пода:
```
microk8s kubectl -n lesson1 run curl-client --image=wbitt/network-multitool --restart=Never
```
`pod/curl-client created`

`--restart=Never` - используется при создании самостоятельного пода.

проверяем под:
```
microk8s kubectl -n lesson1 get pods
```

# 1-6

12.
Проверка через `curl`:
```
microk8s kubectl -n lesson1 exec -it curl-client -- /bin/sh
```

проверка `dns`:
```
nslookup demo-web
```

`curl` к `nginx`:
```
curl -I http://demo-web:80
```

`curl` к `mltitool`:
```
curl http://demo-web:8080
```

чек полного dns имени сервиса:
```
nslookup demo-web.lesson1.svc.cluster.local
```

выход:
```
exit
```

# 1-7

13.
Мне показалось недостаточно полагаться на `curl` в качестве единичной проверки. Это не доказывает, что доступны обе реплики. Можно через цикл обратиться к каждой реплики напрямую через `curl-client`:

```
for ip in $(microk8s kubectl -n lesson1 get pods -l app=demo-web -o jsonpath='{.items[*].status.podIP}'); do

  echo "Testing Pod IP: $ip"

  microk8s kubectl -n lesson1 exec curl-client -- curl -sS -o /dev/null -w 'nginx HTTP %{http_code}\n' "http://$ip:80"

  microk8s kubectl -n lesson1 exec curl-client -- curl -sS "http://$ip:8080"
done
```

- Получает ip всех pod с меткой `app=demo-we`b и проверяет оба http сервера на каждом адресе.

---

# 2:

1.
`service` указывает на этот же под, нельзя заставлять init контейнер ждать http ответа от nginx через service. Получится циклическая зависимость: nginx ждёт initконтейнер, а init контейнер ждёт nginx. Поэтому проверяю существование dns записи обычного clusterip service, которая появляется независимо от готовности nginx.

Подготовка.
Создание нового namespace:
```
microk8s kubectl create namespace lesson2
```
`namespace/lesson2 created`

Создание директории для нового манифеста:
```
mkdir ./task2
cd ./task2
```

Проверь что в новом namespace нет ресурсов:
```
microk8s kubectl -n lesson2 get pods,deploy,svc
```
`No resources found in lesson2 namespace.`

2.
Создание deplyment с init контейнером:
task2/deployment.yaml

`until` - выполняет цикл, пока команда возвращает ненулевой код завершения.
Когда `nslookup` находит service, цикл завершается.
Следующая команда `echo` выполняется, и `shell` заканчивает работу с кодом 0.

Формат coredns:
`nginx-service.lesson2.svc.cluster.local` состоит из имени service, mamespace и dns кластера.

3.
ЗАпуск без service:
```
microk8s kubectl apply -f deployment.yaml
```

проверка:
```
microk8s kubectl -n lesson2 get pods
```

# 2-1

4.
`Init:0/1` - под находится в процессе инициализации: один init контейнер предусмотрен, но ни один не завершился успешно. по этому `READY 0/1`.

Проверка `busybox`:
```
microk8s kubectl -n lesson2 logs deployment/nginx-init-demo -c wait-for-service
```

Логи с `nginx`:
```
microk8s kubectl -n lesson2 logs deployment/nginx-init-demo -c nginx
```

# 2-2

`nginx` ещё не запущен, поскольку `service` отсутствует.

5.
Создание service:
task2/service.yaml

`metadata.name: nginx-service` имя которое ищет busybox через dns.
`selector.app: nginx-init-demo` метка под, к которому service будет направлять соединения.

6.
Применить:
```
microk8s kubectl apply -f service.yaml
```
`service/nginx-service created`

Проверить:
```
microk8s kubectl -n lesson2 get svc
```
```
microk8s kubectl -n lesson2 get pods
```

# 2-3

```
microk8s kubectl -n lesson2 logs deployment/nginx-init-demo -c wait-for-service
```

# 2-4

busybox обнаружил service через dns -> цикл until завершился -Ю init контейнер закончил работу с кодом 0 -> кубер запустил основной контейнер nginx.

7.
Доп.проверка - доказательство:
```
microk8s kubectl -n lesson2 get pods
```
```
microk8s kubectl -n lesson2 describe pod nginx-init-demo-cbd84b774-ts6ch
```

# 2-5

8.
Проверка http доступность nginx:
```
microk8s kubectl -n lesson1 exec curl-client -- curl -I http://nginx-service.lesson2.svc.cluster.local
```

# 2-6

---

Доп.наблюдение изменения состояний:
Открой вотую ssh сессию и введи в ней наблюдение в реалтайме:
```
microk8s kubectl -n lesson2 get pods -w
```

В первой ssh окне-сессии:

Удали только service:
```
microk8s kubectl -n lesson2 delete service nginx-service
```
- уже запущенный nginx при этом не остановится, initконтейнер не выполняет проверку постоянно.

# 3-1

Пересоздай под:
```
microk8s kubectl -n lesson2 rollout restart deployment/nginx-init-demo
```

# 3-2

Снова сосздай service:
```
microk8s kubectl apply -f service.yaml
```
- увидишь переход с промежуточными состояниями в `STATUS`.

# 3-3

---

Удаление:
- если удалить только pod, контроллер deployment автоматически создаст новые, поскольку k8s поддерживает заданное количество реплик.

1.
Удалить все ресурсы вместе с namespace:
```
microk8s kubectl delete namespace lesson1 lesson2
```
- если внутри есть PVC, удаление namespace может привести и к удалению связанных томов и данных.

Проверка:
```
microk8s kubectl get namespaces
```

2.
Удалить ресурсы, но сохранить namespace:
- для каждого namespace отдельно.
```
microk8s kubectl -n lesson1 delete deployment --all
```
```
microk8s kubectl -n lesson1 delete pod --all
```
```
microk8s kubectl -n lesson1 delete service --all
```
- сначала удаляй deployment, чтобы он не создавал pod повторно.

Проверка:
```
microk8s kubectl -n lesson1 get all
```

3.
Удалить всё из сохранённых `.yaml` (декларативное удаление):
```
microk8s kubectl delete -f ~/k8s-lab/task2/
```
- удаляет объекты, описанные в `.yaml` файлах, но не затронет ресурсы, которые ты создал вручную и не записал в эти файлы.


---
---
