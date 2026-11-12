# 1:

deployment   - управляет созданием Pod и поддержанием нужного количества реплик.
pod          - объединяет контейнеры, которые могут использовать общие ресурсы.
volume       - предоставляет общий каталог.
volumemounts - подключает этот каталог к файловой системе кокретного контейнера.


1.
Создай deployment.yaml


2.
Запус deployment:
```
microk8s status --wait-ready
```
```
microk8s kubectl apply -f deployment.yaml
```


3.
Проверка:
```
microk8s kubectl rollout status deployment/shared-volume-demo
```
```
microk8s kubectl get pods -l app=shared-volume-demo
```

Чтение из busybox:
```
microk8s kubectl exec deploy/shared-volume-demo -c busybox -- cat /shared/timestamps.log
```

Чтение из multitool:
```
microk8s kubectl exec deploy/shared-volume-demo -c multitool -- cat /shared/timestamps.log
```

# 1-1

Чтение в реальном времени
```
microk8s kubectl exec -it deploy/shared-volume-demo -c multitool -- tail -n 0 -f /shared/timestamps.log
```

# 1-2

# 1-3


4.
Удаление:
```
microk8s kubectl delete -f deployment.yaml
```
- emptyDir существует в течение жизни pod.
- Перезапуск busybox не уничтожит файлы, но пересоздание pod приведёт к потере данных.
- удаление приведёт к тому, что данные в общем томе будут потеряны без возможности восстановления из этого тома.
- pod - это единица куба, внутри которой контейнеры могут совместно использовать ресурсы, в том числе volumes.
- Для временного обмена файлами между контейнерами одного pod подходит emptyDir. Для постоянного хранения данных понадобится PVC.


# 2:

demonset распрастраняется на каждую подходящую ноду, без необходимости указывания кол-ва реплик. нода появилась - demonset распрастранил pod на неё.

deployment использует для размещения приложения и сервисов налюбой подходящей ноде.
demonset используется для мониторинга, сбора логов или агентов на каждой подходящей ноде.

1.
Проверка syslog на хосте:
```
sudo tail -n 5 /var/log/syslog
```
```
systemctl status rsyslog
```

2.
Создай task2/daemonset.yaml

3.
Запуск:
```
microk8s kubectl apply -f daemonset.yaml
```

4.
Проверка:
```
microk8s kubectl get daemonsets
```
```
microk8s kubectl get pods -l app=node-syslog-reader -o wide
```

досуп к файлу из multitool:
```
POD=$(microk8s kubectl get pods -l app=node-syslog-reader -o jsonpath='{.items[0].metadata.name}')
```
```
echo "$POD"
```

определить пользователя внутри контейнера:
```
microk8s kubectl exec "$POD" -- id
```

проверка существование смонтированного файла:
```
microk8s kubectl exec "$POD" -- ls -l /host/var/log/syslog
```

чтение системных логов ноды из контейнера:
```
microk8s kubectl exec "$POD" -- tail -n 20 /host/var/log/syslog
```

# 2-1

доп.проверка:
```
logger -t devops-lab "DaemonSet hostPath test"
```
```
sudo grep "DaemonSet hostPath test" /var/log/syslog
```
```
microk8s kubectl exec "$POD" -- grep "DaemonSet hostPath test" /host/var/log/syslog
```

# 2-2


5.
Удаление ресурсов:
```
microk8s kubectl delete -f daemonset.yaml
```
- удалит daemonset и созданные им pod. Файл /var/log/syslog на хосте останется, потому что кубер его не создавал и не владеет им.


---
---
