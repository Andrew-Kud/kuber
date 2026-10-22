Квест по установке Кубер и Докер под санкциями.

Шаги для Ubuntu server 24.04:


1.
```
sudo apt opdate && sudo apt upgrade -y
```
```
sudo apt install -y snapd
```


2.
```
sudo snap install microk8s --classic --channel=1.35/stable
```
Нужно подождать около 1 минуты, а пока ждёшь:
```
mkdir ~/.kube
```
```
sudo usermod -aG microk8s $USER
```
```
sudo chown -f -R $USER ~/.kube
```
```
newgrp microk8s
```


3.
Проверь статус:
```
microk8s kubectl get nodes
```
```
microk8s kubectl get pods -A
```
Должно быть так:
1-1
1-2

Но скорее всего будет так:
1-3

тут либо vpn, либо зеркало docker.io.
Зеркало настраивается так:
```
sudo mkdir -p /var/snap/microk8s/current/args/certs.d/docker.io
```
```
sudo nano /var/snap/microk8s/current/args/certs.d/docker.io/hosts.toml
```
```
server = "https://docker.io"

[host."https://dh-mirror.gitverse.ru"]
  capabilities = ["pull", "resolve"]
```

Пока что не актуально, но кто знает, может и k8s тоже скажет досвидания, тогда нужно будет:
```
sudo mkdir -p /var/snap/microk8s/current/args/certs.d/registry.k8s.io
```
```
sudo nano /var/snap/microk8s/current/args/certs.d/registry.k8s.io/hosts.toml
```
```
server = "https://registry.k8s.io"

[host."https://mirror.gcr.io"]
  capabilities = ["pull", "resolve"]
```

Далее перезапуск:
```
sudo microk8s stop
```
```
sudo microk8s start
```
и через 1 минуту должно быть так:
1-2


4.
Алиасы:
```
sudo snap alias microk8s.kubectl kubectl
```
```
sudo snap alias microk8s.helm3 helm
```


5.
dns и dashboard:
```
microk8s enable dns
```
```
microk8s enable dashboard
```


6.
Проверь запуск дашборда:
```
kubectl get pods -n kubernetes-dashboard
```
```
kubectl get svc -n kubernetes-dashboard
```
1-4


7.
Теперь опознай токен:
```
kubectl -n kube-system get secret microk8s-dashboard-token -o jsonpath='{.data.token}' | base64 -d
```
и опубликуй сервис:
```
kubectl -n kubernetes-dashboard port-forward --address 0.0.0.0 svc/kubernetes-dashboard-kong-proxy 18443:443
```
с локального пк заходи (в моём случае ВМ с кубером имеет ip - 192.168.10.27):
```
https://192.168.10.27:18443
```
Скопируй токен и вставь в web, после чего увидишь:
1-5



---
---
