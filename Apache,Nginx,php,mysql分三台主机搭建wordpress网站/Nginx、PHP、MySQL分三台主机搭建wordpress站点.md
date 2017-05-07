# Apache/Nginx、PHP、Mariadb三台主机搭建wordpress站点

首先，需要三台CentOS主机，地址分别为``172.16.71.1``,``172.16.71.3``,``172.16.71.4``

![](C:\Users\Jack\OneDrive\文档\Apache,Nginx,php,mysql分三台主机搭建wordpress网站\1.PNG)

第一台主机``172.16.71.1``作为Apache/Nginx服务器，第二台主机`172.16.71.3`作为php服务器，第三台主机`172.16.71.4`作为Mariadb服务器。



##  配置Apache服务器（Nginx的配置后面讲）

- 安装httpd：

```shell
yum install httpd -y
...... 省略
[root@Shining ~]# rpm -qi httpd
Name        : httpd
Version     : 2.4.6
Release     : 40.el7.centos
Architecture: x86_64
Install Date: Wed 02 Nov 2016 04:03:40 PM CST
Group       : System Environment/Daemons
Size        : 9806197
License     : ASL 2.0
Signature   : RSA/SHA256, Wed 25 Nov 2015 10:41:23 PM CST, Key ID 24c6a8a7f4a80eb5
Source RPM  : httpd-2.4.6-40.el7.centos.src.rpm
Build Date  : Fri 20 Nov 2015 05:45:11 AM CST
Build Host  : worker1.bsys.centos.org
Relocations : (not relocatable)
Packager    : CentOS BuildSystem <http://bugs.centos.org>
Vendor      : CentOS
URL         : http://httpd.apache.org/
Summary     : Apache HTTP Server
Description :
The Apache HTTP Server is a powerful, efficient, and extensible
web server.
```

- 配置httpd：

  ```shell
  vim /etc/httpd/conf/httpd.conf

  DocumentRoot "/var/www/html/wordpress" # DocumentRoot改为wordpress根目录

  # 关闭正向代理
  ProxyRequests Off 

  # 将以‘.php’结尾的文件交给后面的php服务器进行处理
  ProxyPassMatch ^/(.*\.php)$ fcgi://172.16.71.3:9000/var/www/html/wordpress/$1 

  # 在DirectoryIndex中添加‘index.php’
  <IfModule dir_module>
      DirectoryIndex index.html index.php
  </IfModule>

  # 添加php文件支持：
  AddType application/x-httpd-php .php
  AddType application/x-httpd-php-source .phps

  [root@Shining ~]# cd /etc/httpd/conf.modules.d/
  [root@Shining /etc/httpd/conf.modules.d]# ls
  00-base.conf  00-lua.conf  00-proxy.conf  00-systemd.conf  10-php.conf
  00-dav.conf   00-mpm.conf  00-ssl.conf    01-cgi.conf      10-wsgi.conf
  [root@Shining /etc/httpd/conf.modules.d]# vim 00-proxy.conf

  # This file configures all the proxy modules:
  LoadModule proxy_module modules/mod_proxy.so
  LoadModule lbmethod_bybusyness_module modules/mod_lbmethod_bybusyness.so
  LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
  LoadModule lbmethod_bytraffic_module modules/mod_lbmethod_bytraffic.so
  LoadModule lbmethod_heartbeat_module modules/mod_lbmethod_heartbeat.so
  LoadModule proxy_ajp_module modules/mod_proxy_ajp.so
  LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
  LoadModule proxy_connect_module modules/mod_proxy_connect.so
  LoadModule proxy_express_module modules/mod_proxy_express.so
  LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so  # 确保此模块处于加载状态
  LoadModule proxy_fdpass_module modules/mod_proxy_fdpass.so
  LoadModule proxy_ftp_module modules/mod_proxy_ftp.so
  LoadModule proxy_http_module modules/mod_proxy_http.so
  LoadModule proxy_scgi_module modules/mod_proxy_scgi.so
  LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so
  ```



## 配置PHP服务器

- 安装php、php-fpm

  ```shell
  yum install php php-fpm -y
  ```

- 配置PHP服务器

  ```shell
  vim /etc/php-fpm.d/www.conf

  # 监听本机的9000端口
  listen = 172.16.71.3:9000

  # 允许Apache服务器向自己发送请求
  listen.allowed_clients = 172.16.71.1
  ```

> 注意：因为Apache服务器和PHP服务器是两台主机，动态资源的请求都会被送去PHP服务器，静态资源的请求还是在Apache服务器被处理。由于wordpress的动态资源和静态资源没有被分开，所以，在Apache服务器上也需要有一份wordpress。

## 在PHP服务器上部署wordpress

下载并解压wordpress到`/var/www/html/wordpress`中，这个路径要与前面httpd配置文件中的路径相同。

```shell
[root@Shining ~]# cd /var/www/html/wordpress
[root@Shining /var/www/html/wordpress]# ls
centos           wp-admin              wp-content         wp-login.php      xmlrpc.php
index.php        wp-blog-header.php    wp-cron.php        wp-mail.php
license.txt      wp-comments-post.php  wp-includes        wp-settings.php
readme.html      wp-config.php         wp-links-opml.php  wp-signup.php
wp-activate.php  wp-config-sample.php  wp-load.php        wp-trackback.php
[root@Shining ~]# cp wp-config-sample.php wp-config.php
[root@Shining ~]# vim wp-config.php

## 按要求修改以下选项： 
// ** MySQL 设置 - 具体信息来自您正在使用的主机 ** //
/** WordPress数据库的名称 */
define('DB_NAME', 'phpmyadmin');

/** MySQL数据库用户名 */
define('DB_USER', 'phpmyadmin');

/** MySQL数据库密码 */
define('DB_PASSWORD', 'magedu');

/** MySQL主机 */
define('DB_HOST', '172.16.71.4');

/** 创建数据表时默认的文字编码 */
define('DB_CHARSET', 'utf8');

/** 数据库整理类型。如不确定请勿更改 */
define('DB_COLLATE', '');

```



## 配置Mariadb服务器



```shell
[root@localhost ~]# yum install mariadb-server

[root@localhost ~]# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 2
Server version: 5.5.46-MariaDB-log MariaDB Server

Copyright (c) 2000, 2015, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database phpmyadmin;
Query OK, 1 row affected (0.28 sec)

MariaDB [(none)]> grant all on phpmyadmin.* to 'phpmyadmin'@'%' identified by 'magedu';
Query OK, 0 rows affected (1.23 sec)

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| phpmyadmin         |
| test               |
+--------------------+
5 rows in set (0.56 sec)

MariaDB [(none)]> show grants for phpmyadmin;
+-----------------------------------------------------------------------------------------------------------+
| Grants for phpmyadmin@%                                                                                   |
+-----------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'phpmyadmin'@'%' IDENTIFIED BY PASSWORD '*6B8CCC83799A26CD19D7AD9AEEADBCD30D8A8664' |
| GRANT ALL PRIVILEGES ON `phpmyadmin`.* TO 'phpmyadmin'@'%'                                                |
+-----------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)

```



## 启动三个服务

`172.16.71.4`

```shell
[root@localhost ~]# systemctl start mariadb
[root@localhost ~]# ss -tnl
State       Recv-Q Send-Q Local Address:Port                Peer Address:Port              
LISTEN      0      50                 *:3306                           *:*                  
LISTEN      0      5      192.168.122.1:53                             *:*                  
LISTEN      0      128                *:22                             *:*                  
LISTEN      0      128        127.0.0.1:631                            *:*                  
LISTEN      0      100        127.0.0.1:25                             *:*                  
LISTEN      0      128               :::22                            :::*                  
LISTEN      0      128              ::1:631                           :::*                  
LISTEN      0      100              ::1:25                            :::*   
```

`172.16.71.3`

```shell
[root@Shining /var/www/html/wordpress]# systemctl start php-fpm
[root@Shining /var/www/html/wordpress]# ss -tnl
State      Recv-Q Send-Q        Local Address:Port                       Peer Address:Port              
LISTEN     0      128             172.16.71.3:9000                                  *:*                  
LISTEN     0      5             192.168.122.1:53                                    *:*                  
LISTEN     0      128                       *:22                                    *:*                  
LISTEN     0      128               127.0.0.1:631                                   *:*                  
LISTEN     0      100               127.0.0.1:25                                    *:*                  
LISTEN     0      128                      :::22                                   :::*                  
LISTEN     0      128                     ::1:631                                  :::*                  
LISTEN     0      100                     ::1:25                                   :::*                  

```

`172.16.71.1`

```shell
[root@Shining /etc/httpd/conf.modules.d]# systemctl start httpd
[root@Shining /etc/httpd/conf.modules.d]# ss -tnl
State       Recv-Q Send-Q           Local Address:Port                          Peer Address:Port              
LISTEN      0      50                           *:3306                                     *:*                  
LISTEN      0      5                192.168.122.1:53                                       *:*                  
LISTEN      0      128                          *:22                                       *:*                  
LISTEN      0      128                  127.0.0.1:631                                      *:*                  
LISTEN      0      100                  127.0.0.1:25                                       *:*                  
LISTEN      0      128                         :::80                                      :::*                  
LISTEN      0      32                          :::21                                      :::*                  
LISTEN      0      128                         :::22                                      :::*                  
LISTEN      0      128                        ::1:631                                     :::*                  
LISTEN      0      100                        ::1:25                                      :::*                  
LISTEN      0      128                         :::443                                     :::*      
```



## 测试：

在实体机浏览器中输入`172.16.71.1`

![](C:\Users\Jack\OneDrive\文档\Apache,Nginx,php,mysql分三台主机搭建wordpress网站\3.PNG)

```shell
[root@Shining /var/www/html/wordpress]# tail /var/log/php-fpm/error.log
[13-Jan-2017 22:43:56] NOTICE: [pool www] child 11830 started
[13-Jan-2017 22:46:23] WARNING: [pool www] child 11830, script '/var/www/html/wordpress/wp-admin/admin-ajax.php' (request: "GET /wp-admin/admin-ajax.php") execution timed out (12.348141 sec), terminating
[13-Jan-2017 22:46:23] WARNING: [pool www] child 11830 exited on signal 15 (SIGTERM) after 146.629210 seconds from start
[13-Jan-2017 22:46:23] NOTICE: [pool www] child 11860 started
[14-Jan-2017 17:59:51] NOTICE: fpm is running, pid 8206
[14-Jan-2017 17:59:51] NOTICE: ready to handle connections
[14-Jan-2017 17:59:51] NOTICE: systemd monitor interval set to 10000ms
[14-Jan-2017 18:03:58] WARNING: [pool www] child 8252, script '/var/www/html/wordpress/index.php' (request: "GET /index.php") execution timed out (13.331337 sec), terminating
[14-Jan-2017 18:03:59] WARNING: [pool www] child 8252 exited on signal 15 (SIGTERM) after 247.658835 seconds from start
[14-Jan-2017 18:03:59] NOTICE: [pool www] child 8541 started

```

原因是请求超时，解决方法为：

```shell
[root@Shining /var/www/html/wordpress]# vim /etc/php-fpm.d/www.conf

# 将这个时间改的大一点
request_terminate_timeout = 100s

# 重启php-fpm
[root@Shining /var/www/html/wordpress]# systemctl restart php-fpm
```

重新测试：

![](C:\Users\Jack\OneDrive\文档\Apache,Nginx,php,mysql分三台主机搭建wordpress网站\4.PNG)

成功！

![](C:\Users\Jack\OneDrive\文档\Apache,Nginx,php,mysql分三台主机搭建wordpress网站\5.PNG)

![](C:\Users\Jack\OneDrive\文档\Apache,Nginx,php,mysql分三台主机搭建wordpress网站\6.PNG)

![](C:\Users\Jack\OneDrive\文档\Apache,Nginx,php,mysql分三台主机搭建wordpress网站\7.PNG)

![](C:\Users\Jack\OneDrive\文档\Apache,Nginx,php,mysql分三台主机搭建wordpress网站\8.PNG)



## Nginx配置

首先停止httpd服务

```shell
[root@Shining /etc/httpd/conf.modules.d]# systemctl stop httpd
[root@Shining /etc/httpd/conf.modules.d]# ss -tnl
State      Recv-Q Send-Q                       Local Address:Port                                      Peer Address:Port              
LISTEN     0      50                                       *:3306                                                 *:*                  
LISTEN     0      5                            192.168.122.1:53                                                   *:*                  
LISTEN     0      128                                      *:22                                                   *:*                  
LISTEN     0      128                              127.0.0.1:631                                                  *:*                  
LISTEN     0      100                              127.0.0.1:25                                                   *:*                  
LISTEN     0      32                                      :::21                                                  :::*                  
LISTEN     0      128                                     :::22                                                  :::*                  
LISTEN     0      128                                    ::1:631                                                 :::*                  
LISTEN     0      100                                    ::1:25                                                  :::* 
```

配置Nginx：

```shell
[root@Shining /etc/httpd/conf.modules.d]# vim /etc/nginx/conf.d/default.conf 

    location / {
        root   /var/www/html/wordpress; # 修改根目录
        index  index.html index.htm index.php forum.php; # 添加 index.php
    }

# 添加或修改下面的内容：
    location ~ .*\.php$ {
        root           /var/www/html/wordpress;
        fastcgi_pass   172.16.71.3:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }

[root@Shining /etc/httpd/conf.modules.d]# nginx -t # 测试配置文件的正确性
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@Shining /etc/httpd/conf.modules.d]# nginx  # 启动Nginx
[root@Shining /etc/httpd/conf.modules.d]# ss -tnl
State      Recv-Q Send-Q                       Local Address:Port                                      Peer Address:Port              
LISTEN     0      50                                       *:3306                                                 *:*                  
LISTEN     0      128                                      *:80                                                   *:*                  
LISTEN     0      5                            192.168.122.1:53                                                   *:*                  
LISTEN     0      128                                      *:22                                                   *:*                  
LISTEN     0      128                              127.0.0.1:631                                                  *:*                  
LISTEN     0      100                              127.0.0.1:25                                                   *:*                  
LISTEN     0      32                                      :::21                                                  :::*                  
LISTEN     0      128                                     :::22                                                  :::*                  
LISTEN     0      128                                    ::1:631                                                 :::*                  
LISTEN     0      100                                    ::1:25                                                  :::*        
```

![](C:\Users\Jack\OneDrive\文档\Apache,Nginx,php,mysql分三台主机搭建wordpress网站\9.PNG)

**成功！**



# 总结

在配置Nginx完之后，访问出现了‘404’，原因是location下的root路径写错了。

还出现一次下载.php文件而不是以网页形式出现的情况，原因是`    location ~ .*\.php$ `写错了。

**写配置文件的时候还是需要细心，细心再细心！**



