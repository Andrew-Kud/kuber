# 1:

multitool - содержит диагностические сетевые утилиты. Постоянно размещать такой контейнер рядом с приложением в проде не стоит.

ClusterIP предоставляет доступ внутри кластера, но не ограничивает доступ между Pod. Для сетевой изоляции в проде применяй NetworkPolicy.

Контейнеры внутри одного Pod используют общий сетевой namespace — один IP-адрес и общее пространство TCP/UDP.

---

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
Должен увидеть только служебные поды. Они часть самого кластера и нужны для его работы.
- calico - сеть между подами.
- coredns - dns внутри кластера.
- kubelite - управляющие компоненты.

# 1-1

2.
Включить dns:
```
microk8s enable dns
```

Создать namespace:
```
microk8s kubectl create namespace network-lab
```

Создадим deployment.yaml.
Старался приблизиться к продакш best pratctices и примеры брал из:
https://github.com/Fatalito/k8s-service-blueprint#1

Вместо жёсткой привязки Service к портам 80 и 8080 присваиваю имена nginx-http и multitool-http.
Service сможет ссылаться на эти имена. Если порт приложения изменится, достаточно согласованно изменить его в конфигурации Pod.
Версии образов указаны явно, без latest. Это делает поведение развёртывания более предсказуемым.

3.
Применяю деплой:
```
microk8s kubectl -n network-lab apply -f deployment.yaml
```

Ожидание развёртываня:
```
microk8s kubectl -n network-lab rollout status deployment/web-app
```

Проверка:
```
microk8s kubectl -n network-lab get pods -o wide
```

# 1-2

4.
Создаю service.yaml.
Service не привязан к конкретному dployment. Он использует метки Pod и номера портов (а так же имена портов).

Применяю:
```
microk8s kubectl -n network-lab apply -f service.yaml
```

Проверяю:
```
microk8s kubectl -n network-lab get svc app-service
```
полезно узнать какие pod обнаружены сервисом:
```
microk8s kubectl -n network-lab get endpointslices -l kubernetes.io/service-name=app-service
```

# 1-3

5.
Создам отдельный под для тестирования client-pod.yaml.
Не стал изменять порт, потому что внутри работает только один Pod контейнер и конфликтовать ему не с кем.

Применяю:
```
microk8s kubectl -n network-lab apply -f client-pod.yaml
```

Ожидаю готовность:
```
microk8s kubectl -n network-lab wait --for=condition=Ready pod/network-client --timeout=120s
```

Проверка доступности через curl:

nginx — порт 9001
```
microk8s kubectl -n network-lab exec network-client -- curl -fsS http://app-service:9001/
```

multitool — порт 9002
```
microk8s kubectl -n network-lab exec network-client -- curl -fsS http://app-service:9002/
```

# 1-4

6.
Проверка dns:
```
microk8s kubectl -n network-lab exec network-client -- nslookup app-service.network-lab.svc.cluster.local
```
т.к стандартный домен: `app-service.network-lab.svc.cluster.local`

nginx:
```
microk8s kubectl -n network-lab exec network-client -- curl -fsS http://app-service.network-lab.svc.cluster.local:9001/
```

multitool:
```
microk8s kubectl -n network-lab exec network-client -- curl -fsS http://app-service.network-lab.svc.cluster.local:9002/
```

---

# 2:

1.
Создай task2/service-nodeport.yaml
- Выбрали 30080 прт чтобы адрес оставался статическим.

чек деплоя из задания 1:
```
microk8s kubectl -n network-lab get deployment web-app
```

применяю новый манифест:
```
microk8s kubectl -n network-lab apply -f service-nodeport.yaml
```

Проверяю:
```
microk8s kubectl -n network-lab get svc
```
```
microk8s kubectl -n network-lab get endpointslices -l kubernetes.io/service-name=web-app-nodeport
```
- Service может успешно создаться даже тогда, когда его selector не совпадает ни с одним Pod. В этом случае запросы к приложению не будут проходит.

# 2-1

2.
Проверка доступности:
```
ip -br -4 addr
```
```
microk8s kubectl get nodes -o wide
```
```
curl -i http://192.168.10.26:30080/
```
```
http://192.168.10.26:30080/
```

Локальный пк -> VM:30080 -> Service -> nginx:80

# 2-2

В production не используй NodePort как публичный вход для веб-приложения. Перед приложением ставят Ingress Controller, а доступ к сетевым портам узлов ограничивается.

3.
Удалить ресурсы в namespace:
```
microk8s kubectl delete namespace network-lab
```

проверка:
```
microk8s kubectl get namespace network-lab
```


---
---
