---
layout: post
title: Nginx config for Yii 2 Advanced App Template (subdomains)
description: Centos7 下 Yii 2 的 Nginx 配置
tags: [Nginx, Yii 2]
---

# frontend
```ruby
	server {
	    listen        80;
	    server_name   yii2.lo;
	    server_tokens off;

	    client_max_body_size 128M;
	    charset       utf-8;

	    access_log    /var/log/nginx/yii2-access.log main buffer=50k;
	    error_log     /var/log/nginx/yii2-error.log;

	    set           $host_path      "/srv/http/yii2/public";
	    set           $yii_bootstrap  "index.php";

	    root          $host_path/frontend/web;
	    index         $yii_bootstrap;

	    location / {
	        try_files $uri $uri/ /$yii_bootstrap?$args;
	    }

	    location ~ \.php$ {
	        try_files $uri =404;

	        fastcgi_split_path_info ^(.+\.php)(/.+)$;
	        fastcgi_index           $yii_bootstrap;

	        # Connect to php-fpm via socket
	        fastcgi_pass 127.0.0.1:9000;
	        # Dedian-like: unix:/var/run/php5-fpm.sock;

	        fastcgi_connect_timeout     30s;
	        fastcgi_read_timeout        30s;
	        fastcgi_send_timeout        60s;
	        fastcgi_ignore_client_abort on;
	        fastcgi_pass_header         "X-Accel-Expires";

	        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
	        fastcgi_param  PATH_INFO        $fastcgi_path_info;
	        fastcgi_param  HTTP_REFERER     $http_referer;
	        include fastcgi_params;
	    }

	    location ~* \.(js|css|less|png|jpg|jpeg|gif|ico|woff|ttf|svg|tpl)$ {
	        expires 24h;
	        access_log off;
	    }

	    location = /favicon.ico {
	        log_not_found off;
	        access_log off;
	    }

	    location = /robots.txt {
	        log_not_found off;
	        access_log off;
	    }

	    location ~ /\. {
	        deny all;
	        access_log off;
	        log_not_found off;
	    }
	}
```
# backend
```ruby
	server {
	    listen        80;
	    server_name   admin.yii2.lo;
	    server_tokens off;

	    client_max_body_size 128M;
	    charset       utf-8;

	    access_log    /var/log/nginx/yii2-access.log main buffer=50k;
	    error_log     /var/log/nginx/yii2-error.log;

	    set           $host_path      "/srv/http/yii2/public";
	    set           $yii_bootstrap  "index.php";

	    root          $host_path/backend/web;
	    index         $yii_bootstrap;

	    location / {
	        try_files $uri $uri/ /$yii_bootstrap?$args;
	    }

	    location ~ \.php$ {
	        try_files $uri =404;

	        fastcgi_split_path_info ^(.+\.php)(/.+)$;
	        fastcgi_index           $yii_bootstrap;

	        # Connect to php-fpm via socket
	        #fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
	        fastcgi_pass 127.0.0.1:9000;

	        fastcgi_connect_timeout     30s;
	        fastcgi_read_timeout        30s;
	        fastcgi_send_timeout        60s;
	        fastcgi_ignore_client_abort on;
	        fastcgi_pass_header         "X-Accel-Expires";

	        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
	        fastcgi_param  PATH_INFO        $fastcgi_path_info;
	        fastcgi_param  HTTP_REFERER     $http_referer;
	        include fastcgi_params;
	    }

	    location ~* \.(js|css|less|png|jpg|jpeg|gif|ico|woff|ttf|svg|tpl)$ {
	        expires 24h;
	        access_log off;
	    }

	    location = /favicon.ico {
	        log_not_found off;
	        access_log off;
	    }

	    location = /robots.txt {
	        log_not_found off;
	        access_log off;
	    }

	    location ~ /\. {
	        deny all;
	        access_log off;
	        log_not_found off;
	    }
	}
```