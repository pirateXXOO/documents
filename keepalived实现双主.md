# keepalived实现双主

```shell
vim /etc/keepalived/keepalived.conf
# 第一个主机

! Configuration File for keepalived

global_defs {
   notification_email { 
    root@localhost
   }   
    
   notification_email_from kaadmin@localhost 
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id node2
   vrrp_mcast_group4 224.0.71.1  # 组播地址，两个主机要一致
}

vrrp_instance VI_1 {   # 第一个主机组
    state BACKUP   # 从机
    interface eno16777736 # 网络接口
    virtual_router_id 71 # 虚拟路由id
    priority 98 # 优先级，两个要不一样，主机要高于从机
    advert_int 1 # 时间间隔
    authentication {
        auth_type PASS # 验证方式为密码
        auth_pass bashrc  # 密码
    }   
    virtual_ipaddress {
        172.16.71.80/16 dev eno16777736 label eno16777736:0 # 虚拟地址和标记
    }   
    track_interface {
        eno16777736 # 检查接口
    }   
	# 检查脚本
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
```

```shell
# 第二个主机

vim /etc/keepalived/keepalived.conf

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

```

```shell
# 自动报警脚本

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

