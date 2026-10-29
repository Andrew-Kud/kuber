# 1.

1.
Нужно создать и файл `echoserver-pod.yaml`

Далее запустить его:
```
kubectl apply -f echoserver-pod.yaml
```

Проверка через:
```
kubectl get pods
```
# 1-1

Если под не стартует или ведёт себя неправильно:
```
kubectl describe pod echoserver
```
```
kubectl logs echoserver
```
Тут может выскочить ошибка tls:
`Error from server: Get "https://192.168.10.26:10250/containerLogs/default/echoserver/echoserver": tls: failed to verify certificate: x509: certificate is valid for 192.168.10.27, not 192.168.10.26`

Если появилось что то подобное, нужно решить её, иначе следующий шаг выйдет с ошибкой:

На какой ноде работает под:
```
kubectl get pods -o wide
```

ip адрес всех нод:
```
kubectl get nodes -o wide
```
`INTERNAL-IP`

Узнай какой ip адрес действительно используется:
```
hostname -I
```

Посмотри сертификат, который предъявляется на порту kubelet:
```
openssl s_client -connect 192.168.10.26:10250 </dev/null 2>/dev/null |
openssl x509 -noout -subject -issuer -ext subjectAltName
```

Теперь для `MicroK8s` нужно перевыпустить сертификат под новый ip:

Резевная копия старого сертификата:
```
sudo install -d -m 700 /root/kubelet-cert-fix

sudo cp -a \
  /var/snap/microk8s/current/certs/kubelet.crt \
  /var/snap/microk8s/current/certs/kubelet.key \
  /root/kubelet-cert-fix/
```

Подготовка нового нового сертификата с указанием корректного IP ноды:
```
sudo tee /root/kubelet-cert-fix/san.cnf >/dev/null <<'EOF'
[v3_req]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = DNS:netology,IP:192.168.10.26
EOF
```

Перевыпуск сертификата с сохранением закрытого ключа:
```
CERTDIR=/var/snap/microk8s/current/certs
WORKDIR=/root/kubelet-cert-fix

sudo openssl req -new -sha256 \
  -key "$CERTDIR/kubelet.key" \
  -subj "/CN=system:node:netology/O=system:nodes" \
  -out "$WORKDIR/kubelet.csr"

sudo openssl x509 -req -sha256 \
  -in "$WORKDIR/kubelet.csr" \
  -CA "$CERTDIR/ca.crt" \
  -CAkey "$CERTDIR/ca.key" \
  -CAcreateserial \
  -out "$WORKDIR/kubelet-new.crt" \
  -days 365 \
  -extensions v3_req \
  -extfile "$WORKDIR/san.cnf"
```

Проверка:
```
sudo openssl verify -CAfile "$CERTDIR/ca.crt" "$WORKDIR/kubelet-new.crt"

sudo openssl x509 -in "$WORKDIR/kubelet-new.crt" -noout -subject -ext subjectAltName
```
Нужно получить OK при проверке CA и IP Address:192.168.10.26 в SAN.


Установка сертификата и перезапуск ноды:
```
sudo microk8s stop

sudo install -m 644 "$WORKDIR/kubelet-new.crt" "$CERTDIR/kubelet.crt"

sudo microk8s start
sudo microk8s status --wait-ready
```

Проверка результата:
```
kubectl get nodes
```
```
kubectl logs echoserver
```


2.
port-forward для подключения (в терминале где вводил kubectl):
```
kubectl port-forward pod/echoserver 8888:8080
```
`kubectl port-forward pod/ИМЯ_POD ЛОКАЛЬНЫЙ_ПОРТ:ПОРТ_КОНТЕЙНЕРА`

Но это будет работать только с локальными loopback

# 1-2

А вот как пробросить в сеть:
```
kubectl port-forward --address 127.0.0.1,192.168.10.26 pod/echoserver 8888:8080
```
или так (но так не безопасно):
```
kubectl port-forward --address 0.0.0.0 pod/echoserver 8888:8080
```

# 1-3


3.
Удалить тестовый pod:
```
kubectl delete -f echoserver-pod.yaml
```

Проверка:
```
microk8s kubectl get nodes -o wide
```
```
microk8s kubectl get pods -A -o wide
```
только запущенные поды:
```
microk8s kubectl get pods -A --field-selector=status.phase=Running
```

---

# 2.

1.
Создай манифест для пода `netology-web.yaml`.

Создай манифест для сервиса `netology-svc.yaml`.

labels: app: netology-web в под
и
selector: app: netology-web в сервисе
должны совпадать, Kubernetes автоматически связывает Service с подходящими Pod.

Service не привязывается напрямую к имени Pod! Он выбирает Pod по меткам, поэтому один Service может обслуживать несколько Pod с одинаковыми метками

2.
Запуск пода и сервиса:
```
kubectl apply -f netology-web.yaml
```
```
kubectl apply -f netology-svc.yaml
```

Проверка:
```
kubectl get pods
```
```
kubectl get svc
```

# 2-1

Service получил IP 10.152.183.29. Этот IP принадлежит Service, а не VM.

Проверка Service на нахождение Pod:
```
kubectl describe svc netology-svc
```

# 2-2

Endpoints показывает адреса Pod, доступных через Service.

3.
Подключение:

Локально. Через port-forward обращение к Сервису:
```
kubectl port-forward svc/netology-svc 8888:80
```
svc/netology-svc — обращаемся к Service.
8888 — порт на Ubuntu VM.
80 — порт Service.
8080 — конечный порт Pod, указанный через targetPort в YAML

# 2-3

Находит Pod через Service и создаёт туннель непосредственно к выбранному Pod. Сам сетевой механизм ClusterIP при этом не проверяется.


Сетевой проброс:
```
kubectl port-forward --address 0.0.0.0 svc/netology-svc 8888:80
```
```
http://192.168.10.26:8888/
```

# 2-4

4.
Удалить ресурсы:
```
kubectl delete -f netology-web.yaml
```
```
kubectl delete -f netology-svc.yaml
```

Проверка:
```
kubectl get pods
```
```
kubectl get svc
```

- Service не знает конкретное имя Pod. Он работает с наборами Pod, которые удовлетворяют его selector. Важное условие: Service выбирает Pod в пределах одного namespace.

- Service не управляет Pod, а обеспечивает сетевой доступ к ним.
В этом смысле Service действительно работает как внутренний балансировщик. Но он не создаёт кластер из Pod и не запускает их.

- Структура:
Объект       Ответственность
Pod        - Запуск контейнера или группы контейнеров
Service    - Стабильная сетевая точка входа и распределение соединений
Deployment - Управление количеством Pod, их пересоздание и обновление

Типичная связка выглядит так:
Deployment
  └── ReplicaSet
       ├── Pod web-1
       ├── Pod web-2
       └── Pod web-3
            ^
      Service netology-svc
       (через selector)
            ^
          Клиент

Здесь Deployment отвечает за жизненный цикл Pod, а Service — за сетевой доступ к ним.

- Один Service = одна группа Pod.
При этом группы не обязаны быть взаимоисключающими: несколько Service могут выбирать одни и те же Pod, например для разных портов.

---
---
