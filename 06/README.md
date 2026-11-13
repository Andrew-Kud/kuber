# 1:

1.
проверь состояние и готовность:
```
microk8s status --wait-ready
```
```
microk8s kubectl get nodes
```

2.
создай отдельный неймспейс:
```
microk8s kubectl create namespace k8s-lab
```

3.
создай configmap-web.yaml
- передача индексов строго вот так `index.html: |` далее перевод строки.

4.
создй deployment.yaml
- монтирование configmap НА директорию скрывает ранее существовавшие в ней файлы.

5.
запускаем сначала конфигмап, потом деплоймент:
```
microk8s kubectl apply -f configmap-web.yaml -n k8s-lab
```
```
microk8s kubectl apply -f deployment.yaml -n k8s-lab
```

6.
проверка:
```
microk8s kubectl -n k8s-lab rollout status deployment/web-demo
```
```
microk8s kubectl get pods -n k8s-lab
```

теперь проверка связи контейнеров:
```
microk8s kubectl -n k8s-lab exec deploy/web-demo -c multitool -- curl -sS http://127.0.0.1:80
```

проверка через curl (два терминала):
```
microk8s kubectl -n k8s-lab port-forward deployment/web-demo 8080:80
```
```
curl -i http://127.0.0.1:8080
```

<img width="1316" height="1397" alt="1-1" src="https://github.com/user-attachments/assets/9b11b216-3ea1-40dd-bef2-31df4997a905" />


Изменил заголовок в configmap-web.yaml для проверки configmap и применил.
```
microk8s kubectl apply -f configmap-web.yaml -n k8s-lab
```
далее проверка через curl (два терминала):
```
microk8s kubectl -n k8s-lab port-forward deployment/web-demo 8080:80
```
```
curl -i http://127.0.0.1:8080
```

<img width="819" height="1027" alt="1-2" src="https://github.com/user-attachments/assets/7b13fc0b-eb35-443c-9bf4-a293d80d3aec" />


вывод:
- перезапуск deploment не нужен. кубер обновляет смонтированные файлы configmap автоматически.
- после применения configmap можно переименовать локальный configmap-web.yaml. Приложение продолжит работать, потому что кубер УЖЕ хранит этот объект независимо от локального файла.
- nginx в pod не имеет доступа к моему локальному файлу ~/06/configmap-web.yaml и вообще не знает о его существовании.
- configmap предназначен для обычных настроек - не для паролей или токенов.


# 2:

1.
чекай поды и конфигмап:
```
microk8s kubectl get pods,configmaps -n k8s-lab
```

2.
генерируй самоподписанные серт:
```
umask 077
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=myapp.example.com" -addext "subjectAltName=DNS:myapp.example.com"
```
- Современные tls используют san для проверки имени сервера, одного CN недостаточно.

С ключами осторожее:
req -x509        - создать самоподписанный сертификат.
-nodes           - не шифровать приватный ключ паролем.
-days 365        - срок действия 365 дней.
-newkey rsa:2048 - создать rsa ключ длиной 2048 бит.
-keyout tls.key  - сохранить приватный ключ.
-out tls.crt     - сохранить сертификат,
-subj            - задать имя сервера (cn).
-addext          - добавить имя сервера в san.

3.
проверка серта:
```
openssl x509 -in tls.crt -noout -subject -dates -ext subjectAltName
```

4.
создай tls секрет secret-tls.yaml:
```
microk8s kubectl create secret tls web-demo-tls --cert=tls.crt --key=tls.key -n k8s-lab --dry-run=client -o yaml > secret-tls.yaml
```
- использовал встроенную команду кубера.
create secret tls   - формирует secret типа kubernetes.io/tls.
--cert и --key      - берёт сертификат и ключ из файлов.
--dry-run=client    - формирует объект локально, не создавая его в кубере.
-o yaml             - выводит результат вyaml.
`> secret-tls.yaml` - сохраняет результат в файл.

5.
защита:
```
chmod 600 tls.key secret-tls.yaml
```

6.
создание секрета:
```
microk8s kubectl apply -f secret-tls.yaml
```
проверка:
```
microk8s kubectl get secret web-demo-tls -n k8s-lab
```
- DATA: 2 - secret содержит два элемента: tls.crt и tls.key.

7.
создай service для nginx - task2/service.yaml

8.
примени сервис:
```
microk8s kubectl apply -f service.yaml -n k8s-lab
```
проверка:
```
microk8s kubectl get svc -n k8s-lab
```
```
microk8s kubectl get endpointslices -n k8s-lab -l kubernetes.io/service-name=web-demo-svc
```

9.
в 1.35 кубере появился аддон с ingress. документация прямо настаивает на его применении, а не костылить через nginx.
```
microk8s enable ingress
```
```
microk8s kubectl get ingressclass
```
```
microk8s kubectl get pods,svc -n ingress
```

10.
создай https ingress - task2/ingress-tls.yaml.

11.
примени новый манифест:
```
microk8s kubectl apply -f ingress-tls.yaml -n k8s-lab
```
проверь:
```
microk8s kubectl get ingress -n k8s-lab
```
```
microk8s kubectl describe ingress web-demo-tls -n k8s-lab
```
проверка через curl для https:
```
curl -k -i --resolve myapp.example.com:443:127.0.0.1 https://myapp.example.com/
```
- команда рассчитана на то, что ingress контролер доступен по 443.

<img width="954" height="836" alt="2-1" src="https://github.com/user-attachments/assets/118f2ce6-4353-4e71-b0ba-a07b78516e8d" />


12.
проверка валидности серта:
- успешный curl -k подтверждает работу https, но сам по себе не доказывает, что ingress отдаёт твой серт, а не серт traefik по умолчанию.
```
echo | openssl s_client -connect 127.0.0.1:443 -servername myapp.example.com 2>/dev/null | openssl x509 -noout -subject -issuer
```

<img width="286" height="127" alt="2-2" src="https://github.com/user-attachments/assets/d9cac821-a517-4129-8a0c-0b62020bd160" />


# 3:

1.
включи RBAC:
```
microk8s enable rbac
```

2.
проверь:
```
sudo grep -- '--authorization-mode' /var/snap/microk8s/current/args/kube-apiserver
```
`--authorization-mode=RBAC,Node`

проверь что админский доступ остался:
```
microk8s kubectl auth can-i list pods -n k8s-lab
```
`yes`

3.
генерируй серт пользователя (понадобится 3):
```
umask 077
openssl genrsa -out developer.key 2048
```
```
openssl req -new \
  -key developer.key \
  -out developer.csr \
  -subj "/CN=developer"
```
developer.key - приватный ключ.
developer.csr - запрос на выпуск серта.
developer.crt - серт пользователя подписанный CA кластера.
- не добавляй группу O=system:masters, она предоставляет админский доступ.

4.
подпиши серт пользователя через CA mickro8s:

копировать публичный ca в текущую директорию:
```
sudo cat /var/snap/microk8s/current/certs/ca.crt > ca.crt
```

создание расширения tasl3/developer-ext.cnf
- явно указывает, что сертификат предназначен только для клиентской аутентификации (а не для роли центра сертификации).

подпиши серт:
```
sudo openssl x509 -req \
  -in developer.csr \
  -CA /var/snap/microk8s/current/certs/ca.crt \
  -CAkey /var/snap/microk8s/current/certs/ca.key \
  -set_serial "0x$(openssl rand -hex 16)" \
  -out /dev/stdout \
  -days 365 \
  -sha256 \
  -extfile developer-ext.cnf > developer.crt
```
- вместо -CAcreateserial используй случайный серийный номер, чтобы не создавать и не менять файл последовательности серийных номеров рядом с CA кластера. При этом приватный ключ CA используется только при подписании и остаётся на своём месте.

проверь сертификат:
```
openssl x509 -in developer.crt -noout -subject -issuer -dates -ext extendedKeyUsage
```

проверь крипто-подпись:
```
openssl verify -CAfile ca.crt developer.crt
```

<img width="954" height="419" alt="3-1" src="https://github.com/user-attachments/assets/c077e200-5860-48df-8aa1-976b31735db6" />


5.
создай роль - task3/role.yaml
- pods/log — отдельный subresource. Поэтому право get на обычный pods автоматически не даёт права читать его логи.

6.
создай task3/rolebinding.yaml
subjects.name: developer     - имя пользователя из CN сертификата.
roleRef.name: pod-log-reader - имя созданной роли.
namespace: k8s-lab           - область действия rolebinding.

7.
примени новый манифест:
```
microk8s kubectl apply -f role.yaml
```
```
microk8s kubectl apply -f rolebinding.yaml
```

проверь:
```
microk8s kubectl get role,rolebinding -n k8s-lab
```

<img width="772" height="246" alt="3-2" src="https://github.com/user-attachments/assets/0f0d3ef0-75d9-459f-8c2f-7afcda06f73d" />


8.
создай отдельный kubeconfig для developer юзера.

зарегистрируй кластер:
```
microk8s kubectl \
  --kubeconfig="$PWD/developer.kubeconfig" \
  config set-cluster local-microk8s \
  --server=https://127.0.0.1:16443 \
  --certificate-authority="$PWD/ca.crt" \
  --embed-certs=true
```
`Cluster "local-microk8s" set.`
- указываем адрес kubernetes API server и ca, кторому доверяем.

9.
добавь сертификат пользователя:
```
microk8s kubectl \
  --kubeconfig="$PWD/developer.kubeconfig" \
  config set-credentials developer \
  --client-certificate="$PWD/developer.crt" \
  --client-key="$PWD/developer.key" \
  --embed-certs=true
```
`User "developer" set.`
- `--embed-certs=true` сохраняет сертификат и приватный ключ внутри kubeconfig в base64. теперь сам developer.kubeconfig является чувствительным файлом.

10.
создай context:
```
microk8s kubectl \
  --kubeconfig="$PWD/developer.kubeconfig" \
  config set-context developer@local-microk8s \
  --cluster=local-microk8s \
  --user=developer \
  --namespace=k8s-lab
```
`Context "developer@local-microk8s" created.`

11.
активируй контекст:
```
microk8s kubectl --kubeconfig="$PWD/developer.kubeconfig" config use-context developer@local-microk8s
```
`Switched to context "developer@local-microk8s".`

12.
ограничь доступ к файлу:
```
chmod 600 developer.kubeconfig
```

kubeconfig содержит:
- Cluster (куда подключаться).
- User (чем аунтефицироваться).
- Context (какой кластер, пользователь, namespace использовать).
- namespace в kubeconfig - только значение по умолчанию.
- само ограничение прав обеспечивает RoleBinding.

13.
проверка разрешённых операций:

кто подключился:
```
microk8s kubectl --kubeconfig="$PWD/developer.kubeconfig" auth whoami
```
`Username - developer` подтверждает, что кубер видит нас так.

посмотри под:
```
microk8s kubectl --kubeconfig="$PWD/developer.kubeconfig" get pods
```
- должен вернуться успех.

посмотри описание под:
```
POD=$(microk8s kubectl -n k8s-lab get pods -l app=web-demo -o jsonpath='{.items[0].metadata.name}')
```
- я использую админа только для получения имени.

```
microk8s kubectl --kubeconfig="$PWD/developer.kubeconfig" describe pod "$POD"
```
- должены вернуться сведенья о контейнерах/

проверь чтение логов:
```
microk8s kubectl --kubeconfig="$PWD/developer.kubeconfig" logs "$POD" -c nginx --tail=10
```
- должен увидить логи.

<img width="1143" height="1249" alt="3-3" src="https://github.com/user-attachments/assets/1d6ab8d0-d17e-4941-ab8d-a05099019861" />


14.
проверка запрещённых операция:

пробую получить секреты:
```
microk8s kubectl --kubeconfig="$PWD/developer.kubeconfig" get secret web-demo-tls
```
`Error from server`

проверка rbac:
```
microk8s kubectl --kubeconfig="$PWD/developer.kubeconfig" auth can-i delete pods
```
`no`

проверка выполнения комманд внутри контейнера:
```
microk8s kubectl --kubeconfig="$PWD/developer.kubeconfig" auth can-i create pods/exec
```
`no`

проверка границы namespace:
```
microk8s kubectl --kubeconfig="$PWD/developer.kubeconfig" auth can-i get pods -n default
```
`no`

сертификат не определяет права пользователя, он подтверждает, что пользователь - developer. роли определяют, что он может делать, а rolebinding связывает эти права с пользователем.


<img width="1288" height="424" alt="3-4" src="https://github.com/user-attachments/assets/48e24994-6544-46d6-a469-07d77909a955" />

15.
удалить созданные ресурсы:
```
microk8s kubectl delete namespace k8s-lab
```

16.
отключить rbac и ingress
```
microk8s disable rbac
```
```
microk8s disable ingress
```

серты CA не удаляй. они принадлежат кластеру.


---
---
