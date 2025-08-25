### 简介
前端使用nginx代理，将不同前缀的请求转发给不同的服务。
由于nginx不能直接获取系统的环境变量，这里使用envsubst来解决这个问题。

### nginx.conf
```
#user  nobody;
worker_processes  4;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  5000;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       ${FRONTEND_PORT};
        server_name  localhost;#填写你的宿主机ip

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   /usr/local/project/frontend;
            index  index.html;
        }

        location ^~ /decryptor/ {
            proxy_pass   http://127.0.0.1:${BACKEND_PORT}/;
        }

        location ^~ /eaglets/ {
            proxy_pass   ${EAGLETS_IS_ADDR};
        }


        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }

}
```

### Dockerfile
```dockerfile
FROM docker.servicewall.cn/debian:bullseye-slim

WORKDIR /usr/local/
ARG JDK8_VERSION

COPY frontend /usr/local/project/frontend
RUN  apt-get update && apt-get install -y nginx gettext
COPY nginx.conf /usr/local/nginx.conf

COPY backend/target/decryptor-2.1.2.RELEASE.jar /usr/local/project/backend/decryptor.jar
COPY backend/file /usr/local/file
ADD https://res-download.s3.cn-northwest-1.amazonaws.com.cn/tmp/jdk8/jdk-8u401-linux-$JDK8_VERSION.tar.gz /opt
RUN tar -zxvf /opt/jdk-8u401-linux-$JDK8_VERSION.tar.gz -C /opt && \
echo 'export JAVA_HOME=/opt/jdk1.8.0_401/' >> /etc/profile && \
echo 'export JRE_HOME=/opt/jdk1.8.0_401/jre' >> /etc/profile && \
echo 'export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib:$JRE_HOME/lib' >> /etc/profile && \
echo 'export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin' >> /etc/profile && \
echo 'source /etc/profile' >> /root/.bashrc && \
rm /opt/jdk-8u401-linux-$JDK8_VERSION.tar.gz


ARG GIT_HASH
LABEL org.opencontainers.image.revision=$GIT_HASH

COPY start.sh /usr/local/start.sh
CMD ["/bin/bash", "start.sh"]
```

### 启动脚本
```bash
ulimit -n 65535
source /etc/profile
envsubst < /usr/local/nginx.conf > /etc/nginx/nginx.conf
service nginx start
java -jar /usr/local/project/backend/decryptor.jar --server.port=${BACKEND_PORT}
```