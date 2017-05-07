# lvs

## Linux Cluster：

**Cluster**:计算机集合，为解决某个特定问题组合起来形成的单个系统；



#### Linux Cluster类型：

- **LB**:Load Balancing,负载均衡；

- **HA**:High Availability,高可用；高可用的衡量公式：

  A=MTBF/(MTBF+MTTR)          可用性=平均无故障时间/(平均无故障时间+平均修复时间)

  是一个0到1的开区间(0,1)：90%, 95%, 99%, 99.5%,  99.9%, 99.99%, 99.999%, 99.9999%

- **HP**:高兴能；



#### 分布式系统：

- 分布式存储
- 分布式计算



#### 系统扩展方式：

- Scale Up:向上扩展
- Scale Out:向外扩展



### LB Cluster的实现：

- 硬件：F5 Big-IP、Citrix Netscaler、A10 A10

- 软件：lvs：Linux Virtual Server、

  nginx、

  haproxy、

  ats：apache traffic server 、

  perlbal、

  pound

- 基于工作的协议层次划分：

  传输层（通用）：（DPORT）

  ​	lvs：

  ​	Nginx：（stream）

  ​	haproxy：（modd tcp）

  应用层（专用）：（自定义的请求模型分类）

  ​	proxy sferver:

  ​		http:nginx,httpd,haproxy(mode httpd),...

  ​		fastcgi:nginx,httpd,...

  ​		mysql:mysql-proxy,...

  ​		.......

  站点指标：

  ​	PV:Page View

  ​	UV:Unique Vistor

  ​	IP:

- 会话保持：

  (1) session sticky

  ​	Source IP

  ​	Cookie

  (2) session replication;

  ​	session multicate cluster

  (3) session server

  ​

# lvs

​	VS:Virtual Server

​	RS:Real Server

**l4**:四层路由，四层交换机

​	**VS**：根据请求报文的目标IP和目标协议及端口将其调度转发至某Real Server，根据调度算法来挑选RS；



**iptables/netfilter**:

- iptables:用户控件的管理工具；

- netfilter:内核控件上的框架；

  流入：PREROUTING --> INPUT

  流出：OUTPUT --> POSTROUTING

  转发：PREROUTING --> FORWARD --> POSTROUTING

- DNAT:目标地址转换：PREROUTING；



**lvs:ipvsadm/ipvs**

- ipvsadm:用户控件的命令行工具，规则管理器，用于管理技巧服务及Real Server；
- ipvs:工作于内核空间的netfilter的INPUT钩子之上的框架；



**lvs集群类型中的术语：**

- vs：Virtual Server, Director, Dispatcher, Balancer
- rs：Real Server, upstream server, backend server
- CIP：Client IP；VIP: Virtual serve IP；RIP: Real server IP, 
- DIP： Director IP
- CIP <--> VIP == DIP <--> RIP 



**lvs集群的类型：**

- lvs-nat：修改请求报文的目标IP；多目标IP的DNAT；
- lvs-dr：操纵封装新的MAC地址；
- lvs-tun：在原请求IP报文之外新加一个IP首部；
- lvs-fullnat：修改请求报文的源和目标IP；



**lvs-nat：**

​	多目标IP的DNAT，通过将请求报文中的目标地址和目标端口修改为某挑出的RS的RIP和PORT实现转发；

（1）RIP和DIP必须在同一个IP网络，且应该使用私网地址；RS的网关要指向DIP；

（2）请求报文和响应报文都必须经由Director转发；Director易于成为系统瓶颈；

（3）支持端口映射，可修改请求报文的目标PORT；

（4）vs必须是Linux系统，rs可以是任意系统；

![](C:\Users\Jack\OneDrive\文档\lvs\lvs-nat.png)

**lvs-dr:**

Direct Routing，直接路由；

通过为请求报文重新封装一个MAC首部进行转发，源MAC是DIP所在的接口的MAC，目标MAC是某挑选出的RS的RIP所在接口的MAC地址；源IP/PORT，以及目标IP/PORT均保持不变；

Director和各RS都得配置使用VIP；

(1) 确保前端路由器将目标IP为VIP的请求报文发往Director：

​	(a) 在前端网关做静态绑定；

​	(b) 在RS上使用arptables；

​	(c) 在RS上修改内核参数以限制arp通告及应答级别；

​		arp_announce

​		arp_ignore

(2) RS的RIP可以使用私网地址，也可以是公网地址；RIP与DIP在同一IP网络；RIP的网关不能指向DIP，以确保响应报文不会经由Director；

(3) RS跟Director要在同一个物理网络；

(4) 请求报文要经由Director，但响应不能经由Director，而是由RS直接发往Client；

(5) 不支持端口映射；

![](C:\Users\Jack\OneDrive\文档\lvs\lvs-dr.png)



**lvs-tun:**

转发方式：不修改请求报文的IP首部（源IP为CIP，目标IP为VIP），而在原IP报文之外再封装一个IP首部（源IP是DIP，目标IP是RIP），将报文发往挑选出的目标RS；RS直接响应给客户端（源IP是VIP，目标IP是CIP）；

(1) DIP, VIP, RIP都应该是公网地址；

(2) RS的网关不能，也不可能指向DIP；

(3) 请求报文要经由Director，但响应不能经由Director；

(4) 不支持端口映射；

(5) RS的OS得支持隧道功能；

![](C:\Users\Jack\OneDrive\文档\lvs\lvs-tun.png)

**lvs-fullnat:**

通过同时修改请求报文的源IP地址和目标IP地址进行转发；

​	CIP --> DIP 

​	VIP --> RIP 

(1) VIP是公网地址，RIP和DIP是私网地址，且通常不在同一IP网络；因此，RIP的网关一般不会指向DIP；

(2) RS收到的请求报文源地址是DIP，因此，只需响应给DIP；但Director还要将其发往Client；

(3) 请求和响应报文都经由Director；

(4) 支持端口映射；

![](C:\Users\Jack\OneDrive\文档\lvs\lvs-fullnat.png)



# 总结：

lvs-nat,lvs-fullnat:请求和响应报文都经由Director；

​	lvs-nat：RIP的网关要指向DIP；

​	lvs-fullnat：RIP和DIP未必在同一IP网络，但要能通信

lvs-dr，lvs-tun：请求报文要经由Director，但响应报文由RS直接发往Client；

​	lvs-dr：通过封装新的MAC首部实现，通过MAC网络转发；

​	lvs-tun：通过在原IP报文之外封装新的IP报文实现转发，支持远距离通信；

​	