---
layout: post
title: CentOS 7 用户怎样安装 LNMP（Nginx+PHP+MySQL）
description: 关于 Nginx 这是一款免费、开源、高效的 HTTP 服务器，Nginx是以稳定著称，丰富的功能，结构简单，低资源消耗。本教程演示如何在CentOS 6.5服务器（适用于 CentOS 7）安装Nginx与PHP（通过php-fpm）和MySQL（MariaDB）。
---

>关于 Nginx （发音 “engine x”)这是一款免费、开源、高效的 HTTP 服务器，Nginx是以稳定著称，丰富的功能，结构简单，低资源消耗。本教程演示如何在CentOS 6.5服务器（适用于 CentOS 7）安装Nginx与PHP（通过php-fpm）和MySQL（MariaDB）。

![LNMP_logo](/public/images/linuxwebserver_logo.png)

### 1 先说一下

本文使用的主机名称： server1.example.com 和IP地址： 192.168.1.105。这些可能与你的计算机有所不同，注意进行修改。
### 2 使用外部仓库

Nginx不是从官方CentOS库安装，我们从 nginx 项目安装库安装，修改源：

```language-nginx
	vi /etc/yum.repos.d/nginx.repo
```

修改为：  

```language-nginx
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
```  

[Nginx Install](http://wiki.nginx.org/Install)

### 3 安装 MySQL

我们先安装MariaDB。一个免费的MySQL 分支。运行此命令：

```language-nginx
	yum install mariadb mariadb-server net-tools
```

然后我们创建MySQL系统启动链接（所以MySQL的自动启动时，系统启动）启动MySQL服务器：

```language-nginx
	systemctl enable mariadb.service
	systemctl start mariadb.service
```

现在检查网络启用。运行

```language-nginx
	netstat -tap | grep mysql
```

它应该显示出这样的内容：

```language-nginx
	[root@example ~]# netstat -tap | grep mysql
	tcp 0 0 0.0.0.0:mysql 0.0.0.0:* LISTEN 10623/mysqld
```
 

运行

```language-nginx
	mysql_secure_installation

	为用户设置根口令（否则，任何人都可以访问你的MySQL数据库！）：

	[root@example ~]# mysql_secure_installation
	/usr/bin/mysql_secure_installation: line 379: find_mysql_client: command not found

	NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
	SERVERS IN PRODUCTION USE! PLEASE READ EACH STEP CAREFULLY!

	In order to log into MariaDB to secure it, we’ll need the current
	password for the root user. If you’ve just installed MariaDB, and
	you haven’t set the root password yet, the password will be blank,
	so you should just press enter here.

	Enter current password for root (enter for none):
	OK, successfully used password, moving on…

	Setting the root password ensures that nobody can log into the MariaDB
	root user without the proper authorisation.

	Set root password? [Y/n] <– 回车
	New password: <– 输入ROOT密码
	Re-enter new password: <– 再输入一次ROOT密码
	Password updated successfully!
	Reloading privilege tables..
	… Success!

	By default, a MariaDB installation has an anonymous user, allowing anyone
	to log into MariaDB without having to have a user account created for
	them. This is intended only for testing, and to make the installation
	go a bit smoother. You should remove them before moving into a
	production environment.

	Remove anonymous users? [Y/n] <– 回车
	… Success!

	Normally, root should only be allowed to connect from ‘localhost’. This
	ensures that someone cannot guess at the root password from the network.

	Disallow root login remotely? [Y/n] <– 回车
	… Success!

	By default, MariaDB comes with a database named ‘test’ that anyone can
	access. This is also intended only for testing, and should be removed
	before moving into a production environment.

	Remove test database and access to it? [Y/n] <– 回车
	– Dropping test database…
	… Success!
	– Removing privileges on test database…
	… Success!

	Reloading the privilege tables will ensure that all changes made so far
	will take effect immediately.

	Reload privilege tables now? [Y/n] <– 回车
	… Success!

	Cleaning up…

	All done! If you’ve completed all of the above steps, your MariaDB
	installation should now be secure.

	Thanks for using MariaDB!

	[root@example ~]#

	[root@server1 ~]# mysql_secure_installation
```

### 4 安装 Nginx

Nginx可以作为一个包从nginx.org安装，运行：

```language-nginx
	yum install nginx
```

然后我们创建的系统启动nginx的链接和启动它：

```language-nginx
	systemctl enable nginx.service
	systemctl start nginx.service
```

有时，你会得到一个错误，如80端口已在使用中，错误消息会是这样的

```language-nginx
	[root@server1 ~]# service nginx start
	Starting nginx: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
	nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
	nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
	nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
	nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
	nginx: [emerg] still could not bind()
	[FAILED]
	[root@server1 ~]#
```

这就意味着有时在运行Apache服务。停止服务，进一步启动服务nginx如下

```language-nginx
	systemctl stop httpd.service
	yum remove httpd
	systemctl disable httpd.service

	systemctl enable nginx.service
	systemctl start nginx.service
```

开放的HTTP和HTTPS防火墙中的端口

```language-nginx
	firewall-cmd –permanent –zone=public –add-service=http
	firewall-cmd –permanent –zone=public –add-service=https
	firewall-cmd –reload
```

输出的shell结果将看起来像这样：

```language-nginx
	[root@example ~]# firewall-cmd –permanent –zone=public –add-service=http
	success
	[root@example ~]# firewall-cmd –permanent –zone=public –add-service=https
	success
	[root@example ~]# firewall-cmd –reload
	success
	[root@example ~]#
```

在你的Web服务器的IP地址或主机名称输入到浏览器（如HTTP：/ /192.168.1.105），你应该看到nginx的欢迎页面。
### 5 安装 PHP5

我们可以通过PHP-FPM使nginx的PHP5工作（PHP-FPM（FastCGI进程管理器）是一种替代PHP FastCGI执行一些额外的功能，支持任何规模大小，尤其是繁忙的站点很有用）。我们可以安装php-fpmtogether用PHP-CLI和一些PHP5的模块，如PHP，MySQL，你需要的，如果你想使用MySQL的PHP命令如下：

```language-nginx
	yum install php-fpm php-cli php-mysql php-gd php-ldap php-odbc php-pdo php-pecl-memcache php-pear php-mbstring php-xml php-xmlrpc php-mbstring php-snmp php-soap
```

APC是一个自由和开放的PHP操作码来缓存和优化PHP的中间代码。它类似于其他PHP操作码cachers，如eAccelerator和XCache。强烈建议有这些安装，以加快您的PHP页面。

我会从PHP PECL库中安装的APC。 PECL要求CentOS开发工具beinstalled编译APC包。

```language-nginx
	yum install php-devel
	yum groupinstall ‘Development Tools’
```

安装 APC

```language-nginx
	pecl install apc

	[root@example ~]# pecl install apc
	downloading APC-3.1.13.tgz …
	Starting to download APC-3.1.13.tgz (171,591 bytes)
	……………..done: 171,591 bytes
	55 source files, building
	running: phpize
	Configuring for:
	PHP Api Version: 20100412
	Zend Module Api No: 20100525
	Zend Extension Api No: 220100525
	Enable internal debugging in APC [no] : <– 回车
	Enable per request file info about files used from the APC cache [no] : <– 回车
	Enable spin locks (EXPERIMENTAL) [no] : <– 回车
	Enable memory protection (EXPERIMENTAL) [no] : <– 回车
	Enable pthread mutexes (default) [no] : <–回车
	Enable pthread read/write locks (EXPERIMENTAL) [yes] : <– 回车
	building in /var/tmp/pear-build-rootVrjsuq/APC-3.1.13
	……
```

然后打开 /etc/php.ini 并设置 cgi.fix_pathinfo=0:

```language-nginx
	vi /etc/php.ini

	[...]
	; cgi.fix_pathinfo provides *real* PATH_INFO/PATH_TRANSLATED support for CGI.  PHP's
	; previous behaviour was to set PATH_TRANSLATED to SCRIPT_FILENAME, and to not grok
	; what PATH_INFO is.  For more information on PATH_INFO, see the cgi specs.  Setting
	; this to 1 will cause PHP CGI to fix its paths to conform to the spec.  A setting
	; of zero causes PHP to behave as before.  Default is 1.  You should fix your scripts
	; to use SCRIPT_FILENAME rather than PATH_TRANSLATED.
	; http://www.php.net/manual/en/ini.core.php#ini.cgi.fix-pathinfo
	cgi.fix_pathinfo=0
	[...]
```

并添加行：

```language-nginx
	[...]
	extension=apc.so
```

在 /etc/php.ini 文件后面。

除此之外，为了避免这样的时区的错误：

```language-nginx
	[21-July-2014 10:07:08] PHP Warning: phpinfo(): It is not safe to rely on the system’s timezone settings. You are *required* to use the date.timezone setting or the date_default_timezone_set() function. In case you used any of those methods and you are still getting this warning, you most likely misspelled the timezone identifier. We selected ‘Europe/Berlin’ for ‘CEST/2.0/DST’ instead in /usr/share/nginx/html/info.php on line 2
```

… in /var/log/php-fpm/www-error.log 当你在浏览器中调用一个PHP脚本，你应该设置 date.timezone in /etc/php.ini:

```language-nginx
	[...]
	[Date]
	; Defines the default timezone used by the date functions
	; http://www.php.net/manual/en/datetime.configuration.php#ini.date.timezone
	date.timezone = "Europe/Berlin"
	[...]
```

您可以通过运行正确的时区支持您的系统：

```language-nginx
	cat /etc/sysconfig/clock

	[root@server1 nginx]# cat /etc/sysconfig/clock
	ZONE=”Europe/Berlin”
	[root@server1 nginx]#
```

接下来，创建系统启动链接的PHP-FPM并启动它：

```language-nginx
	systemctl enable php-fpm.service
	systemctl start php-fpm.service
```

PHP-FPM是一个守护进程（使用init脚本/etc/init.d/php-fpm) 运行在端口9000的FastCGI服务器。

 
### 6 配置 nginx

现在我们打开配置文件 /etc/nginx/nginx.conf ：

```language-nginx
	vi /etc/nginx/nginx.conf
```

配置是很容易理解（你可以了解更多配置信息： http://wiki.codemongers.com/NginxFullExample 、http://wiki.codemongers.com/NginxFullExample2)

首先（这是可选的），你可以增加工作进程的数量和设置keepalive_timeout到一个合理的值：

```language-nginx
	[...]
	worker_processes  4;
	[...]
	    keepalive_timeout  2;
	[...]
```

虚拟主机的定义 server {} 配置文件 /etc/nginx/conf.d 。默认主机文件 (in /etc/nginx/conf.d/default.conf) 配置：

```language-nginx
	vi /etc/nginx/conf.d/default.conf  

	[...]
	server {
	    listen       80;
	    server_name  localhost;

	    #charset koi8-r;
	    #access_log  /var/log/nginx/log/host.access.log  main;

	    location / {
	        root   /usr/share/nginx/html;
	        index  index.html index.htm;
	    }

	    #error_page  404              /404.html;

	    # redirect server error pages to the static page /50x.html
	    #
	    error_page   500 502 503 504  /50x.html;
	    location = /50x.html {
	        root   /usr/share/nginx/html;
	    }

	    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
	    #
	    #location ~ .php$ {
	    #    proxy_pass   http://127.0.0.1;
	    #}

	    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
	    #

	    location ~ .php$ {
	        root           /usr/share/nginx/html;
	        try_files $uri =404;
	        fastcgi_pass   127.0.0.1:9000;
	        fastcgi_index  index.php;
	        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
	        include        fastcgi_params;
	    }
		
		# deny access to .htaccess files, if Apache's document root
	    # concurs with nginx's one
	    #
	    location ~ /.ht {
	        deny  all;
	    }
	}
```  

server_name _; 使这是一个包罗万象的默认虚拟主机（当然，你可以同时喜欢在这里指定主机名 www.example.com).

在 location /部分，我们添加 index.php 到 index 行，根目录 /usr/share/nginx/html，网站文件默认目录 /usr/share/nginx/html.

在PHP中的重要组成部分是 location ~ .php$ {} 节。取消注释来启用它。改变 root 行到网站的文档根目录 (e.g. root /usr/share/nginx/html;)。请注意，我已经添加了一行 try_files $uri =404;以防止零日漏洞(参看http://wiki.nginx.org/Pitfalls#Passing_Uncontrolled_Requests_to_PHP 和http://forum.nginx.org/read.php?2,88845,page=3)。 请确保您更改fastcgi_param 行为 fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name； 否则PHP解释器不会发现你在浏览器中调用PHP脚本 ($document_root 翻译为 /usr/share/nginx/html 因为这是我们已经设置为我们的文档根目录）。

PHP-FPM 默认监听端口 9000 ，因此，我们告诉nginx的连接 127.0.0.1:9000 与行 fastcgi_pass 127.0.0.1:9000；另外，也可以使 PHP-FPM 使用Unix套接字。

现在保存文件并重新加载nginx：

```language-nginx
	systemctl restart nginx.service
```

现在创建的文档根目录下的PHP探针文件 /usr/share/nginx/html…

```language-php
	vi /usr/share/nginx/html/info.php

	<?php
	phpinfo();
	?>
```
浏览探针文件 (e.g. http://192.168.1.105/info.php):
 
### 7 让PHP-FPM使用Unix套接字

默认情况下监听端口 9000 。 另外，也可以使PHP-FPM使用Unix套接字，这避免了TCP的开销。要做到这一点，打开 /etc/php-fpm.d/www.conf…

```language-nginx
	vi /etc/php-fpm.d/www.conf
```

… 修改后如下：

```language-nginx
	[...]
	;listen = 127.0.0.1:9000
	listen = /var/run/php-fpm/php5-fpm.sock
	[...]
```

然后重新加载 PHP-FPM：

```language-nginx
	systemctl restart php-fpm.service
```

接下来通过你的nginx的配置和所有的虚拟主机和改线 fastcgi_pass 127.0.0.1:9000; to fastcgi_pass unix:/tmp/php5-fpm.sock;,像这样：

```language-nginx
	vi /etc/nginx/conf.d/default.conf

	[...]
	    location ~ .php$ {
	        root           /usr/share/nginx/html;
	        try_files $uri =404;
	        fastcgi_pass   unix:/var/run/php-fpm/php5-fpm.sock;
	        fastcgi_index  index.php;
	        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
	        include        fastcgi_params;
	    }
	[...]
```

最后重新加载 nginx：

```language-nginx
	systemctl restart nginx.service
```

### 8 相关连接：

[nginx](http://nginx.org/) http://nginx.org/  
[nginx Wiki](http://wiki.nginx.org/)  http://wiki.nginx.org/  
[PHP](http://www.php.net/)  http://www.php.net/  
[PHP-FPM](http://php-fpm.org/)  http://php-fpm.org/  
[MySQL](http://www.mysql.com/)  http://www.mysql.com/  
[CentOS](http://www.centos.org/)  http://www.centos.org/  

在最后一行后输入ctrl+z，文件会自动保存并退出  
再次在git上提交时就不用重复输入用户名密码了 