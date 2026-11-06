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

<img width="772" height="85" alt="1-1" src="https://github.com/user-attachments/assets/bc07af5f-a4e6-48a3-850f-86a4ab2bb798" />

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

<img width="982" height="227" alt="1-2" src="https://github.com/user-attachments/assets/421f7640-34e2-482e-b8a8-492f26ccb02d" />

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

<img width="700" height="158" alt="1-3" src="https://github.com/user-attachments/assets/0aab0204-ed1d-4746-9859-308a5014c2a9" />

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

<img width="1135" height="601" alt="1-4" src="https://github.com/user-attachments/assets/a1d71bda-32db-4847-897a-1f0bce070ba8" />

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

<img width="1209" height="761" alt="1-5" src="https://github.com/user-attachments/assets/f59f4e63-095d-4381-9dc9-e6d57e07696c" />

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

<img width="762" height="154" alt="2-1" src="https://github.com/user-attachments/assets/7b03a3d7-5c2d-4184-beee-c92e2e4a2ba0" />

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

<img width="2560" height="861" alt="2-2" src="https://github.com/user-attachments/assets/8a050242-6674-43c6-9cb3-286f1f58bff5" />

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
