# ipvs

## ipvs scheduler:

根据其调度时是否考虑各RS当前的负载状态，可分为静态方法和动态方法两种：

**静态方法：**仅根据算法本身进行调度：

​	RR：roundrobin，轮询；

​	WRR：Weighted RR，加权轮询；

​	SH：Source Hashing，实现session sticy，源IP地址hash；将来自于同一个IP地址的请求始终发往第一次挑中的RS，从而实现会话绑定；

​	DH：Destination Hashing；目标地址哈希，将发往同一个目标地址的请求始终转发至第一次挑中的RS；

**动态方法：**主要根据每RS当前的负载状态及调度算法进行调度；

​	LC：least connections

​		Overhead=activeconns*256+inactiveconns

​	WLC：Weighted LC

​		Overhead=(activeconns*256+inactiveconns)/weight

​	SED：Shortest Expection Delay

​		Overhead=(activeconns+1)*256/weight

​	NQ：Never Queue

​	LBLC：Locality-Based LC，动态的DH算法；

​	LBLCR：LBLC with Replication，带复制功能的LBLC；



ipvsadm/ipvs:

​	ipvsadm :

​		程序包：ipvsadm

​			Unit file：ipvsadm.service

​			主程序：/usr/sbin/ipvsadm

​			规则保存工具：/usr/sbin/ipvsadm-save

​			故障重载工具：/usr/sbin/ipvsadm-restore

​			配置文件：/etc/sysconfig/ipvsadm-config

​	ipvsadm命令：

​		核心功能：

​			集群服务管理：增、删、改；

​			集群服务的RS管理：增、删、改；

​			查看：

```shell
ipvsadm -A|E -t|u|f service-address [-s scheduler] [-p [timeout]] [-M netmask] [--pe persistence_engine] [-b sched-flags]
ipvsadm -D -t|u|f service-address
ipvsadm -C
ipvsadm -R
ipvsadm -S [-n]
ipvsadm -a|e -t|u|f service-address -r server-address [options]
ipvsadm -d -t|u|f service-address -r server-address
ipvsadm -L|l [options]
ipvsadm -Z [-t|u|f service-address]
```

管理集群服务：增、改、删；

​	增、改：`ipvsadm -A|E -t|u|f service-address [-s scheduler] [-p [timeout]]`

​	删：`ipvsadm -D -t|u|f service-address`

​	service-address:

​		-t|u|f:

​			-t: TCP协议的端口，VIP:TCP_PORT

​			-u: UDP协议的端口，VIP:UDP_PORT

​			-f：firewall MARK，是一个数字；

​	[-s scheduler]：指定集群的调度算法，默认为wlc；



管理集群上的RS：增、改、删；

​	增、改：`ipvsadm -a|e -t|u|f service-address -r server-address [-g|i|m] [-w weight]`

​	删：`ipvsadm -d -t|u|f service-address -r server-address`

​	server-address：rip[:port]

​	选项：

​		lvs类型：

​			-g: gateway, dr类型

​			-i: ipip, tun类型

​			-m: masquerade, nat类型

​		-w weight:权重；

清空定义的所有内容：`ipvsadm -C`

查看：`ipvsadm -L|l [options]`

```shell
--numeric, -n：numeric output of addresses and ports 
--exact：expand numbers (display exact values)
--connection， -c：output of current IPVS connections
--stats：output of statistics information
--rate ：output of rate information
```

保存和重载：

​	`ipvsadm -S = ipvsadm-save`

​	`ipvsadm -R = ipvsadm-restore `

-----

负载均衡集群的设计要点：

​	(1) 是否需要会话保持；

​	(2) 是否需要共享存储；

​		共享存储：NAS， SAN， DS（分布式存储）

​		数据同步：



## lvs-nat实现：

三台主机

- 第一台：两个网卡，eno16777736,ip`172.16.71.1`,网络为vmnet0桥接模式;

  eno33554984,ip`192.168.71.1`,网络为vmnet1仅主机模式。

- 第二台：一个网卡，eth0,ip`192.168.71.2` ，网络为vmnet1仅主机模式。

- 第三台：一个网卡，eth0,ip`192.168.71.3` ，网络为vmnet1仅主机模式。



