# Wstęp

Ogólne instrukcje instalacji ShinyProxy z użyciem Kubernetesa do uruchamiania kontenerów.

# Instalacja ShinyProxy

W maszynie nadzorcy wykonujemy:
```shell
kubectl create namespace shinyproxy1
```

Tworzymy konto dla ShinyProxy w Kubernetesie.
Ogólna instrukcja dla Kubernetesa zainstalowanego za pomocą Kubespray jest w dokumentacji do Kubernetesa.  
Instrukcja specyficzna dla minikube jest w pliku [ShinyProxy-instalacja-minikube.md](ShinyProxy-instalacja-minikube.md).

Nadajemy uprawnienia.

```shell
$ cat > 4-create-role.yaml << EOF
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: shinyproxy1
  name: deployment-manager
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods", "services", "endpoints", "pods/exec" ]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # You can also use ["*"]
EOF
$ kubectl create -f 4-create-role.yaml
role.rbac.authorization.k8s.io/deployment-manager created
```

```shell
$ cat > 5-bind-role.yaml << EOF
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: deployment-manager-binding
  namespace: shinyproxy1
subjects:
- kind: User
  name: shinyproxy1user1
  apiGroup: ""
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: ""
EOF
$ kubectl create -f 5-bind-role.yaml
rolebinding.rbac.authorization.k8s.io/deployment-manager-binding created
```

Teraz już nie powinno być takiego błędu, jak wcześniej:
```shell
$ kubectl --context=shinyproxy1user1-context get pods
No resources found.
```

Tworzymy sekret z loginem i hasłem do Docker Registry.

```shell
$ kubectl create secret docker-registry dockerregistry1 -n shinyproxy1 --docker-server=https://filesrv1.example.com:6756/ --docker-username=__LOGIN__ --docker-password=__HASŁO__ --docker-email=root@example.com
```

Sprawdzamy czy sekret utworzył się prawidłowo:

```shell
$ kubectl get secret dockerregistry1 -n shinyproxy1 --output=yaml
```

Jeśli nie mamy jeszcze żadnego kontenera z dashboardem Shiny i potrzbuhemy jakiegoś do testów, to należy na dowolnym komputerze, który ma dostęp do Docker Registry, zbudować lokalnie obraz Dockera `openanalytics/shinyproxy-template` (szczegółów poszukać w Google). No chyba, że już został kiedyś zbudowany (tzn. stawiamy kolejny klaster Kubernetes, który korzysta z tego samego Docker Registry).

Wgrywanie obrazu kontenera `openanalytics/shinyproxy-template` do Docker Registry:
```shell
$ docker image tag openanalytics/shinyproxy-template filesrv1.example.com:6756/openanalytics/shinyproxy-template
$ docker push filesrv1.example.com:6756/openanalytics/shinyproxy-template
```

Zbudowanie obrazu sidecar dla ShinyProxy. (No chyba, że już został kiedyś zbudowany (tzn. stawiamy kolejny klaster Kubernetes, który korzysta z tego samego Docker Registry).):

```shell
$ git clone https://github.com/openanalytics/shinyproxy-config-examples
$ cd shinyproxy-config-examples/03-containerized-kubernetes/kube-proxy-sidecar
$ docker build . -t kube-proxy-sidecar
$ docker image tag kube-proxy-sidecar filesrv1.example.com:6756/kube-proxy-sidecar
$ docker push filesrv1.example.com:6756/kube-proxy-sidecar
```

Przygotowujemy obraz kontenera z ShinyProxy. Alternatywnie można użyć tego samego obrazu, który został zbudowany dla innej instancji ShinyProxy.  
Zakładam, że plik `Dockerfile` oraz wszystkie inne pliki kontenera umieszczamy w `"${HOME}/shinyproxy-docker`.  
Obraz musi zawierać klucz i certyfikat konta w Kubernetesie. Polecenia kopiujące je ze źródłowej lokalizacji są zależne od tego jak zainstalowaliśmy klaster.

Przykład dla minikube:

```shell
$ mkdir -p "${HOME}/shinyproxy-docker/kubernetes-certs"
$ cp "${HOME}/minikube-shinyproxy1/shinyproxy1user1.crt" "${HOME}/shinyproxy-docker/kubernetes-certs/cert.pem"
$ cp "${HOME}/minikube-shinyproxy1/shinyproxy1user1.key" "${HOME}/shinyproxy-docker/kubernetes-certs/key.pem"
$ cp "${HOME}/minikube-shinyproxy1/var/lib/minikube/certs/ca.crt" "${HOME}/shinyproxy-docker/kubernetes-certs/ca.pem"
$ cd "${HOME}/shinyproxy-docker/"
```

Przykład dla Kubernetesa zainstalowanego za pomocą kubespray:

```shell
jkowalski@localhost:/kubespray$ ssh -F ./ssh-config vagrant@k8s-1 sudo tar -C /etc/kubernetes -c ssl | tar -C "${HOME}" -vx; mv "${HOME}/ssl" "${HOME}/kubernetes-ssl"
ssl/
ssl/apiserver.crt
ssl/apiserver-kubelet-client.crt
ssl/apiserver.key
ssl/ca.crt
ssl/apiserver-kubelet-client.key
ssl/front-proxy-ca.key
ssl/etcd/
ssl/etcd/node-k8s-4-key.pem
ssl/etcd/node-k8s-4.pem
ssl/etcd/ca.pem
ssl/ca.srl
ssl/front-proxy-client.key
ssl/front-proxy-ca.crt
ssl/sa.pub
ssl/sa.key
ssl/ca.key
ssl/front-proxy-client.crt
```

```shell
$ mkdir -p "${HOME}/shinyproxy-docker/kubernetes-certs"
$ cp "${HOME}/kubernetes-accounts/shinyproxy1user1/shinyproxy1user1.crt" "${HOME}/shinyproxy-docker/kubernetes-certs/cert.pem"
$ cp "${HOME}/kubernetes-accounts/shinyproxy1user1/shinyproxy1user1.key" "${HOME}/shinyproxy-docker/kubernetes-certs/key.pem"
$ cp "${HOME}/kubernetes-ssl/ca.crt" "${HOME}/shinyproxy-docker/kubernetes-certs/ca.pem"
$ cd "${HOME}/shinyproxy-docker/"
```

```shell
$ cat > Dockerfile << EOF
FROM openjdk:8-jre

RUN apt update; apt install -y netcat-traditional tcpdump
RUN mkdir -p /opt/shinyproxy/ \
        /opt/shinyproxy/container-logs/ \
        /root/ssl-poke/
RUN wget https://www.shinyproxy.io/downloads/shinyproxy-2.1.0.jar -O /opt/shinyproxy/shinyproxy.jar

COPY application.yml /opt/shinyproxy/application.yml
COPY kubernetes-certs /opt/shinyproxy
COPY ssl-poke /root/ssl-poke

WORKDIR /opt/shinyproxy/
CMD ["java", "-jar", "/opt/shinyproxy/shinyproxy.jar"]
EOF
```

```shell
$ cat > application.yml << EOF
proxy:
  title: Shiny Proxy on kubernetes-dev
  logo-url: https://www.openanalytics.eu/oldimg/logo.png
  landing-page: /
  heartbeat-rate: 10000
  heartbeat-timeout: 60000
  port: 8080
  authentication: simple
  admin-groups: scientists
  container-backend: kubernetes
  container-wait-time: 120000
  container-log-path: ./container-logs
  users:
  - name: tesla
    password: password
    groups: scientists
  - name: jack
    password: password
    groups: scientists
  - name: jeff
    password: password
    groups: mathematicians
  kubernetes:
    url: __WSTAW_ADRES_KLASTRA__
    # Przykład: url: https://192.168.99.101:8443/
    # Adres znajdziesz w ~/.kube/config.
    image-pull-secret: dockerregistry1
    image-pull-policy: Always
    internal-networking: true
    cert-path: /opt/shinyproxy/kubernetes-certs
    namespace: shinyproxy1
  specs:
    - id: bigdata-test-oneone
      display-name: bigdata-test-oneone
      description: bigdata-test-oneone
      container-cmd: ["R", "-e", "shiny::runApp('/root/shinyproxy-dashboard-bigdata-test-oneone')"]
      container-image: filesrv1.example.com:6756/shinyproxy-dashboard-bigdata-test-oneone
    - id: bigdata-test-twotwo
      display-name: bigdata-test-twotwo
      description: Aaaaa bbb cccc -- -- DDD eee FFF ggg HHhh
      container-cmd: ["R", "-e", "shiny::runApp('/root/shinyproxy-dashboard-bigdata-test-twotwo')"]
      container-image: filesrv1.example.com:6756/shinyproxy-dashboard-bigdata-test-twotwo
    - id: zespol-test-2-test-threethree
      display-name: zespol-test-2-test-threethree
      description: zespol-test-2-test-threethree
      container-cmd: ["R", "-e", "shiny::runApp('/root/shinyproxy-dashboard-zespol-test-2-test-threethree')"]
      container-image: filesrv1.example.com:6756/shinyproxy-dashboard-zespol-test-2-test-threethree
    - id: zzz_test_01_hello
      display-name: ZZZ Test Hello Application
      description: Application which demonstrates the basics of a Shiny app
      container-cmd: ["R", "-e", "shinyproxy::run_01_hello()"]
      container-image: openanalytics/shinyproxy-demo
    - id: zespol-3-akowalski
      display-name: zespol-3-akowalski
      description: Wzór dashboardu.
      container-cmd: ["R", "-e", "shiny::runApp('/root/shinyproxy-dashboard-zespol-3-akowalski')"]
      container-image: filesrv1.example.com:6756/shinyproxy-dashboard-zespol-3-akowalski
logging:
  file:
    shinyproxy.log

server:
  servlet.session.timeout: 36000
  useForwardHeaders: true

EOF
```

Trzeba zmodyfikować plik `application.yml`. Na pewno trzeba zmienić dane dostępowe do Kubernetesa.

`ssl-poke` skopiować z https://gitlab.example.com/shinyproxy-dashboards/administration/kubernetes-shinyproxy/tree/master/ssl-poke.

```shell
$ docker login https://filesrv1.example.com:6756/
$ docker build . -t shinyproxy-__WSTAW_COŚ__
$ docker image tag shinyproxy-__WSTAW_COŚ__ filesrv1.example.com:6756/shinyproxy-__WSTAW_COŚ__
$ docker push filesrv1.example.com:6756/shinyproxy-__WSTAW_COŚ__
```

Konfigurujemy Kubernetesa:

```shell
$ cat << EOF > sp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shinyproxy
  namespace: shinyproxy1
spec:
  selector:
    matchLabels:
      run: shinyproxy
  replicas: 1
  template:
    metadata:
      labels:
        run: shinyproxy
    spec:
      containers:
      - name: shinyproxy
        image: filesrv1.example.com:6756/shinyproxy-__WSTAW_COŚ__
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
      - name: kube-proxy-sidecar
        image: filesrv1.example.com:6756/kube-proxy-sidecar
        imagePullPolicy: Always
        ports:
        - containerPort: 8001
      imagePullSecrets:
      - name: dockerregistry1
EOF
$ kubectl create -f sp-deployment.yaml
```

```shell
$ cat > sp-authorization.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: shinyproxy-auth
subjects:
  - kind: ServiceAccount
    name: default
    namespace: shinyproxy1
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
$ kubectl create -f sp-authorization.yaml
clusterrolebinding.rbac.authorization.k8s.io/shinyproxy-auth created
```

```shell
$ cat << EOF > sp-service.yaml
kind: Service
apiVersion: v1
metadata:
  namespace: shinyproxy1
  name: shinyproxy
spec:
  type: NodePort
  selector:
    run: shinyproxy
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
    nodePort: 32094
EOF
$ kubectl create -f sp-service.yaml
```

```shell
$ cat << EOF > serviceshinyproxy.yml
apiVersion: v1
kind: Service
metadata:
  name: serviceshinyproxy
  namespace: shinyproxy1
spec:
  selector:
    run: shinyproxy
  ports:
   - protocol: TCP
     port: 8080
EOF
$ kubectl apply -f serviceshinyproxy.yml
```

Prawie skończyliśmy. :-)

Sprawdzamy czy deployment został dodany:

```shell
$ kubectl get deployments --namespace shinyproxy1
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
shinyproxy   1/1     1            1           9s
```

Sprawdzamy czy pody działają:

```shell
$ kubectl get pods --namespace shinyproxy1
NAME                          READY   STATUS    RESTARTS   AGE
shinyproxy-588795c7fc-n528w   2/2     Running   0          20s
```

Musimy jakoś dostać się do ShinyProxy. W tym celu możemy np. przekierować port TCP.

Przykład dla minikube:

```shell
ssh -L 32094:127.0.0.1:32094 -i /home/jkowalski/.minikube/machines/minikube/id_rsa docker@192.168.99.101
```

Przykład dla Kubernetesa uruchomionego za pomocą kubespray i Vagranta:

```shell
jkowalski@localhost:~/kubespray$ vagrant ssh-config > ssh-config
jkowalski@localhost:~/kubespray$ ssh -F ssh-config -L 32094:10.233.66.9:8080 k8s-4

```

W przeglądarce otwieramy adres `http://127.0.0.1:32094/`.

# Ingress

Potrzebny jest certyfikat SSL.

Jeśli nie mamy oficjalnego certyfikatu (np. Let's Encrypt), to do przeprowadzenia testów możemy wygenerować sobie certyfikat self-signed:

```shell
echo -e 'PL\nMazowieckie\nWarszawa\ntest123\nISP\nlocalhost1\nroot@localhost' | openssl req -x509 -nodes -newkey rsa:4096 -sha512 -days 9999 -keyout key.pem -out crt.pem
```

Tworzymy nowy sekret w Kubernetesie zawierający certyfikat. Przykład dla Let's Encrypt:

```shell
root@kubernetes-dev-m0:/etc/letsencrypt/live/kubernetes-dev.example.com_rsa# kubectl create secret tls sslwildcard2 --namespace=shinyproxy1 --key privkey.pem --cert fullchain.pem
secret/sslwildcard2 created
```

Tak wygląda prawidłowo utworzony sekret z certyfikatem Let's Encrypt:

```shell
root@kubernetes-dev-m0:~# kubectl get secret -n shinyproxy1 sslwildcard2
NAME                             TYPE                DATA   AGE
sslwildcard2                     kubernetes.io/tls   2      102d
```

Przykład tworzenia sekretu z certyfikatem self-signed:

```shell
kubectl create secret tls localhostselfsignedcert --namespace=shinyproxy1 --key key.pem --cert crt.pem
```

Tak wygląda prawidłowo utworzony sekret z certyfikatem self-signed:

```shell
% kubectl get secret -n shinyproxy1 localhostselfsignedcert
NAME                      TYPE                DATA   AGE
localhostselfsignedcert   kubernetes.io/tls   2      11m
```

Tworzymy Ingress:

```shell
# cat << EOF > myingressresource2-shinyproxy1.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myingressresource2-shinyproxy
  annotations:
    # shinyproxy wymaga timeoutu 20 dni
    nginx.ingress.kubernetes.io/proxy-read-timeout: "1728000"
    nginx.ingress.kubernetes.io/proxy-request-buffering: "off"
  namespace: shinyproxy1
spec:
  rules:
  - host: shinyproxy.lan
    http:
      paths:
      - backend:
          serviceName: serviceshinyproxy
          servicePort: 8080
        path: /
  tls:
  - hosts:
    - shinyproxy.lan
    secretName: localhostselfsignedcert
EOF
# kubectl apply -f myingressresource2-shinyproxy1.yaml
```

Sprawdzamy jaki adres IP został przydzielony Ingressowi przez MetalLB:
```shell
jkowalski@localhost:~% kubectl get svc -n ingress-nginx ingress-nginx
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.233.54.42   192.168.121.201   80:31149/TCP,443:30480/TCP   14d
```

Szukany adres IP to 192.168.121.201.

Sprawdzamy czy można się połączyć przez HTTP do ShinyProxy:

```shell
% curl -D- http://192.168.121.201 -H 'Host: shinyproxy.lan'
HTTP/1.1 302 Found
Server: nginx/1.15.10
Date: Tue, 16 Apr 2019 14:49:30 GMT
Content-Length: 0
Connection: keep-alive
Expires: 0
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Set-Cookie: JSESSIONID=uytjeTlnkgX5D6Imw6Shr3IvxzFBMmX-y88LFFiW; path=/
X-XSS-Protection: 1; mode=block
Pragma: no-cache
Location: http://shinyproxy.lan/login
X-Content-Type-Options: nosniff
```

Sprawdzamy czy można się połączyć przez HTTPS do ShinyProxy:

```shell
% curl -v -k -D- https://192.168.121.201 -H 'Host: shinyproxy.lan'
* Rebuilt URL to: https://192.168.121.201/
*   Trying 192.168.121.201...
* Connected to 192.168.121.201 (192.168.121.201) port 443 (#0)
* found 151 certificates in /etc/ssl/certs/ca-certificates.crt
* found 608 certificates in /etc/ssl/certs
* ALPN, offering http/1.1
* SSL connection using TLS1.2 / ECDHE_RSA_AES_256_GCM_SHA384
*        server certificate verification SKIPPED
*        server certificate status verification SKIPPED
*        common name: Kubernetes Ingress Controller Fake Certificate (does not match '192.168.121.201')
*        server certificate expiration date OK
*        server certificate activation date OK
*        certificate public key: RSA
*        certificate version: #3
*        subject: O=Acme Co,CN=Kubernetes Ingress Controller Fake Certificate
*        start date: Wed, 17 Apr 2019 13:25:36 GMT
*        expire date: Thu, 16 Apr 2020 13:25:36 GMT
*        issuer: O=Acme Co,CN=Kubernetes Ingress Controller Fake Certificate
*        compression: NULL
* ALPN, server accepted to use http/1.1
> GET / HTTP/1.1
> Host: shinyproxy.lan
> User-Agent: curl/7.47.0
> Accept: */*
>
< HTTP/1.1 302 Found
HTTP/1.1 302 Found
< Server: nginx/1.15.10
Server: nginx/1.15.10
< Date: Wed, 01 May 2019 12:44:37 GMT
Date: Wed, 01 May 2019 12:44:37 GMT
< Content-Length: 0
Content-Length: 0
< Connection: keep-alive
Connection: keep-alive
< Expires: 0
Expires: 0
< Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
< Set-Cookie: JSESSIONID=MYDy4Oy2-ZwR39WuWJ-uYT8RREIRJ78I46eHqun_; path=/
Set-Cookie: JSESSIONID=MYDy4Oy2-ZwR39WuWJ-uYT8RREIRJ78I46eHqun_; path=/
< X-XSS-Protection: 1; mode=block
X-XSS-Protection: 1; mode=block
< Pragma: no-cache
Pragma: no-cache
< Location: https://shinyproxy.lan/login
Location: https://shinyproxy.lan/login
< X-Content-Type-Options: nosniff
X-Content-Type-Options: nosniff
< Strict-Transport-Security: max-age=15724800; includeSubDomains
Strict-Transport-Security: max-age=15724800; includeSubDomains

<
* Connection #0 to host 192.168.121.201 left intact
```

