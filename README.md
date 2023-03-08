# Running [Coder](https://coder.com/) in [k3s](https://k3s.io/)

[Coder](https://coder.com/) is a self-Hosted Remote Development Platform which allows to build your code on servers. This guide helps to deploy [Coder](https://coder.com/) on a k3s cluster.

Before starting the [Coder](https://coder.com/) install process, you must have:

* Infrastructure (to reproduce the experimentation)
  * Three Ubuntu 22.04 machines with SSH credentials.
    * Hostname for each machine (physical or virtual): `k3sserver`, `k3snode1`, `k3snode2`.
    * Same vlan for all machine.
    * All machine have ports 22 (SSH), 80 (HTTP), 443 (HTTPS) and 6443 (Kubernetes) exposed.
  * A domain (coder.mydomain.com), subdomain (*.coder.mydomain.com) and a configured DNS to redirect to `k3sserver`.
  * Reverse Proxy (Apache HTTP or NGINX) will be hosted in outside the Kubernetes cluster.
  * [Docker](https://www.docker.com/) installation on `k3sserver` to deploy Reverse Proxy.

* Local
  * kubectl
  * [HELM](https://helm.sh/)

The plan of this guide is the following:

* [Install k3s](#install-k3s)
* [Deploy Coder](#deploy-coder)
* [Deploy Reverse Proxy](#deploy-reverse-proxy)
    * [NGINX](#nginx)
    * [Apache HTTP](#apache-http)
* [Run](#run)

## Install k3s

* Connect to the server node (`k3sserver`) and run:

```console
$ curl -sfL https://get.k3s.io | sh -
```

* Extract the [k3s](https://k3s.io/) token on the server node:

```console
$ sudo cat /var/lib/rancher/k3s/server/node-token
K20545dbddda0f19bf1c9ac794546d200cdc4ede3fe9ad82d5e560ad0748cc28fd4::server:17a174d18d4fd82c0f99b687bd9aabcd
```

> 🤓 It is my [k3s](https://k3s.io/) token, of course, extract YOUR [k3s](https://k3s.io/) token.

* Connect to the first agent node (`k3snode1`) and run:

```
$ mynodetoken=K20545dbddda0f19bf1c9ac794546d200cdc4ede3fe9ad82d5e560ad0748cc28fd4::server:17a174d18d4fd82c0f99b687bd9aabcd
$ curl -sfL https://get.k3s.io | K3S_URL=https://k3sserver:6443 K3S_TOKEN=mynodetoken sh -
```

* Connect to the second agent node (`k3snode2`) and execute the same command line:

```
$ mynodetoken=K20545dbddda0f19bf1c9ac794546d200cdc4ede3fe9ad82d5e560ad0748cc28fd4::server:17a174d18d4fd82c0f99b687bd9aabcd
$ curl -sfL https://get.k3s.io | K3S_URL=https://k3sserver:6443 K3S_TOKEN=mynodetoken sh -
```

* To get the cluster access file (_k3s.yaml_), from your host, run:

```
$ scp k3sserver:/etc/rancher/k3s/k3s.yaml .
```

* Update the [k3s](https://k3s.io/) server address (old value: 127.0.0.1) by the new hostname:

```
$ sed -i '' "s/127.0.0.1/k3sserver/" k3s.yaml
```

* Check the [k3s](https://k3s.io/) cluster:

```
$ export KUBECONFIG=$PWD/k3s.yaml
$ kubectl get nodes
NAME        STATUS   ROLES                  AGE   VERSION
k3sserver   Ready    control-plane,master   21d   v1.25.6+k3s1
k3snode1    Ready    <none>                 21d   v1.25.6+k3s1
k3snode2    Ready    <none>                 21d   v1.25.6+k3s1
```

> 📄 These instructions are based on the [k3s](https://k3s.io/) website: https://docs.k3s.io/quick-start.

## Deploy Coder

* Create a namespace for [Coder](https://coder.com/), such as `coder`:

```console
$ kubectl create namespace coder
namespace/coder created
```

* Deploy PostgreSQL to the [k3s](https://k3s.io/) cluster from [Bitnami](https://bitnami.com/) repository:

```console
$ helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories

$ helm install coder-db bitnami/postgresql \
    --namespace coder \
    --set auth.username=coder \
    --set auth.password=coder \
    --set auth.database=coder \
    --set persistence.size=10Gi

NAME: coder-db
LAST DEPLOYED: Wed Feb  31 09:53:58 2666
NAMESPACE: coder
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 11.1.28
APP VERSION: 14.2.0

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    coder-db-postgresql.coder.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_ADMIN_PASSWORD=$(kubectl get secret --namespace coder coder-db-postgresql -o jsonpath="{.data.postgres-password}" | base64 --decode)

To get the password for "coder" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace coder coder-db-postgresql -o jsonpath="{.data.password}" | base64 --decode)

To connect to your database run the following command:

    kubectl run coder-db-postgresql-client --rm --tty -i --restart='Never' --namespace coder --image docker.io/bitnami/postgresql:14.2.0-debian-10-r88 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host coder-db-postgresql -U coder -d coder -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace coder svc/coder-db-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U coder -d coder -p 5432
```

* Check if a PostgreSQL pod has been created:

```console
$ kubectl get pods --namespace coder
NAME                     READY   STATUS    RESTARTS   AGE
coder-db-postgresql-0    1/1     Running   0          22d
```

The cluster-internal DB URL for the PostgreSQL database is:

```
postgres://coder:coder@coder-db-postgresql.coder.svc.cluster.local:5432/coder?sslmode=disable
```

* Create a secret with the database URL:

```
$ kubectl create secret generic coder-db-url -n coder --from-literal=url="postgres://coder:coder@coder-db-postgresql.coder.svc.cluster.local:5432/coder?sslmode=disable"
```

* Add the Coder Helm repository:

```
$ helm repo add coder-v2 https://helm.coder.com/v2
```

* Create a [values.yaml](coder/values.yaml) with the configuration settings you wouldd like for your deployment. Update the content following the `# TODO` comments:

```yaml
coder:
  env:
    - name: CODER_PG_CONNECTION_URL
      valueFrom:
        secretKeyRef:
          name: coder-db-url
          key: url

    - name: CODER_ACCESS_URL
      # TODO
      value: "https://coder.mydomain.com"
    - name: CODER_WILDCARD_ACCESS_URL
      # TODO
      value: "*.coder.mydomain.com"

  service:
    enable: true
    type: ClusterIP
    sessionAffinity: ClientIP
    externalTrafficPolicy: Cluster
    loadBalancerIP: ""
    annotations: {}

  ingress:
    enable: true
    className: ""
    # TODO
    host: "coder.mydomain.com"
    # TODO
    wildcardHost: "*.coder.mydomain.com"
    annotations: {}
    tls:
      enable: false
      secretNames: ""
      wildcardSecretName: ""
```

Service will be configured as a ClusterIP. We configure Ingress to request the host (`coder.mydomain.com`) HTTP queries.

* Install the HELM chart in your [k3s](https://k3s.io/) cluster:

```console
$ helm install coder coder-v2/coder --namespace coder --values values.yaml
...
```

* Check the pods into the `coder` namespace:

```console
$ kubectl get pods --namespace coder
NAME                     READY   STATUS    RESTARTS   AGE
coder-db-postgresql-0    1/1     Running   0          22d
coder-59c6bc9c77-6f2wj   1/1     Running   0          9m47s
```

> 📄 These instructions are based on the [Coder](https://coder.com/) website: https://coder.com/docs/v2/latest/install/kubernetes

## Deploy Reverse Proxy

As mentioned in the introduction, a Reverse Proxy will be deployed outside of your Kubernetes cluster.

The Reverse Proxy will also be in charge of managing SSL/TLS certificates. Let's describe how to generate certificate with LetsEncrypt.

* Connect to the server node (`k3sserver`) and install Certbot:

```console
$ sudo apt-get update 
$ sudo apt-get install certbot -y
```

* Create the SSL/TLS certificate:

```console
$ sudo certbot certonly --agree-tos -m YOUR_EMAIL --manual --preferred-challenges=dns -d 'coder.mydomain.com' -d '*.coder.mydomain.com' -v
...
```

* Copy the SSL/TLS certificate files (_fullchain.pem_ and _privkey.pem_) into a directory (i.e. _/ssl_):

```console
$ mkdir /ssl
$ cp /etc/letsencrypt/live/coder.mydomain.com-0001/fullchain.pem privkey.pem /ssl
```

* Generate _dhparams.pem_ file:

```console
$ cd /ssl
$ openssl dhparam -out dhparams.pem 4096
```

We need to configure our Kubernetes cluster to update HTTP and HTTPS listen ports.

* Connect to the server node (`k3sserver`) and create _/var/lib/rancher/k3s/server/manifests/traefik-config.yaml_ file with the following content:

```yaml
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ports:
      web:
        exposedPort: 8080
      websecure:
        exposedPort: 8443
```

* Apply this configuration:

```console
$ kubectl apply -f /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
```

We suppose [Docker](https://www.docker.com/) is installed in the server node (`k3sserver`). 

* Create a Docker network called `reverseproxynetwork`:

```
$ docker network create reverseproxynetwork
```

Two Reverse Proxy will be presented: NGINX and Apache HTTP. Choose only ONE at your convenience.

### NGINX

* Connect to the server node (`k3sserver`).

* Create _nginx_ configuration files directory.

```console
$ mkdir ~/nginx
``` 

* Create NGINX configuration file [~/nginx/conf/coder.conf](nginx/conf/coder.conf) with the following content.

```
server {
    listen       80;
    listen       [::]:80;
    server_name  *.coder.mydomain.com;
    return	 301 https://$host$request_uri;
}

server {
   listen       443 ssl;
   server_name  *.coder.mydomain.com;

   ssl_protocols TLSv1.2 TLSv1.3;
   ssl_certificate /ssl/fullchain.pem;
   ssl_certificate_key /ssl/privkey.pem;
   ssl_dhparam /ssl/dhparam.pem;
   ssl_ecdh_curve secp384r1;
   ssl_prefer_server_ciphers on;
   ssl_ciphers EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH;

   location / {
       proxy_pass http://k3sserver:8080/;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "Upgrade";
       proxy_set_header Host $host;
   }
}
```

* Create [~/nginx/docker-compose.yaml](nginx/docker-compose.yaml) with the following content:

```yaml
services:

  nginx:
    container_name: nginx
    image: nginx:latest
    volumes:
      - ./conf:/etc/nginx/conf.d
      - /ssl:/ssl
    restart: always
    ports:
      - "80:80"
      - "443:443"
    networks:
      - reverseproxynetwork

networks:
  reverseproxynetwork:
    name: reverseproxynetwork
    external: true
```

* Create and start NGINX container:

```console
$ cd ~/nginx
$ docker compose up -d
```

### Apache HTTP

* Connect to the server node (`k3sserver`).

* Create _apachehttp_ configuration files directory.

```console
$ mkdir ~/apachehttp
``` 

* Create Apache HTTP configuration file [~/apachehttp/conf/coder.conf](apachehttp/conf/coder.conf) with the following content.

```
<VirtualHost *:443>
	SSLEngine On
	SSLProxyEngine on

    SSLProxyVerify none
    SSLProxyCheckPeerCN off
    SSLProxyCheckPeerName off
    SSLProxyCheckPeerExpire off

    SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
    SSLHonorCipherOrder On
    SSLCipherSuite "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4 !SHA1 !SHA256 !SHA384"
    SSLCompression off

    # HSTS (http://fr.wikipedia.org/wiki/HTTP_Strict_Transport_Security)
    Header unset Strict-Transport-Security
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    RequestHeader set X-Forwarded-Proto https"
    RequestHeader set X-Forwarded-Port "443"

	# Certificat
    SSLCertificateFile /ssl/fullchain.pem
    SSLCertificateKeyFile /ssl/privkey.pem

	ServerName *.coder.mydomain.com

	ProxyPreserveHost On
	ProxyRequests off
    ProxyPass / http://k3sserver:8080/ upgrade=any
    ProxyPassReverse / http://k3sserver:8080/

  	RewriteEngine on

  	RewriteCond %{HTTP:Connection} Upgrade [NC]
  	RewriteCond %{HTTP:Upgrade} websocket [NC]
  	RewriteRule /(.*) ws://k3sserver:8080/$1 [P,L]

	# Custom log file for SSL
  	ErrorLog /var/log/apachehttp/coder/error.log
  	CustomLog /var/log/apachehttp/coder/access.log combined
</VirtualHost>

<VirtualHost *:80>
	ServerName *.coder.mydomain.com
	RewriteEngine On
	RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
</VirtualHost>
```

* Copy and update at your convenience [~/apachehttp/httpd.conf](apachehttp/httpd.conf).

* Create [~/apachehttp/docker-compose.yaml](apachehttp/docker-compose.yaml) with the following content:

```yaml
services:

  httpd:
    container_name: httpd
    image: httpd:latest
    volumes:
      - ./httpd.conf:/usr/local/apache2/conf/httpd.conf
      - ./conf:/usr/local/apache2/conf/sites
      - /ssl:/ssl
    restart: always
    ports:
      - "80:80"
      - "443:443"
    networks:
      - reverseproxynetwork

networks:
  reverseproxynetwork:
    name: reverseproxynetwork
    external: true
```

> ℹ️ the Git repository supplies _httpd.conf_ 

* Create and start Apache HTTP container:

```
$ docker compose up -d
```

## Run

* Open your favorite web browser to https://coder.mydomain.com