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

<img width="771" height="1235" alt="1-1" src="https://github.com/user-attachments/assets/05a59085-c151-4588-85fb-f2f066fa4447" />


Чтение в реальном времени
```
microk8s kubectl exec -it deploy/shared-volume-demo -c multitool -- tail -n 0 -f /shared/timestamps.log
```

<img width="1083" height="169" alt="1-2" src="https://github.com/user-attachments/assets/aa2a1758-4306-44ab-afb0-e3b00d86c0a9" />

<img width="1049" height="237" alt="1-3" src="https://github.com/user-attachments/assets/22c418f4-2287-4fdf-971e-9984cabca2ff" />


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

<img width="1282" height="917" alt="2-1" src="https://github.com/user-attachments/assets/d2a66b25-7faa-479d-afad-c5a1da96001e" />

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

<img width="923" height="108" alt="2-2" src="https://github.com/user-attachments/assets/560940e0-6893-4c95-9cd5-6e1a73d8c200" />



5.
Удаление ресурсов:
```
microk8s kubectl delete -f daemonset.yaml
```
- удалит daemonset и созданные им pod. Файл /var/log/syslog на хосте останется, потому что кубер его не создавал и не владеет им.


---
---
