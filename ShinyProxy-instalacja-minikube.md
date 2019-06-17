# ShinyProxy instalacja na minikube

Instrukcja uruchomienia ShinyProxy na minikube. Można użyć takiego środowiska do testów.

# Wymagania
Kubernetes uruchomiony za pomocą minikube.

# Instalacja

Patrz: [ShinyProxy-instalacja-Kubernetes.md](ShinyProxy-instalacja-Kubernetes.md).  
Poniżej opisane są tylko elementy specyficzne dla minikube.

## Tworzenie konta dla ShinyProxy w Kubernetesie

Procedura specyficzna dla minikube.

Wykonujemy w maszynie nadzorcy:
```shell
$ ~/.local/bin/minikube ip
192.168.99.101
$ ~/.local/bin/minikube ssh-key
/home/jkowalski/.minikube/machines/minikube/id_rsa
$ cd ~/minikube-shinyproxy1/
$ ssh -i /home/jkowalski/.minikube/machines/minikube/id_rsa docker@192.168.99.101 sudo tar c /var/lib/minikube/certs | tar -vx
$ openssl genrsa -out shinyproxy1user1.key 2048
$ openssl req -new -key shinyproxy1user1.key -out shinyproxy1user1.csr -subj "/CN=shinyproxy1user1/O=shinyproxy1"
$ cd ~/minikube-shinyproxy1/var/lib/minikube/certs
$ openssl x509 -req -in ~/minikube-shinyproxy1/shinyproxy1user1.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ~/minikube-shinyproxy1/shinyproxy1user1.crt -days 900
```

Robimy backup pliku `~/.kube/config`.
```shell
$ cp -v ~/.kube/config ~/.kube/config.$(date +%Y%m%d-%H%M%S).backup
```

Dodajemy nowego użytkownika do `~/.kube/config`:
```shell
$ kubectl config set-credentials shinyproxy1user1 --client-certificate=${HOME}/minikube-shinyproxy1/shinyproxy1user1.crt --client-key=${HOME}/minikube-shinyproxy1/shinyproxy1user1.key
$ kubectl config set-context shinyproxy1user1-context --cluster=minikube --namespace=shinyproxy1 --user=shinyproxy1user1
```

```shell
$ kubectl --context=shinyproxy1user1-context get pods
Error from server (Forbidden): pods is forbidden: User "shinyproxy1user1" cannot list resource "pods" in API group "" in the namespace "shinyproxy1"
```

Tak, powyższe polecenie powinno zwrócić błąd, bo nie ustawiliśmy jeszcze uprawnień.

