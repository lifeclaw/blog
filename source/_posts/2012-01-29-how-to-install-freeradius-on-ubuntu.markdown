---
layout: post
title: "在Ubuntu上安装FreeRADIUS服务"
date: 2012-01-29 23:19
comments: true
categories: [Service, VPN]
---

　　FreeRADIUS是RADIUS协议的开源实现，而RADIUS主要用来实现认证(Authentication)、授权(Authorization)以及计费(Accounting)功能。这里我们安装FreeRADIUS服务的目的主要是为了实现VPN服务的用户认证和流量控制，所以安装之前请确认已经安装好了PPTP VPN服务，请参见<a href="/articles/how-to-install-pptp-vpn-service-on-ubuntu/" rel="newtar">在Ubuntu上安装PPTP VPN服务</a>。

A.FreeRADIUS服务端的安装
{% codeblock lang:bash %}
sudo apt-get -y update
sudo apt-get -y install freeradius
{% endcodeblock %}
ubuntu的自动安装确实非常智能，安装完毕后基本服务也配置好了，FreeRADIUS服务会默认启动4个网络监听进程，我们可以用如下的方式测试是否成功：
{% codeblock lang:bash %}
sudo vi /etc/freeradius/users
#在末尾添加一行
test Cleartext-Password := "5678"
#执行如下命令
radtest test 5678 localhost 0 testing123
#返回"Access-Accept packet" 表示成功，"Access-Reject" 表示失败。
{% endcodeblock %}


B.FreeRADIUS服务端的配置<br />
最重要的是一定要把默认的加密串testing123修改掉，请注意这是必须做的！这里假设你想使用abcde为加密串
{% codeblock lang:bash %}
sudo vi /etc/freeradius/clients.conf
#找到secret = testing123改成
secret = abcde
{% endcodeblock %}
<strong>如果你的FreeRADIUS服务和PPTP服务在同一台机器的话</strong>，请把FreeRADIUS的监听ip设置成localhost，并且关闭proxy代理，这样能极大增强安全性：
{% codeblock lang:bash %}
sudo vi /etc/freeradius/radiusd.conf
#请将所有listen配置段的ipaddr = *改成
ipaddr = localhost
#同时找到PROXY配置段，将其修改成如下所示
proxy_request = no
#$INCLUDE proxy.conf
{% endcodeblock %}

C.RADIUS客户端的安装和配置<br />
RADIUS客户端的作用是为了让PPTP服务能连接FreeRADIUS服务：
{% codeblock lang:bash %}
#请确保Ubuntu已经升级到11.10版本或更新的版本
sudo apt-get -y install radiusclient1
{% endcodeblock %}
接下来设置通信加密串
{% codeblock lang:bash %}
sudo vi /etc/radiusclient/servers
#在里面添加如下配置，其中abcde就是你之前所设置的加密串
localhost abcde
{% endcodeblock %}
下面这一步也挺重要，这能保证windows客户端连接的时候不出错
{% codeblock lang:bash %}
sudo cd /etc/radiusclient/
sudo wget -c http://small-script.googlecode.com/files/dictionary.microsoft
sudo vi /etc/radiusclient/dictionary
#在其中增加如下配置
INCLUDE /etc/radiusclient/dictionary.ascend
INCLUDE /etc/radiusclient/dictionary.merit
INCLUDE /etc/radiusclient/dictionary.compat
INCLUDE /etc/radiusclient/dictionary.microsoft
{% endcodeblock %}
接下来在PPTP的配置中启用RADIUS客户端连接
{% codeblock lang:bash %}
sudo vi /etc/ppp/pptpd-options
#在其中增加如下配置
#请注意64位的插件位置是plugin /usr/lib64/pppd/2.4.5/radius.so
plugin /usr/lib/pppd/2.4.5/radius.so
radius-config-file /etc/radiusclient/radiusclient.conf
{% endcodeblock %}
修改完成之后重启FreeRADIUS和PPTPD服务
{% codeblock lang:bash %}
sudo service pptpd restart
sudo service freeradius restart
{% endcodeblock %}
不出意外的话，接下来你就可以用户test，密码5678来连接VPN服务了。在此基础上，如果想要实现用户认证授权和计费等高级功能，请参见<a href="/articles/vpn-bandwidth-management-via-freeradius/" rel="newtar">使用Freeradius来管理VPN流量</a>。

