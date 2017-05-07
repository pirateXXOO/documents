# rsyslog+mysql+loganalyzer 搭建日志服务器及监控

## rsyslog

日志:历史事件；
​	历史事件：时间、地点、事件；

​	syslog:
​		klogd:kernel
​		yslogd:system(application)

​	事件记录格式：日期时间 主机 进程[pid]:事件内容;

​	C/S架构；通tcp或udp协议的服务完成日志记录的传送；

​	rsyslog：
​		rsyslog的特性：	
​			- 多线程;
​			- UDP/TCP/SSL/TLS/RELP;
​			- 存储日志信息于MySQL,PGSQL,oracle,等RDBMS；
​			- 强大的过滤器，实现过滤日志信息中的任何部分的内容；
​			- 自定义输出格式；
​			- ......

### elk

ELK由Elasticsearch、Logstash和Kibana三部分组件组成；
​	**Elasticsearch***是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。
​	**Logstash**是一个完全开源的工具，它可以对你的日志进行收集、分析，并将其存储供以后使用
​	**kibana** 是一个开源和免费的工具，它可以为 Logstash 和 ElasticSearch 提供的日志分析友好的 Web 界面，可以帮助您汇总、分析和搜索重要数据日志。

### rsyslog日志收集器的基本术语；

​	**facility**:设施，收束日志数据流为有限几个；
​		auth,authpriv,cron,daemon,kern,lpr,mail,mark,news,security,user,uucp,syslog,local0-local7
​	**priority**:优先级
​		debuf,info,notice,warn(warning),err(error),crit(critical),alert,emerg(panic)

### 程序包：rsyslog

​	程序环境：
​		配置文件:``/etc/rsyslog.conf,/etc/rsyslog.d/\*.conf``
​		主程序：``/usr/sbin/rsyslogd``
​		CentOS 6：``service rsyslogs {start|stop|restart|status}``
​		CentOS 7：``/usr/lib/systemd/system/rsyslog.service``

​	配置文件格式：
​		由三部分组成：
​			- MODULES ：模块配置
​			- GLOBAL DIRECTIVES：全局配置
​			- RULES：日志记录相关配置
​			- ( begin forwarding rule 
​	RULES：

​		配置格式：
​			``facility.priority 	target      - 表示异步写入``

​		**facility:**
​			*:所有的facility；
​			f1,f2,f3,...:指定的facility列表;
​		**priority:**
​			*:所有级别
​			none：没有级别；
​			PRIORITY：指定级别（含）以上的所有级别；
​			=PRIORITY：仅记录指定级别的日志信息；

​		**target**：
​			文件：将日志信息记录到指定的文件中，文件路径前的-表示异步写入；
​			用户：将日志时间通知给指定的用户；
​			日志服务器：@host，把日志通过网络送往指定的服务器记录，而非由本地记录；
​			管道：|COMMAND

### 其他的日志文件：

​	``/var/log/secure``:系统安全日志，应该周期性分析；
​	``/var/log/btmp``:当前系统上用户的失败尝试登录相关的日志信息，lastb命令可以查看
​	``/var/log/wtmp``:当前系统上，用户正常登陆系统的相关日志信息，last命令可以查看

​	``lastlog``：用于查看当前系统上每一个用户最后一次登陆的信息。

​	``/var/log/messages``:系统日志信息；
​	``/var/log/desg``:系统引导过程中的日志信息；
​		文本查看工具查看；
​		也可以使用专用命令dmesg查看；

-----

## 配置rsyslog成为日志服务器；

```shell
vim /etc/rsyslog.conf

##### MODULES #####
Provides UDP syslog reception
$ModLoad imudp
$UDPServerRun 514
Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514

systemctl restart rsyslog
```


## rsyslog将日志记录于MySQL中；

#### (1)准备MySQL server
```shell
yum install mariadb mariadb-server
```


#### (2)在MySQL server上授权rsyslog能连接至当前服务器；
```shell
mysql
grant all on Syslog.\* to 'rsyslog'@'%' identified by 'magedu';
```
#### (3)在rsyslog主机上安装MySQL模块相关的程序包

```shell
yum install rsyslog-mysql
```
####(4)为rsyslog创建数据库及表；
```shell
mysql -uUSERNAME -hHOST -pPASSWORD < /usr/share/doc/rsyslog-7.4.7/mysql-createDB.sql
```
####(5)配置rsyslog将日志保存于MySQL中；
```shell
vim /etc/rsyslog.conf
### MODULES ###
$ModLoad ommysql

### RULES ###
facility.priority :ommysql:DBHOST,DBNAME,DBUSER,DBUSERPASSWORD
```
#### (6)重启rsyslog服务；
```shell
systemctl restart rsyslog
```
---

## 通过loganalyzer展示数据库中的日志：

#### (1)准备lamp或lnmp组合；

```shell
yum install -y httpd php php-mysql php-gd
```
启动服务
```shell
systemctl start httpd
systemctl start mariadb
```
#### (2)安装loganalyzer

```shell
tar xvf loganalyzer-3.6.5.tar.gz
cp -a loganalyzer-3.6.5/src /var/www/html/log
cd /var/www/html/log
touch config.php
chmod 666 config.php
```
#### (3)配置loganalyzer

```shell
systemctl start httpd
```
浏览器中输入：http://HOST/log

点击``here``

![](C:\Users\Jack\OneDrive\文档\loganalyzer\1.PNG)

点击`next`

![](C:\Users\Jack\OneDrive\文档\loganalyzer\2.PNG)

点击`next`

![](C:\Users\Jack\OneDrive\文档\loganalyzer\3.PNG)

点击`next`

![](C:\Users\Jack\OneDrive\文档\loganalyzer\4.PNG)

填写数据库信息

点击`next`

![](C:\Users\Jack\OneDrive\文档\loganalyzer\5.PNG)

点击`finish`

![](C:\Users\Jack\OneDrive\文档\loganalyzer\6.PNG)



![](C:\Users\Jack\OneDrive\文档\loganalyzer\7.PNG)



#### (4)安全加强

```shell
cd /var/www/html/log
chmod 644 config.php
```