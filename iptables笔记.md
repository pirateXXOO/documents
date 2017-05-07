# iptables笔记

	iptables/netfilter
	Packets Filter Firewall：
		包过滤型防火墙：
	
	Firewall :隔离工具，工作于主机或网络边缘处，对经由的报文根据预先的规则（识别标准）进行检测，对于能够被规则匹配到的报文实行某种预定于的处理机制的一套组件；
		硬件防火墙：在硬件级别实现部分防火墙功能的防火墙；
		软件防火墙：应用软件逻辑在通用硬件基础上实现；
		主机防火墙：对单一主机进行包过滤的防火墙；
		网络防火墙：对进出某个网络的包进行过滤的防火墙；


	ipfw-->ipchians-->iptables(ip6tables)

	iptables/netfilter:
		iptables:规则管理工具
		netfilter：防火墙框架，承载并生效规则
	
		hook functions:(netfilter)
			prerouting
			input
			forward
			output
			postrouting
	
		iptables:
			PREROUTING
			INPUT
			FORWARD
			OUTPUT
			POSTROUTING
	
			允许用户自定义规则链：需要手动关联至指定的hook（钩子）；
	
		netfilter:功能
			“防火”：包过滤
			NAT：Network Address Translation 网络地址转换
			mangle：拆分报文，做出修改，然后重新封装后发送
			raw：关闭nat表上启用的连接追踪机制；
	
		功能(表)<-->钩子
			filter：	input,forward,output
			NAT： 	   prerouting,input,output,postrouting
			mangle:	 	prerouting,input,forward,output,postrouting
			raw: 		prerouting,output
	
			优先级自下而上
			优先级：raw-->mangle-->NAT-->filter


		iptables规则的组成部分：
			匹配条件：
				匹配网络层首部：SourceIP，DestinationIP，...
				传输层首部：SourcePort，DestinationPort，TCP flags(SYN,ACK,FIN,URG,PSH,RST),...
			跳转目标：target
				内建的处理机制：ACCEPT,DROP,REJECT,SNAT,DNAT,MASQUERADE,LOG,...
				用户自定义链：
	
			添加规则时需要考量的因素：
				(1)实现的功能：用于判断将规则添加至哪个表；
				(2)报文的流经的位置：用于判断将规则添加至哪个链；
				(3)报文的流向：判定规则中何为“源”，何为“目标”；
				(4)匹配条件：用于编写正确的匹配规则：
					(a)专用于某种应用的同类规则，匹配范围小的放前面；
					(b)专用于某种应用的不同类规则，匹配到的可能性较多的放前面；同一列表的规则可使用自定义链单独存放；
					(c)用于通用目的的规则放前面；
	
		filter表：过滤，“防火墙”意义的核心所在；
			INPUT,FORWARD,OUTPUT
	
	安装：
		netfilter:位于内核中的tcp/ip协议栈报文处理框架；
		iptables：
			CentOS 5/6：iptables命令生成规则，可保存于文件中以反复装载生效；
				# iptables -t filter -F
				# service iptables save
	
				man iptables
			CentOS 7:firewalld,firewall-cmd,firewall-config
				# systemctl disable firewalld.service
	
				man iptables
				man iptables-extensions
	
	iptables命令：
	   iptables [-t table] {-A|-C|-D} chain rule-specification
	   iptables [-t table] -I chain [rulenum] rule-specification
	   iptables [-t table] -R chain rulenum rule-specification
	   iptables [-t table] -D chain rulenum
	   iptables [-t table] -S [chain [rulenum]]
	   iptables [-t table] {-F|-L|-Z} [chain [rulenum]] [options...]
	   iptables [-t table] -N chain
	   iptables [-t table] -X [chain]
	   iptables [-t table] -P chain target
	   iptables [-t table] -E old-chain-name new-chain-name
	       rule-specification = [matches...] [target]
	       match = -m matchname [per-match-options]
	       target = -j targetname [per-target-options]


	    COMMANDs:
	    	iptables -vnL
	    	iptables -t net -nL
	    	iptables -t filter -nL
	    	iptables -t raw -nL
	    	链管理：
	    		-N,--new-chain chain :新建一条自定义规则链;默认为filter
	    		iptables -N input-http
	    		-X,--delete-chain [chain]:删除自定义，空的，零引用的链
	    		-F,--flush [chain]:清空指定的规则链上的规则；
	    		-E,--rename-chain oldchain newchain :重命名自定义链，零引用的链
	    		-Z,--zero [chain [rulenum]]:置零计数器；
	    			注意：每个规则都有两个计数器
	    				packets：被本规则所匹配到的所有报文的个数；
	    				bytes：被本规则所匹配到的所有报文的大小之和；
	    		-P,--policy chain target
	    			iptables -t filter -P　INPUT DROP
	    			iptables -t filter -P　INPUT ACCEPT
	
	    	规则管理：
	    		-A,--append chain roule-specification：追加新规则于指定链的尾部
	    		-I,--insert chain [rulenum] roule-specification,插入新规则于指定链的指定位置，默认放到第一个位置
	    		-R,--replece chain roulenum roule-specification:替换指定的规则为新的规则
	    		-D,--delete chain rulenum：根据规则编号删除规则
	    		-D,--delete chain roule-specification:根据规则本身删除规则
	    	规则显示：
	    		-L,--list [chain]:列出规则；
	    			-v, --verbose：详细信息；
	    				-vv,-vvv
	    			-n, --numeric：数字方式显示地址和端口号；
	    			-x, --exact：显示计数器的精确值，而不是圆整后的值
	    			--line-numbers：列出规则时，显示其在链上的编号
	
	    		-S, --list-rules [chain]：显示指定链的所有规则
	
	    	rule-specification = [matches...] [target]
	    		matches:匹配条件
	    		target:跳转目标
	
	    		匹配条件:
	    			通用匹配(PARAMETERS)
	    				[!] -s, --source address[/mask][,...]：检查报文的源ip地址是否符合此处指定的范围，或是否等于此处指定的地址
	    				[!] -d, --destination address[/mask][,...]:检查报文的目标ip地址是否符合此处指定的范围，或是否等于此处指定的地址
	
	    					iptables -t filter -A INPUT -s 172.16.0.0/16,127.0.0.0/8 -d 172.16.0.6,127.0.0.1 -j ACCEPT
	    					iptables -t filter -A INPUT -s 172.16.0.0/16 -d 172.16.0.6 -j ACCEPT
	    					iptables -t filter -A OUTPUT -s 172.16.0.6/16 -d 172.16.0.6/16 -j ACCEPT
	
	    				[!] -p, --protocol protocol:匹配报文中的协议
	    					tcp, udp, udplite, icmp,icmpv6,esp, ah, sctp, mh 或者 "all",也可以使用数字格式指定协议
	    				-m, --match match：调用指定的扩展匹配模块来扩展匹配条件检查机制
	    				-j, --jump target
	    				-g, --goto chain
	    				[!] -i, --in-interface name：限定报文只能从指定的接口流入only  for  packets  entering the INPUT, FORWARD and PRE‐ROUTING chains
	    				[!] -o, --out-interface name：限定报文只能从指定的接口流出for  packets  entering  the  FORWARD,  OUTPUT  and  POSTROUTING chains


	    			扩展匹配(MATCH EXTENSIONS)
	    				条件扩展
	    				target扩展
	
	    			跳转目标：
	    				target = -j targetname [per-target-options]
		    				ACCEPT
		    				DROP
		    				REJECT
	
		练习：
			1、开放本机的所有tcp服务给所有主机0.0.0.0/0；
				iptables -t filter -A INPUT -p tcp -d 0.0.0.0/0 -j ACCEPT
				iptables -t filter -A OUTPUT -p tcp -s 0.0.0.0/0 -j ACCEPT
	
			2、开放本机的所有udp服务给所有172.16.0.0/16网络中的主机，但不包含172.16.0.200
				iptables -t filter -A OUTPUT -p udp  -s 172.16.0.200 -j DROP
				iptables -t filter -A OUTPUT -p udp  -s 172.16.0.0/16 -j ACCEPT
	
			3、默认策略为REJECT
				iptables -P INPUT -j REJECT
	
			扩展：
			1、仅开放本机的ssh服务给172.16.0.0/16中的主机，而且不包括172.16.0.200
				iptables -t filter -A INPUT  -p tcp --dport 22 -s 172.16.0.200 -j DROP
				iptables -t filter -A INPUT  -p tcp --dport 22 -s 172.16.0.0/16 -j ACCEPT
	回顾：
	iptables/netfilter:
		netfilter:框架
		iptables：管理规则的工具，CentOS 7 -- firewalld
	
		四表：filter，nat,mangle,raw
		五链:PREROUTING,INPUT,FORWARD,OUTPUT,POSTROUTING
	
	CRUD:create,read,update,delete
	
	iptables命令：
		iptables [-t table ] COMMAND [chain]  [PARAMETERS] [-m matchname ] [per-match-options] [-j targetname [per-target-options]]  
	
		PARAMETERS:
			[!] -s,-d,-p{tcp|udp|scmp|sctp|udplite|...},-i,-o
	
		COMMAND:
			链管理：-P,-X,-N,-E,-F,-Z
			规则管理：-A,-I,-R,-D [match，raw member],
			查看：
				-L [-v,-n,-x,--line-mnumbers]
				-S
	
		-j targetname
			ACCEPT,DROP,REJECT

# iptables(2)

	iptables [-t table ] COMMAND [chain]  [PARAMETERS] [-m matchname ] [per-match-options] [-j targetname [per-target-options]]  
		基本匹配条件：PARAMETERS
		扩展匹配条件：
			隐式扩展：在使用-p选项指明了特定的协议时，无需再使用-m选项指明扩展模块的扩展机制
			显式扩展：必须使用-m选项指明要调用的模块的扩展机制；
	
			隐式扩展；
				-p tcp：可直接使用tcp的扩展模块的专用选项：
					[!] --source-port,--sport port[:port]匹配报文源端口；可以给出多个端口，但只能是连续的端口范围；
					[!] --destination-port,--dport port[:port]匹配报文目的端口；可以给出多个端口，但只能是连续的端口范围；
					[!] --tcp-flags mask comp匹配报文中的tcp协议的标志位：Flags are: SYN ACK FIN RST URG PSH ALL NONE.
						mask:要检查的FLAGS list，以逗号分隔；
						comp：在mask给定的诸多FLAGS中其值必须为1的FLAGS列表，余下的其值必须为0
						--tcp-flags SYN,ACK,FIN,RST SYN  (SYN=1,其他为0)
						--tcp-flags ALL ALL
						--tcp-flags ALL NONE
					[!] --syn：--tcp-flags SYN,ACK,FIN,RST SYN



					iptables -t filter -P INPUT DROP
					iptables -t filter -P OUTPUT DROP
	
					iptables -A INPUT -s 172.16.0.0/16 -d 172.16.0.6 -p tcp --dport 22 -j ACCEPT
	
					tcpdump -i eno16777736 -nn tcp port 22 (抓包)
	
					iptables -A OUTPUT -s 172.16.0.6 -d 172.16.0.0/16 -p tcp --sport 22 -j ACCEPT
	
					iptables -I INPUT -s 172.16.0.200 -d 172.16.0.6 -p tcp --dport 22 -j REJECT


					iptables -N clean_invalid_packets
					iptables -A clean_invalid_packets -p tcp --tcp-flags ALL ALL -j REJECT
					iptables -A clean_invalid_packets -p tcp --tcp-flags ALL NONE -j REJECT
					iptables -I INPUT -d 172.16.0.6 -j clean_invalid_packets
	
					iptables -D INPUT 1
					iptables -F clean_invalid_packets
					iptables -X clean_invalid_packets
	
					iptables -A INPUT --syn (只允许三次握手第一次)
	
				-p udp :可直接使用udp协议扩展模块的专用选项：
					[!] --source-port,--sport port[:port]
					[!] --destination-port,--dport port[:port]
	
					iptables -P INPUT DROP
					iptables -P OUTPUT DROP
	
					iptables -P INPUT ACCEPT
					iptables -P OUTPUT ACCEPT
	
					iptables -A INPUT -d 172.16.0.7 -j REJECT
					iptables -A OUTPUT -s 172.16.0.7 -j REJECT
	
					iptables -I INPUT -d 172.16.0.7 -p udp --dport 137:138 -j ACCEPT
					iptables -I OUTPUT -s 172.16.0.7 -p udp --sport 137:138 -j ACCEPT
	
					iptables -I INPUT 2 -d 172.16.0.7 -p tcp --dport 139 -j ACCEPT
					iptables -I INPUT -d 172.16.0.7 -p udp --dport 445 -j ACCEPT
					iptables -I INPUT 2 -s 172.16.0.7 -p tcp --sport 139 -j ACCEPT
					iptables -I INPUT -s 172.16.0.7 -p udp --sport 445 -j ACCEPT
	
				[!] --icmp-type {type[/code]|typename}
					iptables -I INPUT -d 172.16.0.7 -p icmp --icmp-type 8 -j ACCEPT
					tcpdump -i eno16777736 -nn icmp  
					iptables -I OUTPUT -s 172.16.0.7 -p icmp --icmp-type 0 -j ACCEPT
	
					0/0:echo reply
					8/0:echo request
	
					udplite


			显式扩展：必须使用-m选项指明要调用的模块的扩展机制；
				1、multiport
					以离散或连续的方式定义多端口匹配条件，最多15个
					 It can only be used in conjunction with one  of  the       following protocols: tcp, udp, udplite, dccp and sctp.
	
					[!] --source-ports,--sports port[,port|,port:port]...指定多个源端口
					[!] --destination-ports,--dports port[,port|,port:port]...指定多个目标端口
					[!] --ports port[,port|,port:port]...
	
					iptables -F
					iptables -A INPUT -d 172.16.0.7 -j REJECT
					iptables -A OUTPUT -s 172.16.0.7 -j REJECT
	
					iptables -I INPUT  -d 172.16.0.7 -p tcp -m multiport --dports 22,80,139,445,3306 -j ACCEPT
					iptables -I OUTPUT  -s 172.16.0.7 -p tcp -m multiport --sports 22,80,139,445,3306 -j ACCEPT
	
				2、iprange
					以连续地址块的方式来指明多IP地址匹配条件
					[!] --src-range from[-to]
					[!] --dst-range from[-to]
					iptables -I INPUT -d 172.16.0.7 -p tcp -m multiport --dports 22,80,139,445,3306 -m iprange --src-range 127.16.0.61-172.16.0.70 -j REJECT
				3、time
					This  matches  if the packet arrival time/date is within a given range. All options are optional, but are ANDed  when  specified.  All times are interpreted as UTC by default.
	
					--datestart YYYY[-MM[-DD[Thh[:mm[:ss]]]]]
					--datestop YYYY[-MM[-DD[Thh[:mm[:ss]]]]]
					--timestart hh:mm[:ss]
					--timestop hh:mm[:ss]
					[!] --weekdays day[,day...]
					[!] --monthdays day[,day...]
					--kerneltz:使用内核配置的时区而非默认的UTC时间
	
					iptables -I INPUT -d 172.16.0.7 -p tcp --dport 80 -m time --timestart 09:00 --timestop 18:00 --weekdays 1,2,3,4,5 -j REJECT
					iptables -R INPUT 1  -d 172.16.0.7 -p tcp --dport 80 -m time --timestart 09:00 --timestop 18:00 --weekdays 6,7 --kerneltz -j REJECT
	
				4、string
					This modules matches a given string by using some pattern match‐ing strategy. It requires a linux kernel >= 2.6.14.
	
					--algo {bm|kmp}Select  the pattern matching strategy. (bm = Boyer-Moore,kmp = Knuth-Pratt-Morris)
	
					--from offset
					--to offset
	
					[!] --string pattern
					[!] --hex-string pattern
	
					iptables -D INPUT 2 //删除all REJECT
					iptables -D INPUT 1 //删除时间限定
					iptables -I OUTPUT [-s 172.16.0.7] -m string --algo bm --string "gay" -j REJECT
	
				5、connlimit
					Allows you to restrict the number of parallel connections  to  a server per client IP address (or client address block).
	
					--connlimit-upto n
					--connlimit-above n
					--connlimit-mask prefix_length
					--connlimit-saddr
					--connlimit-daddr
	
					# allow 2 telnet connections per client host
					iptables -A INPUT -p tcp --syn --dport  23  -m  connlimit    --connlimit-above 2 -j REJECT
					# you can also match the other way around:
					iptables  -A  INPUT  -p tcp --syn --dport 23 -m connlimit --connlimit-upto 2 -j ACCEPT
	
					iptables -I INPUT -d 127.16.0.7 -p tcp --syn --dport 22 -m connlimit --connlimit-above 2 -j REJECT
	
				6、limit
					This  module matches at a limited rate using a token bucket fil‐ter.  A rule using this extension will match until this limit is reached.   It  can be used in combination with the LOG target to give limited logging, for example.
	
					--limit rate[/second|/minute|/hour|/day]
					--limit-burst number
					iptables -I INPUT -d 127.16.0.7 -p icmp --icmp-type 8 -m limit --limit 20/minute --linit-burst 5 -j ACCEPT
					iptables -I OUTPUT -s 127.16.0.7 -p icmp --icmp-type 0 -j ACCEPT
						
					yum info hping3
	
					hping3 -h
					hping3 
					    --fast
						--faster
						--flood 
	
				7、state
					The  "state"  extension  is  a subset of the "conntrack" module."state" allows access to the connection tracking state for  this packet.
	
					[!] --state state
						Only a subset of the  states  unterstood by "conntrack" are recognized: INVALID, ESTABLISHED, NEW,RELATED or UNTRACKED. 
	
						NEW:新连接请求
						ESTABLISHED:已建立的连接；
						INVALID:无法识别的连接；
						RELATED:相关联的连接，当前连接是一个新请求，但附属于某个已存在的连接；
						UNTRACKED:未追踪的连接；
	
						iptables -I INPUT -d 172.16.0.7 -p tcp -m multiport --dports 22,80,139,445,3306 -m state --state NEW  -j ACCEPT
						
						iptables -I INPUT -m state --state 	ESTABLISHED -j ACCEPT
						iptables -I OUTPUT -m state --state 	ESTABLISHED -j ACCEPT
	
						iptables -I INPUT 2 -d 172.16.0.7 -p icmp --icmp-type 8 -m state --state NEW -j ACCEPT

回顾：iptables规则

	iptables [-t table] SUBCOMMAND [CHAIN] [parameters] [-m matchname [per-match-options]] [-j targetname [per-target-options]]
		match:
			multiport,iprange,time,connlimit,limit,string,state
	
		state:
			conntrack
	
			NEW,ESTABLISHED,RELATED,INBALID,UNTRACKED
	
			追踪到的连接/proc/net/nf_conntrack 		
			能追踪到的最大连接数量定义在/proc/sys/net/nf_conntrack_max

# iptables(3)	

	state扩展：
		内核模块装载：
		lsmod
		modprobe nf_conntrack
		modprobe nf_conntrack_ipv4
	
		手动装载：
			modprobe nf_conntrack_ftp
	
		/etc/sysconf/iptables-confg
			IPTABLES_MODULES="nf_conntrack_ftp"		
	
		iptables -A INPUT -d 172.16.0.6 -j REJECT
		iptables -A OUTPUT -s 172.16.0.6 -j REJECT
	
		iptables -I OUTPUT -s 172.16.0.6 --state ESTABLISHED -j ACCEPT
		iptables -I INPUT -d 172.16.0.6 --state ESTABLISHED -j ACCEPT
	
		iptables -I INPUT 2 -d 172.16.0.6 -p tcp --dport 21 -m state --state NEW -j ACCEPT
		// 删除 iptables -I OUTPUT 2 -s 172.16.0.6 -p tcp --sport 21 -m state --state NEW -j ACCEPT
	
		iptables -R INPUT -d 172.16.0.6	-m state -- state ESTABLISHED,RELATED -j ACCEPT
	
		iptables -R INPUT -d 172.16.0.6 -p tcp -m multiport --dports 21,22,80,3306 -m state --state NEW -j ACCEPT
	
	处理动作(跳转目标)
		-j targetnaem [per-target-options]
			简单target：
				ACCEPT,DROP
	
			扩展target
				rpm -ql iptabls
	
				man iptables-extensions
	
				REJECT
					This is used to send back an error packet  in  response  to  the matched  packet:  otherwise  it is equivalent to DROP so it is a terminating TARGET, ending rule traversal.
	
					--reject-with type
						The    type    given    can    be icmp-net-unreachable, icmp-host-unreachable,icmp-port-unreachable,icmp-proto-unreachable, icmp-net-prohibited,icmp-host-prohibited, or icmp-admin-prohibited (*), which return     the    appropriate ICMP error message (icmp-port-unreachable  is  the  default).
	
				LOG
					Turn on kernel logging of matching packets.
	
					iptables -I INPUT 2 -d 172.16.0.6 -p tcp -m multiport --dports 21,22,80,3306 -m state --state NEW -j LOG --log-prefix "klogd:"
	
					--log-level
					--log-prefix
	
	保存和载人规则：
		保存：iptables-save > /etc/sysconfig/iptables-20170109v1(/PATH/TO/SOME_RULE_FILE)
			
			CentOS 6:
				保存规则：
					service iptables save
					保存规则于/etc/sysconfig/iptables文件，覆盖保存；
					等于；iptables-save > /etc/sysconfig/iptables
				重载规则
					service iptables restart
					默认重载/etc/sysconfig/iptables文件中的规则
	
				配置文件：/etc/sysconfig/iptables-config


			CentOS 7:
				(1)自定义Unit File,进行iptables-restore;
				(2)firewalld服务；
				(3)自定义脚本；
		重载：
			iptables-restore < /etc/sysconfig/iptables-20170109v1s (/PATH/FROM/SOME_RULE_FILE)
			iptables-restore
				-n,--noflush:不清空原有规则；
				-t,--test:仅分析生成规则集，但不提交
	
				通常调度器不要开启ipptables
	
	规则优化的思路；
		使用自定义链管理特定应用的相关规则；
		
		(1)优先放行双方向状态为ESTABLISED的报文；
		(2)服务于不同类别的功能的规则，匹配到报文可能性更大的放前面；
		(3)服务于同一类别的功能的规则，匹配条件较为严格的放在前面；
		(4)设置默认策略，白名单机制
			(a)iptables -P，不建议
			(b)建议在规则的最后定义规则作为默认策略；
	
	iptabls/netfilter网络防火墙：
		(1)网关；
		(2)filter表的FORWARD链；


		tcpudmp -i eno16777736 -nn icmp
		echo 1 >  /proc/sys/net/ipv4/ip_forward
	
		iptables -A FORWARD -j REJECT
	
		iptables -I FORWARD -d 192.168.21.5 -p tcp --dport 80 -j ACCEPT
		iptables -I FORWARD  2 -s 192.168.21.5 -p tcp --sport 80 -j ACCEPT
	
		iptables -D FORWARD 1
		iptables -D FORWARD 1
	
		iptables -I FORWARD -m state --state ESTABLISHED -j ACCEPT
		iptables -I FORWARD 2 -d 192.168.21.5 -p tcp -m multiport --dports 21,22,80,3306 -m state --state NEW -j ACCEPT
	
		lsmod 
		modprobe nf_conntrack_ftp
		iptables -I FORWARD 3 -d 192.168.21.5 -m state --state RELATED -j ACCEPT
	
		要注意的问题：	
			(1)请求和响应报文均会经由FORWARD链，要注意规则的方向性；
			(2)如果要启用conntrack机制，建议将双方向的状态为ESTABLISHED的报文直接ACCEPT	
	
		iptables -F FORWARD
	
		NAT Network Address Translate
			请求报文：由管理员定义；
			响应报文：有NAT的conntrack机制自动实现。
	
			请求报文：
				改源地址：SNAT
				改目标地址：DNAT
	
		iptables/netfilter:
			NAT定义在nat表：
				PREROUTING,INPUT,OUTPUT,POSTROUTING
	
				SNAT:POSTROUTING
				DNAT:PREROUTING
	
				PAT:
	
		target:
			SNAT:
				This target is only valid in the nat table, in  the  POSTROUTING and  INPUT chains, 
	
				--to-source [ipaddr[-ipaddr]][:port[-port]]
				--random
				--persistent
	
					iptables -t nat -A POSTROUTING -s 192.168.21.0/24 -j SNAT --to-source 172.16.0.6
	
					tcpdump -i eno33554984 -nn icmp
					tcpdump -i eno16777736 -nn icmp
	
					iptables -t filter -A FORWARD -s 192.168.21.0/24 -p tcp --dport 80 -j REJECT
	
		iptables -t nat -F FORWARD
	
			DNAT:	
				This  target  is  only valid in the nat table, in the PREROUTING and OUTPUT chains,
	
				--to-destination [ipaddr[-ipaddr]][:port[-port]]
	
					iptables -t nat -A PREROUTING -d 172.16.0.6 -p tcp --dport 80 -j DNAT --to-destination 192.168.21.5
	
					iptables -t nat -R PREROUTING 1 -d 172.16.0.6 -p tcp --dport 80 -j DNAT --to-destination 192.168.21.5:8080


			MASQUERADE
		    	This  target  is only valid in the nat table, in the POSTROUTING chain. 
	
		      	SNAT场景中应用于POSTROUTING链上的规则实现原地址转换，但外网地址不固定时，使用此target。
	
		    REDIRECT
	   			This  target  is  only valid in the nat table, in the PREROUTING and OUTPUT chains, 
	
	   			--to-ports port[-port]
	
	   				iptables -t nat -A PREROUTING -d 172.16.0.67 -p tcp --dport 80 -j REDIRECT --to-ports 8090
	
	   				iptables -t nat vnL


博客作业：iptables/netfilter入门到进阶。