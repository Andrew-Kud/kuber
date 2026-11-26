# 1:

1.
установка help:

```
sudo apt update
```
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
```
```
less get_helm.sh
```
```
chmod 700 get_helm.sh
```
```
./get_helm.sh
```


2.
проверка:
```
helm version
```
```
microk8s status --wait-ready
```
```
microk8s kubectl get nodes
```
- coredns должен быть enable, если нет:
```
microk8s enable dns
```

а так же:
```
sudo usermod -aG microk8s "$USER"
```


3.
подключение helm к кубу:
```
mkdir -p ~/.kube
```
```
umask 077
microk8s config > ~/.kube/microk8s.yaml
```
```
export KUBECONFIG="$HOME/.kube/microk8s.yaml"
```
- Переменная KUBECONFIG действует только в текущей сессии.

проверка:
```
helm list -A
```


4.
принцип Helm - отделить настройки приложения от кубер-манифстов.
Он подставит значения из values.yaml, сформировав обычный кубер deployment (принцип как в терраформе).
Так же объединяет values.yaml с параметрами указанного окружения. Не заданные в valies-prod.yaml значения берутся из values.yaml.

Создал task01 help манифесты для nginx.

Проверка:
```
helm lint ./
```

Посмотреть что будет установлено:
```
helm template demoapp ./ -f ./values-dev.yaml
```


5.
Усановка dev:
```
helm upgrade --install demoapp ./ -n dev --create-namespace -f ./values-dev.yaml --wait --timeout 180s
```
скорее всего будет ошибка `Error: kubernetes cluster unreachable: Get "http://localhost:8080/version": dial tcp 127.0.0.1:8080: connect: connection refused`

тогда:
```
export KUBECONFIG="$HOME/.kube/microk8s.yaml"
```
и проверь что `~/.kube/microk8s.yaml` существует и заполнен.
и ещё это:
```
helm list -A
```
после чего повтори первую команду.

# 1-1


6.
Проверь:
```
helm list -n dev
```
```
microk8s kubectl get deployments -n dev
```
```
microk8s kubectl get pods -n dev
```
```
microk8s kubectl get svc -n dev
```

# 1-2

в одном окне терминала:
```
microk8s kubectl -n dev port-forward svc/demoapp-frontend 8080:80
```
в другом окне:
```
curl http://127.0.0.1:8080/api/
```

# 1-3


7.
установка prod:
```
helm upgrade --install demoapp ./ -n prod --create-namespace -f ./values-prod.yaml --wait --timeout 180s
```
и проверь сразу:
```
microk8s kubectl get deployments,pods -n prod
```

# 1-4

- helm управляет двумя независимыми релизами с именем demoapp, расположенными в разных namespace.
```
microk8s kubectl get deployments,pods -A
```
или
```
microk8s kubectl get deployments,pods -A | grep -E 'NAMESPACE|demoapp'
```

# 1-5


8.
```
microk8s kubectl -n dev get deployment demoapp-frontend -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```
`nginx:1.30.0-alpine`

в `nano ./values-dev.yaml`
и добавь `tag`, что бы в итоге `values-dev.yaml` был:
```
frontend:
  replicaCount: 1
  image:
    tag: "1.30.6-alpine"

backend:
  replicaCount: 1
  message: "Hello from DEV"
```

и выполни:
```
helm upgrade demoapp ./ -n dev -f ./values-dev.yaml --wait
```

обнови:
```
microk8s kubectl rollout status deployment/demoapp-frontend -n dev
```
`deployment "demoapp-frontend" successfully rolled out`

проверь:
```
microk8s kubectl -n dev get deployment demoapp-frontend -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```
`nginx:1.30.6-alpine`

- helm обнаружил изменение в шаблоне deployment и отправил обновлённую конфигурацию в кубер. Контроллер deployment создал pod с новым образом, выполнив rolling update. Версия backend не изменилась.

# 1-6


9.
rollback.
```
helm history demoapp -n dev
```
- helm хранит историю ревизий релиза. Псле первоначальной установки и одного обновления должны существовать как минимум две ревизии.

Откат к первоначальной:
```
helm rollback demoapp 1 -n dev --wait
```

проверка:
```
microk8s kubectl -n dev get deployment demoapp-frontend -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```
`nginx:1.30.0-alpine`
- rollback возвращает kuber-ресурсы к прежней конфигурации helm-релиза, но не откатывает данные в БД или томах.

# 1-7


10.
упаковка:
```
helm package ./
```
- этот helmпакет, который можно перенести на другую машину или опубликовать в репозитории.

# 1-8

проверка:
```
helm template demoapp ./demoapp-0.1.0.tgz -f ./values-dev.yaml
```
- helm берёт шаблоны из архива, подставляет значения и генерирует yaml.

Просмотр уже установленных ресурсов:
```
helm get manifest demoapp -n dev
```

Фактическое состояние кубернетиса:
```
microk8s kubectl get all -n dev
```
```
microk8s kubectl get configmap -n dev
```

---

# 2:

1.
```
export KUBECONFIG="$HOME/.kube/microk8s.yaml"
```
```
helm lint .
```


2.
Первая версия с неймспейсом app1:
```
helm upgrade --install demo-v1 . -n app1 --create-namespace -f values-dev.yaml --set-string frontend.image.tag=1.30.0-alpine --set-string backend.message="Hello from V1" --wait --timeout 180s
```

Вторая версия с неймспейсом app1:
```
helm upgrade --install demo-v2 . -n app1 --create-namespace -f values-dev.yaml --set-string frontend.image.tag=1.30.6-alpine --set-string backend.message="Hello from V2" --wait --timeout 180s
```

Третья версия с неймспейсом app2:
```
helm upgrade --install demo-v3 . -n app2 --create-namespace -f values-dev.yaml --set-string frontend.image.tag=1.30.8-alpine --set-string backend.message="Hello from V3" --wait --timeout 180s
```

проверка:
```
helm list -A
```
```
microk8s kubectl get deployments,pods,svc -n app1
```
```
microk8s kubectl get deployments,pods,svc -n app2
```
```
microk8s kubectl get deployments -A -l app.kubernetes.io/name=demoapp -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image,READY:.status.readyReplicas'
```

# 2-1


3.
проверка http:
открой три локалных проброса портов в фоне:
```
microk8s kubectl -n app1 port-forward svc/demo-v1-frontend 8081:80 >/tmp/demo-v1.log 2>&1 & P1=$!
```
```
microk8s kubectl -n app1 port-forward svc/demo-v2-frontend 8082:80 >/tmp/demo-v2.log 2>&1 & P2=$!
```
```
microk8s kubectl -n app2 port-forward svc/demo-v3-frontend 8083:80 >/tmp/demo-v3.log 2>&1 & P3=$!
```

Отправь запросы:
```
curl -fsS --retry 5 --retry-connrefused http://127.0.0.1:8081/api/
```
```
curl -fsS --retry 5 --retry-connrefused http://127.0.0.1:8082/api/
```
```
curl -fsS --retry 5 --retry-connrefused http://127.0.0.1:8083/api/
```

Отанови пробросы:
```
kill "$P1" "$P2" "$P3"
```

# 2-2


4.
# 2-3


---
---
