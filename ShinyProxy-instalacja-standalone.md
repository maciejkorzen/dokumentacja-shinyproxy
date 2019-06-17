# Wstęp

Instrukcja uruchomienia ShinyProxy działającego jak zwykła (tradycyjna) aplikacja.  
Do uruchamiania kontenerów będzie użyty zwykły Docker.

# Instalacja ShinyProxy

Źródło: [https://www.shinyproxy.io/getting-started/](https://www.shinyproxy.io/getting-started/).

Należy zainstalować Dockera.

```shell
# mkdir /etc/systemd/system/docker.service.d
# cat << EOF > /etc/systemd/system/docker.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -D -H tcp://127.0.0.1:2375
EOF
# systemctl daemon-reload
# systemctl restart docker.service
```

Ze strony [https://www.shinyproxy.io/downloads/](https://www.shinyproxy.io/downloads/) pobieramy plik z pakietem DEB i instalujemy go w systemie.

```shell
cat << EOF > /etc/shinyproxy/application.yml
proxy:
  title: Open Analytics Shiny Proxy
  logo-url: http://www.openanalytics.eu/sites/www.openanalytics.eu/themes/oa/logo.png
  landing-page: /
  heartbeat-rate: 10000
  heartbeat-timeout: 60000
  port: 8080
  authentication: simple
  admin-groups: scientists
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
  docker:
    cert-path: /home/none
    url: http://localhost:2375
    port-range-start: 20000
  specs:
    - id: euler
      display-name: Euler's number
      container-cmd: ["R", "-e", "shiny::runApp('/root/euler')"]
      container-image: openanalytics/shinyproxy-template
      access-groups: [scientists, mathematicians]
    - id: 01_hello
      container-cmd: ["R", "-e", "shinyproxy::run_01_hello()"]
      container-image: openanalytics/shinyproxy-demo
      access-groups: [scientists, mathematicians]
    - id: 06_tabsets
      container-cmd: ["R", "-e", "shinyproxy::run_06_tabsets()"]
      container-image: openanalytics/shinyproxy-demo
      access-groups: [scientists, mathematicians]
    - id: shinyproxy-dashboard-test-one
      container-cmd: ["R", "-e", "shiny::runApp('/root/shinyproxy-dashboard-test-one')"]
      container-image: shinyproxy-dashboard-test-one
      access-groups: [scientists, mathematicians]

logging:
  file:
    shinyproxy.log

server:
  servlet.session.timeout: 3600
EOF
```

```shell
systemctl restart shinyproxy.service
```

ShinyProxy słucha na porcie TCP 8080.

Musimy mieć pobrane do lokalnego Dockera obrazy kontenerów, które będziemy uruchamiać za pomocą ShinyProxy.

