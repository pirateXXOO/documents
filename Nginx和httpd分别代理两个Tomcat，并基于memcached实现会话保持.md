# Nginx/httpd 代理两个Tomcat

## Nginx

## 前端代理服务器

`172.16.71.1`

从ftp下载Nginx

安装

``` shell
vim /etc/nginx/nginx.conf

http {
...
# 添加以下内容
upstream tcsrvs {
         server 172.16.71.4:8080;
         server 172.16.71.5:8080;
     }
...
 }
```

```shell
vim /etc/nginx/conf.d/default.conf

location / {
    root   /usr/share/nginx/html;
    proxy_pass http://tcsrvs/;  #添加此行
    index  index.html index.htm;
}
```

## 后端Tomcat服务器

`172.16.71.4`和`172.16.71.5`

两个主机都要安装jdk和Tomcat

```shell
yum install java-1.8.0-openjdk-devel -y
yum install tomcat -y
```

主机1

```shell
mkdir -pv /var/lib/tomcat/webapps/test/{lib,classes,WEB-INF}
vim /var/lib/tomcat/webapps/test/index.jsp


<%@ page language="java" %>
<html>
  <head><title>TomcatA</title></head>
  <body>
    <h1><font color="red">TomcatA.magedu.com</font></h1>
    <table align="centre" border="1">
      <tr>
        <td>Session ID</td>
    <% session.setAttribute("magedu.com","magedu.com"); %>
        <td><%= session.getId() %></td>
      </tr>
      <tr>
        <td>Created on</td>
        <td><%= session.getCreationTime() %></td>
     </tr>
    </table>
  </body>
</html>


systemctl start tomcat 
```

主机2
```shell
mkdir -pv /var/lib/tomcat/webapps/test/{lib,classes,WEB-INF}
vim /var/lib/tomcat/webapps/test/index.jsp


<%@ page language="java" %>
<html>
  <head><title>TomcatB</title></head>
  <body>
    <h1><font color="red">TomcatB.magedu.com</font></h1>
    <table align="centre" border="1">
      <tr>
        <td>Session ID</td>
    <% session.setAttribute("magedu.com","magedu.com"); %>
        <td><%= session.getId() %></td>
      </tr>
      <tr>
        <td>Created on</td>
        <td><%= session.getCreationTime() %></td>
     </tr>
    </table>
  </body>
</html>

systemctl start tomcat 
```

### 前端代理

```shell
systemctl start nginx 
```



### 测试

从浏览器访问`172.16.71.1`,显示Tomcat欢迎页面

从浏览器访问`172.16.71.1/test`,显示测试页面信息，刷新会变化。

-----

## httpd

```shell
yum install httpd -y

vim /etc/httpd/conf.d/tomcat_httpd.conf

<proxy balancer://tcsrvs>
    BalancerMember http://172.16.71.4:8080
    BalancerMember http://172.16.71.5:8080
    ProxySet lbmethod=byrequests
</Proxy>
<VirtualHost *:80>
    ServerName www.magedu.com
    ProxyVia On
    ProxyRequests Off 
    ProxyPreserveHost On
    <Proxy *>
        Require all granted
    </Proxy>
        ProxyPass / balancer://tcsrvs/
        ProxyPassReverse / balancer://tcsrvs/
    <Location />
        Require all granted
    </Location>
</VirtualHost>    

systemctl start httpd
```

### 测试

从浏览器访问`172.16.71.1`,显示Tomcat欢迎页面

从浏览器访问`172.16.71.1/test`,显示测试页面信息，刷新会变化。

------

# 实现会话粘性方法

`172.16.71.4`

```shell
vim /etc/tomcat/server.xml

	<Engine name="Catalina" defaultHost="localhost" jvmRoute="TomcatA"> #修改此行
	
	
systemctl restart tomcat
```


`172.16.71.5`

```shell
vim /etc/tomcat/server.xml

	<Engine name="Catalina" defaultHost="localhost" jvmRoute="TomcatB"> #修改此行
	
systemctl restart tomcat
```

`172.16.71.1`

```shell
vim /etc/httpd/conf.d/tomcat_httpd.conf

Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" env=BALANCER_ROUTE_CHANGED
<proxy balancer://tcsrvs>
	BalancerMember http://172.18.100.67:8080 route=TomcatA loadfactor=1
	BalancerMember http://172.18.100.68:8080 route=TomcatB loadfactor=2
	ProxySet lbmethod=byrequests
	ProxySet stickysession=ROUTEID
</Proxy>
<VirtualHost *:80>
	ServerName lb.magedu.com
	ProxyVia On
	ProxyRequests Off
	ProxyPreserveHost On
	<Proxy *>
		Require all granted
	</Proxy>
	ProxyPass / balancer://tcsrvs/
	ProxyPassReverse / balancer://tcsrvs/
	<Location />
		Require all granted
	</Location>
</VirtualHost>	

systemctl restart httpd
```

### 测试

从浏览器访问`172.16.71.1`,显示Tomcat欢迎页面

从浏览器访问`172.16.71.1/test`,显示测试页面信息，刷新不会变化。

---

# 启用管理接口

```shell
vim /etc/httpd/conf.d/balaner_manager.conf

<Location /balancer-manager>
	SetHandler balancer-manager
	ProxyPass !
	Require all granted
</Location>				

systemctl restart httpd

```

### 测试

浏览器访问`172.16.71.1/balancer-manager`会出现管理接口



### ajp balancer

```shell
vim /etc/httpd/conf.d/tomcat-ajp.conf

<proxy balancer://tcsrvs>
	BalancerMember ajp://172.16.71.4:8009
	BalancerMember ajp://172.16.71.5:8009
	ProxySet lbmethod=byrequests
</Proxy>
<VirtualHost *:80>
	ServerName lb.magedu.com
	ProxyVia On
	ProxyRequests Off
	ProxyPreserveHost On
	<Proxy *>
		Require all granted
	</Proxy>
	ProxyPass / balancer://tcsrvs/
	ProxyPassReverse / balancer://tcsrvs/
	<Location />
		Require all granted
	</Location>
	<Location /balancer-manager>
		SetHandler balancer-manager
		ProxyPass !
		Require all granted
	</Location>
</VirtualHost>	
```

### 测试

从浏览器访问`172.16.71.1`,显示Tomcat欢迎页面

从浏览器访问`172.16.71.1/test`,显示测试页面信息，刷新会变化。



## 会话绑定与前面相似

```shell
vim /etc/httpd/conf.d/tomcat-ajp.conf

<proxy balancer://tcsrvs>
	BalancerMember ajp://172.16.71.4:8009 route=TomcatA loadfactor=1
	BalancerMember ajp://172.16.71.5:8009 route=TomcatB loadfactor=1
	ProxySet lbmethod=byrequests
</Proxy>
<VirtualHost *:80>
	ServerName lb.magedu.com
	ProxyVia On
	ProxyRequests Off
	ProxyPreserveHost On
	<Proxy *>
		Require all granted
	</Proxy>
	ProxyPass / balancer://tcsrvs/
	ProxyPassReverse / balancer://tcsrvs/
	<Location />
		Require all granted
	</Location>
	<Location /balancer-manager>
		SetHandler balancer-manager
		ProxyPass !
		Require all granted
	</Location>
</VirtualHost>	
```

------

# Tomcat集群

`172.16.71.4`

```shell
vim /etc/tomcat/server.xml

     <Engine name="Catalina" defaultHost="localhost" jvmRoute="TomcatA"> # jvmRoute 一定要有

        <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
                 channelSendOptions="8">

          <Manager className="org.apache.catalina.ha.session.DeltaManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"/>

          <Channel className="org.apache.catalina.tribes.group.GroupChannel">
            <Membership className="org.apache.catalina.tribes.membership.McastService"
                        address="228.0.71.4"
                        port="45564"
                        frequency="500"
                        dropTime="3000"/>
            <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                      address="172.16.71.4"
                      port="4000"
                      autoBind="100"
                      selectorTimeout="5000"
                      maxThreads="6"/>

            <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
              <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
            </Sender>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch15Interceptor"/>
          </Channel>

          <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
                 filter=""/>
          <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve"/>

          <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
                    tempDir="/tmp/war-temp/"
                    deployDir="/tmp/war-deploy/"
                    watchDir="/tmp/war-listen/"
                    watchEnabled="false"/>

          <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
        </Cluster>
```

```shell
cp /etc/tomcat/web.xml  /var/lib/tomcat/webapps/test/WEB-INF/
vim /var/lib/tomcat/webapps/test/WEB-INF/web.xml 

<distributable/> # 添加

systemctl restart tomcat
```



`172.16.71.5`同理

```shell
vim /etc/tomcat/server.xml

     <Engine name="Catalina" defaultHost="localhost" jvmRoute="TomcatB"> # jvmRoute 一定要有

        <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
                 channelSendOptions="8">

          <Manager className="org.apache.catalina.ha.session.DeltaManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"/>

          <Channel className="org.apache.catalina.tribes.group.GroupChannel">
            <Membership className="org.apache.catalina.tribes.membership.McastService"
                        address="228.0.71.4"
                        port="45564"
                        frequency="500"
                        dropTime="3000"/>
            <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                      address="172.16.71.5"
                      port="4000"
                      autoBind="100"
                      selectorTimeout="5000"
                      maxThreads="6"/>

            <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
              <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
            </Sender>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch15Interceptor"/>
          </Channel>

          <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
                 filter=""/>
          <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve"/>

          <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
                    tempDir="/tmp/war-temp/"
                    deployDir="/tmp/war-deploy/"
                    watchDir="/tmp/war-listen/"
                    watchEnabled="false"/>

          <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
        </Cluster>
```

```shell
cp /etc/tomcat/web.xml  /var/lib/tomcat/webapps/test/WEB-INF/
vim /var/lib/tomcat/webapps/test/WEB-INF/web.xml 

<distributable/> # 添加

systemctl restart tomcat
```

### 测试

从浏览器访问`172.16.71.1`,显示Tomcat欢迎页面

从浏览器访问`172.16.71.1/test`,显示测试页面信息，刷新不会变化。



> 注意：一定要先为主机配置网关，否则Tomcat将无法启动

-----

# memcached 实现session存储

`172.16.71.4`和`172.16.71.5`

```shell
vim /etc/tomcat/server.xml

# 删除上一步添加的内容

systemctl stop tomcat
```

```shell
yum install memcached -y

systemctl start memcached
```

下载类库

```shell
cd /usr/share/tomcat/lib
lftp 172.16.0.1
cd pub/Sources/7.x86_64/msm/
mget *
bye
```

```shell
vim /etc/tomcat/server.xml

#在<Host>中添加：
<Context path="/" docBase="test" reloadable="true">
</Context>

systemctl start tomcat
```

### 测试

从浏览器访问`172.16.71.1`,显示Tomcat欢迎页面

从浏览器访问`172.16.71.1/test`,显示测试页面信息，刷新不会变化。

### 添加session保持

`172.16.71.4` `172.16.71.5`

```shell
vim /etc/tomcat/server.xml

      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
                <Context path="/sample" docBase="sample" reloadable="true">
                  <Manager className="de.javakaffee.web.msm.MemcachedBackupSessionManager"
                      memcachedNodes="n1:172.16.71.4:11211,n2:172.16.71.5:11211"
                      failoverNodes="n1"
                      requestUriIgnorePattern=".*\.(ico|png|gif|jpg|css|js)$"
                      transcoderFactoryClass="de.javakaffee.web.msm.serializer.javolution.JavolutionTranscoderFactory"
 

systemctl restart tomcat
```

### 测试

从浏览器访问`172.16.71.1`,显示Tomcat欢迎页面

从浏览器访问`172.16.71.1/sample`,显示测试页面信息，刷新session不会变化，内容会变化。

> 注意：此处前端要使用Nginx，httpd不能实现以上功能，原因未知。

