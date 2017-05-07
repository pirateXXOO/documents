# haproxy负载均衡两个后端httpd和mysql

前端主机：`172.16.71.1`，后端主机：`172.16.71.4`和`172.16.71.5`

前端主机安装haproxy

```shell
yum install haproxy

vim /etc/haproxy/haproxy.cfg

global

    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    #errorfile 503 /etc/haproxy/errorpages/503.html
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

defaults
    mode                    tcp			# 默认为http，若要负载均衡MySQL，要换成tcp
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8 header X-Client
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 30000
frontend main *:80   #可以直接写成*:80，也可以写成下面一行这样，bind *:80
    #bind *:80
    maxconn 6000
    acl invalid_src src 172.16.251.57 # 定义acl非法源地址
    acl adminapp path_beg -i /admin # 定义acl起始路径，-i表示不区分大小写
    acl static path_end -i .jpg .gif .png .jpeg .css .js .html  # 定义acl结尾路径，-i表示不区分大小写
    acl static path_beg /images /imgs /stylesheets /javascripts  # 定义acl起始路径，-i表示不区分大小写
    block if invalid_src adminapp # 如果符合acl invalid_src条件则禁止访问。
    use_backend staticsrvs if static # 如果符合acl static条件，则使用后端主机staticsrvs
    default_backend dynamicsrvs # 默认使用后端主机dynamicsrvs

# 上面的功能主要是动静分离

listen stats *:9022
    stats enable # 开启stats功能
    stats uri /admin?hastats # 定义stats uri
    stats realm Haproxy\ admin\ area # stats提示 
    stats auth admin:magedu # 定义stats认证账户
    stats hide-version # 隐藏版本信息
    stats admin if TRUE # 始终启用stats admin

backend dynamicsrvs
#   balance roundrobin
    balance roundrobin  # 采用轮询方式负载均衡
    hash-type consistent # hash方式采用一致性hash算法
#   option httpchk GET / HTTP/1.1\r\nHost:\ 
#   http-check expect status 200
    server web1 172.16.71.4:80 check weight 2 inter 3000 rise 1 fall 2 cookie web1 # 定义后端server 名称 地址+端口，检查，权重，检查间隔（ms），如果检查成功几次则启用，如果检查失败几次则停用，添加的cookie内容。
backend staticsrvs 
    cookie SRV insert indirect nocache 
    server web2 172.16.71.5:80 check cookie web2 maxconn 2000  # 定义server 名称 地址+端口，检查，cookie添加内容，最大并发连接数。

frontend mysql
    bind *:3306
    log global
#   acl invalid_src src 172.16.71.4 
#   tcp-request connection reject if invalid_src
    mode tcp
    default_backend mysrvs

backend mysrvs
    balance leastconn
    server mysql1 172.16.71.4:3306 check
    server mysql2 172.16.71.5:3306 check
# 后端MySQL服务器内要有相同的账号密码权限等内容。
```

# 添加支持https

```shell
vim /etc/haproxy/haproxy.cfg

frontend sslconn
    bind *:443 ssl crt /etc/haproxy/certs/haproxy.pem
    default_backend dynamicsrvs

```

```shell
# 生成证书
cd /etc/pki/CA/
(umask 077;openssl genrsa -out private/cakey.pem 2048 )
openssl req -new -x509 -key private/cakey.pem -out cacert.pem
touch index.txt
echo 00 > serial

cd /etc/haproxy/
mkdir certs
(umask 077;openssl genrsa -out haproxy.key 2048)
openssl req -new -key haproxy.key -out haproxy.csr 
openssl ca -in haproxy.csr -out haproxy.crt
cat haproxy.crt haproxy.key >haproxy.pem

mv haproxy.pem c
```



