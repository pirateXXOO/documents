# iptables 从入门到放弃

​	Linux的防火墙体系主要工作在网络层，针对TCP/IP数据包实施过滤和限制，属于典型的包过滤防火墙(或网络层防火墙)。在Linux中netfilter和iptables都是指Linux防火墙。区别在于：

​	netfilter：指的是Linux内核中实现包过滤防火墙的内部结构，不以程序或文件的形式存在，属于“内核态”的防火墙功能体系。

​	iptables：指的是用来管理Linux防火墙的命令程序，通常位于``/sbin/iptables``，属于“用户态”的防火墙管理体系。

​	在下述的内容中我们就以iptables来称呼Linux防火墙了。



## 了解iptables的表和链的结构：

​	iptables的作用在于为包过滤机制的实现提供规则，通过各种不同的规则，告诉netfilter对来自某些源，前往某些目的的或具有某些协议特征的数据包应该如何处理。为了更加方便地组织和管理防火墙规则，iptables采用了“ 表”和“链”的分层结构。iptables的规则表和规则链的示意图如下：

![](http://www.it165.net/uploadfile/2013/0807/20130807085143891.jpg)

##### 1、规则表

​	规则表中包含各种规则链，iptables管理着四个不同的规则表，其功能分别由独立的内核模块实现。各表解释如下：

​	**filter表**：主要用来对数据包进行过滤，根据具体的规则要求决定如何处理一个数据包。包含INPUT、FORWAED、OUTPUT等三个规则链。

​	**nat表**：主要用修改数据包IP地址、端口号等信息，也称网络地址转换。包含PREROUTING、POSTROUTING、OUTPUT等三个规则链。

​	**mangle表**：主要用来修改数据包的TOS，TTL值，或者为数据包设置Mark标记，以实现流量整形，策略路由等高级应用。包含PREROUTING、POSTROUTING、INPUT、OUTPUT、FORWARD等五个规则链。

​	**raw表**：主要用来决定是否对数据包进行状态跟踪。包含OUTPUT、PREROUTING两个规则链。

​	在iptables规则表中，filter表和nat表用的比较多，而其他两个表用的相对来说比较少。所以后面我们大多是以filter表和nat表中的规则链做策略。

##### 2、规则链

​	规则链的作用是容纳各种防火墙规则，规则两分为五种，分别在不同的时机处理数据包。

- INPUT链：处理入站数据包。



- OUTOPUT链：处理出站数据包。



- FORWARD链：处理转发数据包。



- POSTROUTING链：在进行路由选择后处理数据包。



- PREROUTING链：在进行路由选择前处理数据包。


​	其中INPUT和OUTPUT链主要用在“主机型防火墙”中，而FORWARD、POSTROUTING和PREROUTING链主要用在“网关防火墙”中。



## 熟知数据包过滤的匹配流程

> ​	注意：熟知数据包过滤的匹配流程是学习iptables的重点，只有知道数据包经过防火墙的过程，我们才知道在那个表的那个链里面设置规则可以对数据包进行如何操作。

![](http://www.it165.net/uploadfile/2013/0807/20130807085145386.jpg)

###### 1、规则之间的顺序

​	当数据包抵达防火墙时，将依次应用raw，mangle，net和filter表中对应链内的规则(如果有的话)。

##### 2、规则链之间的顺序。

​	**入站数据流向**：来自外界的数据包到达防火墙后，首先被PREROUTING链处理，然后进行路由选择，如果数据的目标地址时防火墙本机，那么内核将其传递给INPUT链进行处理，通过处理后再交给上层应用程序。

​	**转发数据流向**：来自外界的数据包到达防火墙，首先被PREROUTING链处理，然后再进行路由选择，如果数据包的目标地址是其他外部地址，则内核将其传递给FORWARD链处理，最后交给POSTROUTING链进行处理。

​	**出站数据流向**：防火墙向外部地址发送数据包，首相被OUTPUT链进行处理，然后进进行路由选择，再交给POSTROUTING链进行处理。

##### 3、规则链内部各条防火墙规则之间的顺序。

​	当数据包经过规则链时，依次按照第一条规则，第二条规则......的顺序进行匹配和处理，链内的过滤遵循“匹配即停止”的原则，一旦找到一条匹配的规则，则不在检查本链中后续的规则。



# 编写防火墙规则



## 基本语法、控制类型

​	使用iptables命令管理，编写防火墙规则时，基本的命令的格式如下所示：

```shell
iptables [-t 表名] 管理选项 [链名][匹配条件] [-j 控制类型]
```

​	``-t``：用来指定表名，默认是filter表。

​	**表名，链名**：用来指定iptables命令所操作的表和链。

​	**管理选项**：表示iptables规则的做方式，如：插入，增加，删除，查看等。

​	**匹配条件**：用来指定要处理的处理数据包的特征，不符合指定条件的将不会处理。

​	**控制类型**：指的是数据包的处理方式，如：允许，拒绝，丢弃等。

 

#####　iptables常见的管理选项如下表所示：

![](http://www.it165.net/uploadfile/2013/0807/20130807085147828.jpg)

##### iptables常见的控制类型如下：

- ACCEPT：允许数据包通过。


- DROP：直接丢弃数据包，不给出任何回应信息。


- REJECT：拒绝数据包通过，必要时会给数据包发送端一个响应信息。


- LOG：在/var/log/messages文件中记录日志信息，然后将数据包传递给下一条规则。




### iptables规则的匹配条件

#### 1、通用匹配

​	通用匹配也称常规匹配，这种匹配方式可以独立使用，不依赖其他条件或扩展模块，常见的通用匹配包括协议匹配，地址匹配，网络接口匹配。

1. 协议pipe

   ​编写iptables规则时使用``-p 协议名``的形式指定，用来检查数据包所使用的网络协议，如：tcp、udp、icmp等。

   ​列如：编写iptables拒绝通过icmp的数据包。

```shell
[root@localhost /]# iptables -A INPUT -p icmp -j DROP
```

2. 地址匹配

   ​编写iptables规则时使用``-s源地址``或``-d目标地址``的形式指定，用来检查数据包的源地址或目标地址。

   ​列如：编写iptables拒绝转发``192.168.1.0/24``到``202.106.123.0/24``的数据包。

```shell
[root@localhost /]# iptables -A FORWARD -s 192.168.1.0/24 -d 202.106.123.0/24 -j DROP
```

3. 网络接口匹配

   ​编写iptables规则时使用``-i 接口名``和``-o 接口名``的形式，用于检查数据包从防火墙的哪一个接口进入或发出，分别对应入站网卡(--in-interface)，出站网卡(--out-interface)。

   ​列如：拒绝从防火墙的eth1网卡接口ping防火墙主机。

```shell
[root@localhost /]# iptables -A INPUT -i eth1 -p icmp -j DROP
```

#### 2、隐含匹配

​	这种匹配方式要求以指定的协议匹配作为前提条件，相当于子条件，因此无法独立使用，其对应的功能由iptables在需要时自动隐含载入内核。常见的隐含匹配包括端口匹配，TCP标记匹配，ICMP类型匹配。

1. 端口匹配

   ​编写iptable规则时使用``--sport 源端口``或``--dport``的形式，针对的协议为TCP或UDP，用来检查数据包的源端口或目标端口。单个端口或者以“：”分隔的端口范围都是可以接受的，但不支持多个不连续的端口号。

   ​列如：编写iptables规则允许FTP数据包通过，则需要允许20,21和用于被动模式的24500-24600的端口范围。

```shell
[root@localhost /]# iptables -A INPUT -p tcp --dport 20:21 -j ACCEPT``
[root@localhost /]# iptables -A INPUT -p tcp --dport 24500:24600 -j ACCEPT``
```

2. TCP标记匹配

   ​编写iptables规则时使用``--tcp-flags 检查范围 被设置的标记``的形式，针对的协议为TCP，用来检查数据包的标记位。其中“检查范围”指出需要检查数据包的那几个标记位，“被设置的标记”则明确匹配对应值为1的标记，多个标记之间以逗号进行分隔。

   ​列如：若要拒绝外网卡接口(eth1)直接访问防火墙本机的TCP请求，但其他主机发给防火墙的TCP响应等数据包应允许，可执行如下操作。

```shell
[root@localhost /]# iptables -A INPUT -i eth1 -p tcp --tcp-flags SYN，RST，ACK SYN -j DROP
```

3. ICMP类型匹配

   ​编写iptables规则时使用``--icmp-type ICMP类型``的形式，针对的协议为ICMP，用来检查ICMP数据包的类型。ICMP类型使用字符串或数字代码表示，如“Echo-quest”(代码为8)，“Echo-Reply”(代码为0)，“Destination-Unreachable”(代码为3)，分别对应ICMP协议的请求，回显，目标不可达。

   ​列如：若要禁止从其他主机ping防火墙本机，但允许防火墙本机ping其他主机，可执行以下操作。

```shell
[root@localhost /]# iptables -A INPUT -p icmp --icmp-type 8 -j DROP
[root@localhost /]# iptables -A INPUT -p icmp --icmp-type 0 -j ACCEPT
[root@localhost /]# iptables -A INPUT -p icmp --icmp-type 3 -j ACCEPT
```



#### 3、显示匹配

​	这种匹配方式要求有额外的内核模块提供支持，必须手动以“-m 模块名称”的形式调用相应的模块。然后方可设置匹配条件。常见的显示匹配包括多端口匹配，IP范围匹配，MAC地址匹配，状态匹配。

1. 多端口匹配

   ​编写iptables规则时使用``-m multiport --dport 端口列表``或``-m multiport --sport 端口列表``的形式，用来检查数据包的源端口，目标端口，多端口之间以逗号进行分隔。

   ​列如：若允许本机开放25,80,110,143等端口，以便提供电子邮件服务，可执行如下操作。

```shell
[root@localhost /]# iptables -A INPUT -p tcp -m multiport --dport 25,80,110,143 -j ACCEPT
```

2. IP地址范围匹配

   ​编写iptables规则时使用``-m iprange --src-range IP范围``，``-m -iprange --dst-range IP地址范围``的形式，用来检查数据包的源地址，目标地址，其中IP范围采用“起始地址-结束地址”的形式表示。

   ​列如：若要允许转发源地址IP位于192.168.4.21与192.168.4.28之间的TCP数据包，可执行如下操作。

```shell
[root@localhost /]# iptables -A FORWARD -p tcp -m iprange --src-range 192.168.4.21-192.168.4.28 -j ACCEPT
```

3. MAC地址匹配

   ​编写iptables规则时使用``-m mac --mac-source MAC地址``的形式，用来检查数据包的源MAC地址。由于MAC地址本身的局限性，此类匹配条件一般只适用于内部网络。

   ​列如：若要根据MAC地址封锁主机，禁止其访问本机的任何应用，可以执行如下操作。

```shell
[root@localhost /]# iptables -A INPUT -m mac --mac-source 00:0c:29:c0:55:3f -j DROP
```

4. 状态匹配

   ​编写iptables规则时使用``-m state --state`` 连接状态”的形式，基于iptables的状态跟踪机制用来检查数据包的连接状态。常见的连接状态包括NEW(如任何连接无关的)，ESTABLISHED(相应请求或者一建立连接的)，RELATED(与已有连接有相关性的，如FTP数据连接)。

   ​列如：编写iptables规则，只开放本机的80端口服务，对于发送给本机的TCP应答数据包给予放行，其他入站数据包均拒绝，可执行如下操作。

```shell
[root@localhost /]# iptables -A INPUT -p tcp -m multiport --dport 80 -j ACCEPT
[root@localhost /]# iptables -A INPUT -p tcp -m state --state ESTABLISHED -j ACCEPT
```

---------

## SNAT和DNAT

​	在配置SNAT和DNAT之前，需要开启Linux系统中的地址转发功能，否则数据无法通过防火墙转发出去。

​	修改``/etc/sysctl.conf``配置文件件，将ip_forward的值设置为1即可。

```shell
[root@localhost /]# vim /etc/sysctl.conf
......//省略部分内容
net.ipv4.ip_forwaed=1             //将此行中的0改为1
[root@localhost /]# sysctl -p     //重新读取修改后的配置
```

也可以开启临时的路由转发，可以执行以下操作。

```shell
[root@localhost /]# echo 1> /proc/sys/net/ipv4/ip_forward
```

或者

```shell
[root@localhost /]# sysctl -w net.ipv4.ip_forward=1
```



## SNAT的策略及应用

​	**SNAT**：源地址转换，是Linux防火墙的一种地址转换操作，也是iptables命令中的一种数据包控制类型，其作用是根据指定条件修改数据包的源IP地址。

​	**SNAT**策略只能用在nat表的POSTROUTING链中，使用iptables命令编写SNAT策略时，需要结合`--to-source  IP地址`选项来指定修改后的源IP地址。

​	列如：Linux网关服务器通过eth0和eth1分别连接Internet和局域网，其中eth0的IP地址为`218.29.30.31`，eth1的IP地址为`192.168.1.1`。现在要求在Linux网关服务器上配置规则，使用局域网内的所有用户可以通过共享的方式访问互联网。可执行以下操作。

1. 开启路由转发，上面已经讲过，这里不在详述了。

2. 在iptables的POSTROUTING中编写SNAT规则。
```shell
[root@localhost /]# iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j SNAT --to-source 218.29.30.31
```
3. 在某些情况下，网关的外网IP地址可能不是固定的，列如使用ADSL宽带接入时，针对这这种需求，iptables提供了一个名为MASQUERASE(伪装)的数据包控制类型，MASQUERADE相当于SNAT的一个特列，同样同来修改数据包的源IP地址只过过它能过自动获取外网接口的IP地址。

   参照上一个SNAT案例，若要使用MASQUERADE伪装策略，只需要去掉SNAT策略中的`--to-source IP地址`。然而改为`-j MASQUERADE`指定数据包的控制类型。对于ADSL宽带连接来说，连接名称通常为ppp0，ppp1等。操作如下
```shell
[root@localhost /]# iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o ppp0 -j MASQUERADE
```
4. 测试SNAT共享接入的结果

   同过上面的配置，这时内部局域网访问互联网所使用的IP地址是网关服务器的外网接口地址了。我们可以在往外客户端执行“tcpdump -i eth0”监听访问本机的数据流，然后在内网ping外网客户端，然后查看外网客户端监听的状态，会发现，访问外网客户机的IP地址是网关服务器的外网接口地址，而不是内部局域网的地址。




## DNAT策略及应用

​	**DNAT**：目标地址转换，是Linux防火墙的另一种地址转换操作，同样也是iptables命令中的一种数据包控制类型，其作用是根据指定条件修改数据包的目标IP地址，目标端口。

​	DNAT策略与SNAT策略非常相似，只不过应用方向反过来了。SNAT用来修改源地址IP，而DNAT用来修改目标IP地址，目标端口；SNAT只能用在nat表的POSTROUTING链，而DNAT只能用在nat表的PREROUTING链和OUTPUT链中。

​	列如：借助上述网络环境，公司内部局域网内搭建了一台web服务器，IP地址为`192.168.1.7`，现在需要将其发布到互联网上，希望通过互联网访问web服务器。那么我们可以执行如下操作。

1. 在iptables的PREROUTING中编写DNAT规则。
```shell
[root@localhost /]# iptables -t nat -A PREROUTING -i eth0 -d 218.29.30.31 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.7:
```
2. 再列如：公司的web服务器`192.168.1.7`需要通过互联网远程管理，由于考虑到安全问题，管理员不希望使用默认端口进行访问，这时我们可以使用DNAT修改服务的默认端口。操作如下：
```shell
[root@localhost /]# iptables -t nat -A PREROUTING -i eth0 -d 218.29.30.31 -p tcp --dport 2346 -j DNAT --to-destination 192.168.1.7:22
```
3. 在外网客户端浏览器中访问网关服务器的外网接口，可以发现访问的居然是内网192.168.1.7的web服务器的网页。而在使用sshd连接2346端口时，居然可以远程连接到192.168.1.7服务器上。