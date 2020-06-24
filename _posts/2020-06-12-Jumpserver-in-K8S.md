---
layout: post
title: Jumpserver容器化部署手册
date: 2020-06-12
Author: gbren 
tags: [JumpServer]
comments: true
toc: true
pinned: true
---

## 组件说明

- Jumpserver     为管理后台, 管理员可以通过 Web 页面进行资产管理、用户管理、资产授权等操作, 用户可以通过 Web 页面进行资产登录, 文件管理等操作
- koko 为 SSH Server 和 Web Terminal Server 。用户可以使用自己的账户通过 SSH 或者 Web Terminal 访问 SSH 协议和 Telnet 协议资产
- Luna 为 Web Terminal Server 前端页面, 用户使用 Web Terminal 方式登录所需要的组件
- Guacamole 为 RDP 协议和 VNC 协议资产组件, 用户可以通过 Web Terminal 来连接 RDP 协议和 VNC 协议资产 (暂时只能通过 Web Terminal 来访问)

## 端口说明

- Jumpserver     默认 Web 端口为 8080/tcp, 默认 WS 端口为 8070/tcp, 配置文件 jumpserver/config.yml
- koko 默认 SSH 端口为 2222/tcp, 默认 Web Terminal 端口为 5000/tcp 配置文件在 koko/config.yml
- Guacamole 默认端口为 8080tcp, 配置文件     /config/tomcat9/conf/server.xml
- Nginx 默认端口为 80/tcp
- Redis 默认端口为 6379/tcp

## 镜像制作

### Jumpserver镜像

官方镜像不可用，需下载源码包自行构建，GitHub下载Jumpserver-1.5.4 Release包；源码下载地址：https://github.com/jumpserver/jumpserver/archive/1.5.4.tar.gz

解压源码包，并使用以下Dockerfile、entrypoint.sh替换源码中的文件，执行docker build命令。

**Dockerfile**

```dockerfile
FROM registry.fit2cloud.com/public/python:v3
MAINTAINER Jumpserver Team <ibuler@qq.com>

WORKDIR /opt/jumpserver
RUN useradd jumpserver

COPY ./requirements /tmp/requirements

RUN yum -y install epel-release && \
      echo -e "[mysql]\nname=mysql\nbaseurl=https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql57-community-el6/\ngpgcheck=0\nenabled=1" > /etc/yum.repos.d/mysql.repo
RUN cd /tmp/requirements && yum -y install $(cat rpm_requirements.txt)
RUN cd /tmp/requirements && pip install --upgrade pip setuptools==33.1.1 && \
    pip install -i https://mirrors.aliyun.com/pypi/simple/ -r requirements.txt || pip install -r requirements.txt
RUN mkdir -p /root/.ssh/ && echo -e "Host *\n\tStrictHostKeyChecking no\n\tUserKnownHostsFile /dev/null" > /root/.ssh/config

COPY . /opt/jumpserver
# RUN echo > config.yml
VOLUME /opt/jumpserver/data
VOLUME /opt/jumpserver/logs

ENV LANG=zh_CN.UTF-8
ENV LC_ALL=zh_CN.UTF-8

EXPOSE 8070
EXPOSE 8080
ENTRYPOINT ["./entrypoint.sh"]
```

**entrypoint.sh**

```bash
#!/bin/bash
function cleanup()
{
    local pids=`jobs -p`
    if [[ "${pids}" != ""  ]]; then
        kill ${pids} >/dev/null 2>/dev/null
    fi
}

SECRET_KEY=${SECRET_KEY:=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 50`}
BOOTSTRAP_TOKEN=${BOOTSTRAP_TOKEN:=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 16`}
DEBUG=${DEBUG:="false"}
LOG_LEVEL=${LOG_LEVEL:="ERROR"}
SESSION_EXPIRE_ABC=${SESSION_EXPIRE_ABC:="true"}
DB_HOST=${DB_HOST:="127.0.0.1"}
DB_PORT=${DB_PORT:="3306"}
DB_USER=${DB_USER:="jumpserver"}
DB_PASSWORD=${DB_PASSWORD:=""}
DB_NAME=${DB_NAME:="jumpserver"}
REDIS_HOST=${REDIS_HOST:="127.0.0.1"}
REDIS_PORT=${REDIS_PORT:="6379"}

if [ ! -f "/opt/jumpserver/config.yml" ]; then
    cp /opt/jumpserver/config_example.yml /opt/jumpserver/config.yml
    sed -i "s/SECRET_KEY:/SECRET_KEY: $SECRET_KEY/g" /opt/jumpserver/config.yml
    sed -i "s/BOOTSTRAP_TOKEN:/BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN/g" /opt/jumpserver/config.yml
    sed -i "s/# DEBUG: true/DEBUG: $DEBUG/g" /opt/jumpserver/config.yml
    sed -i "s/# LOG_LEVEL: DEBUG/LOG_LEVEL: $LOG_LEVEL/g" /opt/jumpserver/config.yml
    sed -i "s/# SESSION_EXPIRE_AT_BROWSER_CLOSE: false/SESSION_EXPIRE_AT_BROWSER_CLOSE: $SESSION_EXPIRE_ABC/g" /opt/jumpserver/config.yml
    sed -i "s/DB_ENGINE: mysql/DB_ENGINE: $DB_ENGINE/g" /opt/jumpserver/config.yml
    sed -i "s/DB_HOST: 127.0.0.1/DB_HOST: $DB_HOST/g" /opt/jumpserver/config.yml
    sed -i "s/DB_PORT: 3306/DB_PORT: $DB_PORT/g" /opt/jumpserver/config.yml
    sed -i "s/DB_USER: jumpserver/DB_USER: $DB_USER/g" /opt/jumpserver/config.yml
    sed -i "s/DB_PASSWORD: /DB_PASSWORD: $DB_PASSWORD/g" /opt/jumpserver/config.yml
    sed -i "s/DB_NAME: jumpserver/DB_NAME: $DB_NAME/g" /opt/jumpserver/config.yml
    sed -i "s/REDIS_HOST: 127.0.0.1/REDIS_HOST: $REDIS_HOST/g" /opt/jumpserver/config.yml
    sed -i "s/REDIS_PORT: 6379/REDIS_PORT: $REDIS_PORT/g" /opt/jumpserver/config.yml
    sed -i "s/# REDIS_PASSWORD: /REDIS_PASSWORD: $REDIS_PASSWORD/g" /opt/jumpserver/config.yml
fi

if [[ "$REDIS_PASSWORD" != "" ]]; then
    sed -i "s/# REDIS_PASSWORD: /REDIS_PASSWORD: $REDIS_PASSWORD/g" /opt/jumpserver/config.yml
fi

if [[ "$AUTH_LDAP_SYNC_IS_PERIODIC" == "True" ]]; then
    AUTH_LDAP_SEARCH_PAGED_SIZE=${AUTH_LDAP_SEARCH_PAGED_SIZE:="1000"}
    AUTH_LDAP_SYNC_INTERVAL=${AUTH_LDAP_SYNC_INTERVAL:="12"}
    AUTH_LDAP_SYNC_CRONTAB=${AUTH_LDAP_SYNC_CRONTAB:="* 6 * * *"}
    sed -i "s/# AUTH_LDAP_SYNC_IS_PERIODIC: True/AUTH_LDAP_SYNC_IS_PERIODIC: $AUTH_LDAP_SYNC_IS_PERIODIC/g" /opt/jumpserver/config.yml
    sed -i "s/# AUTH_LDAP_SYNC_INTERVAL: 12/AUTH_LDAP_SYNC_INTERVAL: $AUTH_LDAP_SYNC_INTERVAL/g" /opt/jumpserver/config.yml
    sed -i "s/# AUTH_LDAP_SYNC_CRONTAB: * 6 * * */AUTH_LDAP_SYNC_CRONTAB: $AUTH_LDAP_SYNC_CRONTAB/g" /opt/jumpserver/config.yml
fi

if [[ "$AUTH_LDAP_SEARCH_PAGED_SIZE" != "" ]]; then
    sed -i "s/# AUTH_LDAP_SEARCH_PAGED_SIZE: 1000/AUTH_LDAP_SEARCH_PAGED_SIZE: $AUTH_LDAP_SEARCH_PAGED_SIZE/g" /opt/jumpserver/config.yml
fi

if [[ "$AUTH_LDAP_USER_LOGIN_ONLY_IN_USERS" != "" ]]; then
    sed -i "s/# AUTH_LDAP_USER_LOGIN_ONLY_IN_USERS: False/AUTH_LDAP_USER_LOGIN_ONLY_IN_USERS: $AUTH_LDAP_USER_LOGIN_ONLY_IN_USERS/g" /opt/jumpserver/config.yml
fi

if [[ "$AUTH_LDAP_OPTIONS_OPT_REFERRALS" != "" ]]; then
    sed -i "s/# AUTH_LDAP_OPTIONS_OPT_REFERRALS: -1/AUTH_LDAP_OPTIONS_OPT_REFERRALS: $AUTH_LDAP_OPTIONS_OPT_REFERRALS/g" /opt/jumpserver/config.yml
fi

service="all"
if [[ "$1" != "" ]];then
    service=$1
fi

trap cleanup EXIT
if [[ "$1" == "bash" ]];then
    bash
else
    python jms start ${service}
fi
```



### koko镜像

镜像版本：jumpserver/jms_koko:1.5.4

### Guacamole镜像

镜像版本：jumpserver/jms_guacamole:1.5.4

### Nginx镜像

官方镜像即可，容器化部署时主要用作jumpserver前端工程及静态文件访问。

### Redis镜像

官方镜像即可。

## 集群部署

### 创建PVC

- kubectl apply -f jms-data-pvc.yaml：主要用于录像存放

**jms-data-pvc.yaml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jms-core-data-pvc
  namespace: ${namespace}
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
  storageClassName: nfs-client
 
```

 

- kubectl apply -f jms-log-pvc.yaml：主要用于日志存放

**jms-log-pvc.yaml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jms-core-log-pvc
  namespace: ${namespace}
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
  storageClassName: nfs-client
 
```

 

- kubectl apply -f jms-luna-pvc.yaml：用于前端工程luna存放

**jms-luna-pvc.yaml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jms-static-luna-pvc
  namespace: ${namespace}
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
  storageClassName: nfs-client
```

 

### 部署Jumpserver核心服务

- kubectl apply -f jms-core-deployment.yaml

**jms-core-deployment.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: jms-core
  name: jms-core
  namespace: ${namespace}
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: jms-core
    spec:
      containers:
      - env:
        - name: DB_HOST
          value: ${DB_HOST}
        - name: DB_USER
          value: jumpserver
        - name: DB_NAME
          value: jumpserver
        - name: DB_PASSWORD
          value: ${DB_PASSWORD}
        - name: DEBUG
          value: "true"
        - name: REDIS_HOST
          value: redis
        - name: LOG_LEVEL
          value: DEBUG
        - name: SECRET_KEY
          value: ${SECRET_KEY}
        - name: BOOTSTRAP_TOKEN
          value: ${BOOTSTRAP_TOKEN}
        image: ${docker_repo}
        name: jms-core
        ports:
        - containerPort: 8080
          name: jms-core-web
          protocol: TCP
        - containerPort: 8070
          name: jms-core-ws
          protocol: TCP
        resources:
          limits:
            cpu: "2"
            memory: 4Gi
          requests:
            cpu: "2"
            memory: 4Gi
        volumeMounts:
        - mountPath: /opt/jumpserver/data
          name: jms-core-data-pv
        - mountPath: /opt/jumpserver/logs
          name: jms-core-log-pv
      imagePullSecrets:
      - name: docker-repo
      volumes:
      - name: jms-core-data-pv
        persistentVolumeClaim:
          claimName: jms-core-data-pvc
      - name: jms-core-log-pv
        persistentVolumeClaim:
          claimName: jms-core-log-pvc
```

 

- kubectl     apply -f jms-core-service.yaml

**jms-core-service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: jms-core
  name: jms-service
  namespace: ${namespace}
spec:
  ports:
    - port: 8080
      name: jms-core-web
      targetPort: jms-core-web
    - port: 8070
      name: jms-core-ws
      targetPort: jms-core-ws
  selector:
    app: jms-core
 
```

 

### 部署Nginx：用于访问jumpserver静态文件及前端工程luna

- kubectl create configmap jms-static-etc --from-file nginx.conf：根据以下nginx.conf文件创建Nginx配置 configmap

**nginx.conf**

```nginx
user  nginx;
worker_processes  auto;
 
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
 
 
events {
    worker_connections  1024;
}
 
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
 
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
 
    access_log  /var/log/nginx/access.log  main;
 
    sendfile        on;
    # tcp_nopush     on;
 
    keepalive_timeout  65;
 
    # 关闭版本显示
    server_tokens off;
 
    # include /etc/nginx/conf.d/*.conf;
 
    server {
        listen 80;
        server_name ${server_name}
 
        client_max_body_size 100m;
 
        location /luna/ {
            try_files $uri / /index.html;
            alias /opt/luna/;  # luna 路径
        }
 
        location /media/ {
            add_header Content-Encoding gzip;
            root /opt/jumpserver/data/;  # 录像位置, 如果修改安装目录, 此处需要修改
        }  
 
        location /static/ {
            root /opt/jumpserver/data/;  # 静态资源, 如果修改安装目录, 此处需要修改
        }
    }
}
 
```



- kubectl apply -f jms-static-deployment.yaml

**jms-statick-deployment.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jms-static
  namespace: ${namespace} 
  labels:
    app: jms-static
spec:
  replicas: 1
  template:
    metadata:
      labels:
       app: jms-static
    spec:
      containers:
       - name: jms-static
         image: ${docker-repo}
         volumeMounts:
          - mountPath: /opt/luna/
            name: jms-static-luna-pv
          - mountPath: /etc/nginx/nginx.conf
            subPath: nginx.conf
            name: jms-static-nginx
          - mountPath: /opt/jumpserver/data/
            name: jms-core-data-pv
         ports:
          - containerPort: 80
         resources:
           limits:
             cpu: "1"
             memory: 1Gi
           requests:
             cpu: "1"
             memory: 1Gi
      imagePullSecrets:
        - name: docker-repo
      volumes:
         - name: jms-static-luna-pv
           persistentVolumeClaim:
            claimName: jms-static-luna-pvc
         - name: jms-core-data-pv
           persistentVolumeClaim:
            claimName: jms-core-data-pvc
         - name: jms-static-nginx
           configMap:
             name: jms-static-etc
             items:
             - key: nginx.conf
               path: nginx.conf 
```

 

### 部署KoKo组件：SSH Server 和 Web Terminal Server

- kubectl apply -f jms-koko-deployment.yaml

**jms-koko-deployment.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jms-koko
  namespace: ${namespace}
  labels:
    app: jms-koko
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: jms-koko
    spec:
      containers:
        - name: jms-koko
          image: ${docker-repo}
          ports:
            - containerPort: 2222
              name: jms-koko-ssh
            - containerPort: 5000
              name: jms-koko-ws
          resources: 
            limits:
              cpu: "1"
              memory: 1Gi
            requests:
              cpu: "1"
              memory: 1Gi
          env:
            - name: CORE_HOST
              value: "http://jms-service:8080/"
            - name: BOOTSTRAP_TOKEN
              value: ${BOOTSTRAP_TOKEN}
      imagePullSecrets:
        - name: docker-repo
      nodeSelector:
        cloud: dmz
 
 
```

 

- kubectl apply -f jms-koko-service.yaml

**jms-koko-service.yaml**

```yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: jms-koko
  name: jms-koko-ssh
  namespace: ${namespace}
spec:
  ports:
    - port: 22222 #自定义端口
      name: jms-koko-ssh
      targetPort: jms-koko-ssh
  selector:
    app: jms-koko
  type: NodePort
 
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: jms-koko
  name: jms-koko-web
  namespace: ${namespace}
spec:
  ports:
    - port: 5000
      name: jms-koko-ws
      targetPort: jms-koko-ws
  selector:
    app: jms-koko
  type: ClusterIP
```

###   部署Guacamole组件：RDP 协议和 VNC 协议资产组件

- kubectl  apply -f jms-gua-deployment.yaml

**jms-gua-deployment.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jms-gua
  namespace: ${namespace} 
  labels:
    app: jms-gua
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: jms-gua
    spec:
      containers:
        - name: jms-guacamole
          image: ${docker-repo}
          ports:
            - containerPort: 8080
              name: jms-guardc
          resources: 
            limits:
              cpu: "1"
              memory: 1Gi
            requests:
              cpu: "1"
              memory: 1Gi
          env:
            - name: JUMPSERVER_SERVER
              value: "http://jms-service:8080/"
            - name: BOOTSTRAP_TOKEN
              value: ${BOOTSTRAP_TOKEN}
            - name: JUMPSERVER_KEY_DIR
              value: /config/guacamole/key
      imagePullSecrets:
        - name: docker-repo
 
```

 

- kubectl apply -f jms-gua-service.yaml

**jms-gua-service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: jms-gua
  name: jms-gua
  namespace: ${namespace} 
spec:
  ports:
    - port: 8080
      name: gua-web
      targetPort: jms-guardc
  selector:
    app: jms-gua
```

### 创建Ingress：服务入口

- kubectl apply -f jumpserver-ingress.yaml

**jumpserver-ingress.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jms
  namespace: ${namespace}
spec:
  rules:
  - host: jump.xxx.com   #自定义
    http:
      paths:
      - backend:
          serviceName: jms-service
          servicePort: 8080
        path: /
      - backend:
          serviceName: jms-service
          servicePort: 8070
        path: /ws/
      - backend:
          serviceName: jms-koko-web
          servicePort: 5000
        path: /koko/
      - backend:
          serviceName: jms-static-service
          servicePort: 80
        path: /luna/
      - backend:
          serviceName: jms-static-service
          servicePort: 80
        path: /media/
      - backend:
          serviceName: jms-static-service
          servicePort: 80
        path: /static/
  tls:
  - hosts:
    - jump.xxx.com
    secretName: xxx-tls   #自定义
status:
  loadBalancer: {}
```

由于Guacamole组件为独立部署的web应用，Ingress无法直接转发；需要创建middleware和ingressroute进行转发。

- kubectl apply -f jumpserver-traefik-middleware.yaml

**jumpserver-traefik-middleware.yaml**

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: replace-gua-path
  namespace: ${namespace}
spec:
  stripPrefix:
    prefixes:
    - /guacamole
```

- kubectl apply -f jumpserver-traefik-ingressroute.yaml

**jumpserver-traefik-ingressroute.yaml**  

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: jump-test
  namespace: ${namespace}
spec:
  entryPoints:
  - web
  routes:
  - kind: Rule
    match: Host(`jump.xxx.com`) && PathPrefix(`/guacamole`)
    middlewares:
    - name: replace-gua-path
    services:
    - kind: Service
      name: jms-gua
      port: 8080
```