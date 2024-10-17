### 使用nginx实现负载均衡

---

- 1、dockerfile

```dockerfile
FROM docker.servicewall.cn/debian:bullseye-slim

WORKDIR /usr/local/

RUN  apt-get update && apt-get install -y nginx gettext
COPY nginx.conf /usr/local/nginx.conf
COPY start.sh /usr/local/start.sh

CMD ["/bin/bash", "start.sh"]
```

- 2、nginx.conf

```
worker_processes  4;

events {
    worker_connections  5000;
}

error_log nginx-error.log info;
http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    upstream  nodes {
    	${NODES}
    }

    server {
        listen       ${LISTEN_PORT};
        server_name  127.0.0.1;


        location / {
            root   html;
            proxy_pass http://nodes;
            proxy_connect_timeout 3s;
            proxy_read_timeout 5s;
            proxy_send_timeout 3s;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

}
```

- 3、start.sh

```shell
ulimit -n 65535
source /etc/profile
envsubst < /usr/local/nginx.conf > /etc/nginx/nginx.conf
cat /etc/nginx/nginx.conf
nginx -g 'daemon off;'
```