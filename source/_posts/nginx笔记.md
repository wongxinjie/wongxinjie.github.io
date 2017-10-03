title: nginx笔记
date: 2017-03-15 22:40:34
categories: nginx
tags:
    - 工作笔记
---

近日看了一下如何用nginx作为web server来配合gunicorn将Python Web项目发布起来，顺便做一下笔记。

##### configuring gunicorn

在生产环境中是不能用Django/Flask自带的wsgi应用服务器的，可以选择uwsgi、FastCGI或者是gunicorn，我用的是gunicorn，配置方便。现在假设项目虚拟环境、数据库等等已经配置好了，现在跑gunicorn

```
cd $PROJECT
" 假设存在website/wsig.py
gunicorn -b 127.0.0.1:8001 -w 17 website.wsgi
```

一般workers数目按照以下公式计算：

```
" cat /proc/cpuinfo| grep processor | wc -l
" 可用上面的命令计算几个cpu_core

workers = 2 * CPU_cores + 1
```

更多关于[gunicorn](http://gunicorn.org/#docs) 可详细读文档，一般gunicorn程序我们都会用supervisor来管理。

<!--more-->

##### configuring site cofig

假设nginx等已经配置好了，现在之需要在/etc/nginx/conf.d/目录下增加一份配置即可。一个比较全的例子:

```
  # 很重要的虚拟主机配置
    server {
        listen       80;
        server_name  itoatest.example.com;
        root   /apps/oaapp;

        charset utf-8;
        access_log  logs/host.access.log  main;

        #对 / 所有做负载均衡+反向代理
        location / {
            root   /apps/oaapp;
            index  index.jsp index.html index.htm;

            proxy_pass        http://backend;  
            proxy_redirect off;
            # 后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy_set_header  Host  $host;
            proxy_set_header  X-Real-IP  $remote_addr;  
            proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;

        }

        #静态文件，nginx自己处理，不去backend请求tomcat
        location  ~* /download/ {  
            root /apps/oa/fs;  

        }
        location ~ .*\.(gif|jpg|jpeg|bmp|png|ico|txt|js|css)$   
        {   
            root /apps/oaapp;   
            expires      7d; 
        }
        location /nginx_status {
            stub_status on;
            access_log off;
            allow 192.168.10.0/24;
            deny all;
        }

        location ~ ^/(WEB-INF)/ {   
            deny all;   
        }
        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
```

对于类似Django项目我们不需要这么复杂的配置，简单几行就行了：

```
server {
    listen       80;
    server_name  a.site.com;

    access_log  /var/log/nginx/log/a.site.access.log  main;
    error_log  /var/log/nginx/log/a.site.error.log  error;

    location ^~ /static/ {
        autoindex on;
        root  $PROJECT/;
    }

    location / {
         proxy_pass         http://127.0.0.1:8001;
         proxy_redirect     off;
         proxy_set_header   Host $host;
         proxy_set_header   X-Real-IP $remote_addr;
         proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_set_header   X-Forwarded-Host $server_name;
    }
}
```

其中要注意的是，假如静态链接指明了static开头， location static一节的目录只需指定到项目层级（带/）。

然后重启nginx:

```
sudo nginx -s reload
" 几个常用的
sudo service nginx stop
sudo service nginx start
sudo service nginx restart
```

##### read more

* [nginx documentation](http://nginx.org/en/docs/)


* [nginx的配置、虚拟主机、负载均衡和反向代理（2）](https://www.zybuluo.com/phper/note/90310)

* [nginx服务器安装及配置文件详解](https://segmentfault.com/a/1190000002797601)

* [How to Deploy Python WSGI Apps Using Gunicorn HTTP Server Behind Nginx](https://www.digitalocean.com/community/tutorials/how-to-deploy-python-wsgi-apps-using-gunicorn-http-server-behind-nginx)

  ​