# openresty perf learning

## useage

* nginx.conf

```code
worker_processes  1;
user root;  
master_process off;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    root html;
    keepalive_timeout  65;
    gzip  on;
    upstream cluster1 {
        # simple round-robin
        server app2:80;
        server app3:80;
        check interval=1000 rise=2 fall=5 timeout=1000 type=http;
        check_http_send "HEAD / HTTP/1.0\r\n\r\n";
        check_http_expect_alive http_2xx http_3xx;
    }  
     upstream cluster2 {
        # simple round-robin
        server app2:80;
        server app3:80;
        check interval=1000 rise=2 fall=5 timeout=1000 type=http;
        check_http_send "HEAD / HTTP/1.0\r\n\r\n";
        check_http_expect_alive http_2xx http_3xx;
    } 
    server {
        listen       81;
        server_name  localhost;
        charset utf-8;
        default_type text/html;
        location / {
            index index.html;
        }
        location /status {
            healthcheck_status;
        }
        location /app.html {
            root html;
        }
        location /css/ {
            trim on;
            trim_js on;
            trim_css on;
            concat on;
            concat_max_files 20;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }    
    }
    server {
        listen       80;
        server_name  localhost;
        charset utf-8;
        default_type text/html;
        location / {
            proxy_set_header Host $http_host;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for; 
            client_body_buffer_size 10M;
            client_max_body_size 10G;
            proxy_buffers 1024 4k;
            proxy_read_timeout 300;
            proxy_connect_timeout 80;
            proxy_pass http://cluster1;
            footer "<!-- dalong demo-->";  
        }
        location /status {
            healthcheck_status;
        }
        location /app.html {
            root html;
        }
        location /css/ {
            trim on;
            trim_js on;
            trim_css on;
            concat on;
            concat_max_files 20;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }    
    }    
}
```
* Dockerfile

```code
FROM dalongrong/openresty-tengine:debug
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && apk update && apk upgrade
RUN apk add --no-cache perf
```

* docker-compose

```code
version: '3'
services:
  app:
    build: ./
    volumes:
      - "./nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf"
      - "./css/:/usr/local/openresty/nginx/html/"
    privileged: true
    cap_add:
       - ALL
    ports:
      - "80:80"
      - "81:81"
  app2:
    image: dalongrong/openresty-tengine:latest
  app3:
    image: dalongrong/openresty-tengine:latest
```

* perf record


```code
docker-compose exec app sh

sh -c " echo 0 > /proc/sys/kernel/kptr_restrict"
perf record -ag -F 99 1
```
