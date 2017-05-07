# keepalived+lvs负载均衡两个httpd

准备4台主机`172.16.71.2` `172.16.71.3` `172.16.71.4` `172.16.71.5`

前两个做后端httpd服务器。后两个做keepalived

-----

首先配置好后端主机`172.16.71.2` 和`172.16.71.3`

```shell
# 172.16.71.2
yum install httpd
cd /var/www/html
vim index.html
server1 71.2
```

```shell
# 172.16.71.3
yum install httpd
cd /var/www/html
vim index.html
server2 71.3
```

由于lvs采用的是DR模式，所以要写脚本修改内核参数并添加路由信息，以下是keepalived双主需要的脚本，两台主机上都需要运行一次：

```shell
#!/bin/bash
#
vip=172.16.71.80
vip2=172.16.71.90
mask='255.255.255.255'
interface='lo:0'
interface2='lo:1'

case $1 in
start)
        echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
        echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
        echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
        echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
        ifconfig $interface  $vip netmask $mask broadcast $vip up
        ifconfig $interface2  $vip2 netmask $mask broadcast $vip2 up
        route add -host $vip dev $interface
        route add -host $vip2 dev $interface2
        ;;
stop)
        ifconfig $interface down
        ifconfig $interface2 down
        echo 0 > /proc/sys/net/ipv4/conf/all/arp_ignore
        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_ignore
        echo 0 > /proc/sys/net/ipv4/conf/all/arp_announce
        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_announce
        ;;
*)
        echo "Usage $(basename $0) start|stop"
        exit 1
        ;;
esac
```

执行完成后启动httpd`systemctl start httpd`

-----

然后配置前端的keepalived，这里采用的是双主模式。

配置文件如下：

```shell
# 172.16.71.4

! Configuration File for keepalived

global_defs {
   notification_email {
    root@localhost
   }   
    
   notification_email_from kaadmin@localhost 
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id node1
   vrrp_mcast_group4 224.0.71.1
}

vrrp_instance VI_1 {
    state MASTER
    interface eno16777736
    virtual_router_id 71
    priority 100 
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass bashrc
    }   
    virtual_ipaddress {
        172.16.71.80/16 dev eno16777736 label eno16777736:0
    }   
    track_interface {
        eno16777736
    }   
    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}
vrrp_instance VI_2 {
    state BACKUP
    interface eno16777736
    virtual_router_id 45
    priority 98
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass shell
    }
    virtual_ipaddress {
        172.16.71.90/16 dev eno16777736 label eno16777736:1
    }
    track_interface {
        eno16777736
    }
    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}

virtual_server fwmark 3 {
    delay_loop 2
    lb_algo rr
    lb_kind DR
    nat_mask 255.255.0.0
    protocol TCP

    sorry_server 127.0.0.1 80

    real_server 172.16.71.2 80 {
        weight 1
        HTTP_GET {
            url {
              path /
              status_code 200
            }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 2
        }
    }

    real_server 172.16.71.3 80 {
        weight 1
        HTTP_GET {
            url {
              path /
              status_code 200
            }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 2
        }
    }
}
```

```shell
# 172.16.71.5

! Configuration File for keepalived

global_defs {
   notification_email {
    root@localhost
   }   
    
   notification_email_from kaadmin@localhost 
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id node2
   vrrp_mcast_group4 224.0.71.1
}

vrrp_instance VI_1 {
    state BACKUP
    interface eno16777736
    virtual_router_id 71
    priority 98
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass bashrc
    }   
    virtual_ipaddress {
        172.16.71.80/16 dev eno16777736 label eno16777736:0
    }   
    track_interface {
        eno16777736
    }   
        notify_master "/etc/keepalived/notify.sh master"
        notify_backup "/etc/keepalived/notify.sh backup"
        notify_fault "/etc/keepalived/notify.sh fault"


}
vrrp_instance VI_2 {
    state MASTER
    interface eno16777736
    virtual_router_id 45
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass shell
    }
    virtual_ipaddress {
        172.16.71.90/16 dev eno16777736 label eno16777736:1
    }
    track_interface {
        eno16777736
    }
        notify_master "/etc/keepalived/notify.sh master"
        notify_backup "/etc/keepalived/notify.sh backup"
        notify_fault "/etc/keepalived/notify.sh fault"
}


virtual_server fwmark 3 {
    delay_loop 2
    lb_algo rr
    lb_kind DR
    nat_mask 255.255.0.0
    protocol TCP

    sorry_server 127.0.0.1 80

    real_server 172.16.71.2 80 {
        weight 1
        HTTP_GET {
            url {
              path /
              status_code 200
                }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 2
        }
    }

    real_server 172.16.71.3 80  {
        weight 1
        HTTP_GET {
            url {
              path /
              status_code 200
            }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 2
        }
    }
}

```

```shell
# 告警脚本

#!/bin/bash
#
contact='root@localhost'
notify() {
	mailsubject="$(hostname) to be $1, vip floating"
	mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"
	echo "$mailbody" | mail -s "$mailsubject" $contact
}
case $1 in
master)
	notify master
	;;
backup)
	notify backup
	;;
fault)
	notify fault
	;;
*)
	echo "Usage: $(basename $0) {master|backup|fault}"
	exit 1
	;;
esac			
```

在两个主机上添加iptables

```shell
iptables -t mangle -A PREROUTING -d 172.16.71.80 -p tcp --dport 80 -j MARK --set-mark 3
iptables -t mangle -A PREROUTING -d 172.16.71.90 -p tcp --dport 80 -j MARK --set-mark 3
```

在两台主机上安装httpd并添加sorry page，启动httpd和keepalived

```shell
# 172.16.71.4
yum install httpd
cd /var/www/html
vim index.html
sorry page 1
systemctl start httpd keepalived
```

```shell
# 172.16.71.5
yum install httpd
cd /var/www/html
vim index.html
sorry page 2
systemctl start httpd keepalived
```

----

### 测试

访问`172.16.71.80`或者`172.16.71.90`时，会显示`server1 71.2` 或`server2 71.3`页面。

断掉后端两个httpd服务器，再次访问会显示`sorry page 1`或者`sorry page 2`。

断掉不同的keepalived会显示不同的sorry  page

----

下面是keepalived单主模式的配置：

```shell
# /etc/keepalived/keepalived.conf

# 172.16.71.4

! Configuration File for keepalived

global_defs {
   notification_email {
    root@localhost
   }   
    
   notification_email_from kaadmin@localhost 
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id node1
   vrrp_mcast_group4 224.0.71.1
}

vrrp_instance VI_1 {
    state MASTER
    interface eno16777736
    virtual_router_id 71
    priority 100 
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass bashrc
    }   
    virtual_ipaddress {
        172.16.71.80/16 dev eno16777736 label eno16777736:0
    }   
    track_interface {
        eno16777736
    }   
    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}
vrrp_instance VI_2 { # 这段可以注释掉，不用。
    state BACKUP
    interface eno16777736
    virtual_router_id 45
    priority 98
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass shell
    }
    virtual_ipaddress {
        172.16.71.90/16 dev eno16777736 label eno16777736:1
    }
    track_interface {
        eno16777736
    }
    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}

virtual_server 172.16.71.80 80 { # 此处需要修改为虚拟地址
    delay_loop 2
    lb_algo rr
    lb_kind DR
    nat_mask 255.255.0.0
    protocol TCP

    sorry_server 127.0.0.1 80

    real_server 172.16.71.2 80 {
        weight 1
        HTTP_GET {
            url {
              path /
              status_code 200
            }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 2
        }
    }

    real_server 172.16.71.3 80 {
        weight 1
        HTTP_GET {
            url {
              path /
              status_code 200
            }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 2
        }
    }
}

```

```shell
# /etc/keepalived/keepalived.conf

# 172.16.71.5

! Configuration File for keepalived

global_defs {
   notification_email {
    root@localhost
   }   
    
   notification_email_from kaadmin@localhost 
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id node2
   vrrp_mcast_group4 224.0.71.1
}

vrrp_instance VI_1 {
    state BACKUP
    interface eno16777736
    virtual_router_id 71
    priority 98
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass bashrc
    }   
    virtual_ipaddress {
        172.16.71.80/16 dev eno16777736 label eno16777736:0
    }   
    track_interface {
        eno16777736
    }   
        notify_master "/etc/keepalived/notify.sh master"
        notify_backup "/etc/keepalived/notify.sh backup"
        notify_fault "/etc/keepalived/notify.sh fault"


}
vrrp_instance VI_2 { # 这段可以注释掉，不用。
    state MASTER
    interface eno16777736
    virtual_router_id 45
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass shell
    }
    virtual_ipaddress {
        172.16.71.90/16 dev eno16777736 label eno16777736:1
    }
    track_interface {
        eno16777736
    }
        notify_master "/etc/keepalived/notify.sh master"
        notify_backup "/etc/keepalived/notify.sh backup"
        notify_fault "/etc/keepalived/notify.sh fault"
}


virtual_server 172.16.71.80 80 { # 此处需要修改为虚拟地址
    delay_loop 2
    lb_algo rr
    lb_kind DR
    nat_mask 255.255.0.0
    protocol TCP

    sorry_server 127.0.0.1 80

    real_server 172.16.71.2 80 {
        weight 1
        HTTP_GET {
            url {
              path /
              status_code 200
                }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 2
        }
    }

    real_server 172.16.71.3 80  {
        weight 1
        HTTP_GET {
            url {
              path /
              status_code 200
            }
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 2
        }
    }
}

```

单主模式不需要手动添加iptables。