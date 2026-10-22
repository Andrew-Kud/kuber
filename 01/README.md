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
<img width="418" height="74" alt="1-1" src="https://github.com/user-attachments/assets/1ae5a7f0-f562-434d-9711-9b68b76a46f0" />

<img width="736" height="88" alt="1-2" src="https://github.com/user-attachments/assets/57d90374-fd8a-4891-8501-9a89f01ea450" />


Но скорее всего будет так:
<img width="792" height="69" alt="1-3" src="https://github.com/user-attachments/assets/b1737cd8-09d5-481b-a755-d2c33292192c" />


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
<img width="736" height="88" alt="1-2" src="https://github.com/user-attachments/assets/d2dadc70-50ae-4b4b-9f67-ed555074e3b5" />


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
<img width="776" height="241" alt="1-4" src="https://github.com/user-attachments/assets/0aba2264-d53f-40af-8a51-c76e24198f17" />



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
<img width="2553" height="1394" alt="1-5" src="https://github.com/user-attachments/assets/04e264c2-2f85-421c-ae8b-b54e7a334a2e" />



---
---
