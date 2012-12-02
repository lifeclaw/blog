---
layout: post
title: "使用FreeRADIUS来管理VPN流量"
date: 2012-01-29 23:21
comments: true
categories: [Service, VPN]
published: true
---

　　<a href="/articles/how-to-install-freeradius-on-ubuntu/" rel="newtar">在Ubuntu上安装FreeRADIUS服务</a>一文描述了如何安装和配置FreeRADIUS服务和RADIUS客户端，如果要实际运用FreeRADIUS来管理VPN用户和流量的话，那么还需要一个数据库服务来存储管理数据，这里我们选用最常见的MySQL。

A.MySQL服务端的安装
{% codeblock lang:bash %}
sudo apt-get -y update
sudo apt-get -y install mysql-server
{% endcodeblock %}
MySQL服务的配置就不在此赘述了。

B.配置FreeRADIUS读写MySQL<br />
首先创建独立的用户和数据库
{% codeblock lang:bash %}
CREATE USER 'radius'@'localhost' IDENTIFIED BY '{radpass}';
CREATE DATABASE IF NOT EXISTS `radius` ;
GRANT ALL PRIVILEGES ON `radius` . * TO 'radius'@'localhost';
{% endcodeblock %}
然后导入基础数据库结构
{% codeblock lang:bash %}
cd /etc/freeradius/sql/mysql
mysql -uradius -p{radpass} radius < schema.sql
mysql -uradius -p{radpass} radius < ippool.sql
mysql -uradius -p{radpass} radius < wimax.sql
mysql -uradius -p{radpass} radius < cui.sql
mysql -uradius -p{radpass} radius < nas.sql
{% endcodeblock %}
这里请將{radpass}替换成为自己钟意的密码。接下来还要用到这个密码
{% codeblock lang:bash %}
sudo vi /etc/freeradius/sql.conf
#配置一下如下字段
password = "{radpass}"
#接下来在FreeRADIUS中打开对MySQL的使用
sudo vi /etc/freeradius/radiusd.conf
#在moudle段中找到如下内容并去除前面的注释
$INCLUDE sql.conf
{% endcodeblock %}
接下来修改一下站点配置
{% codeblock lang:bash %}
sudo vi /etc/freeradius/sites-enabled/default
#找到authorize {}模块，注释掉files（159行），去掉sql前的#号（166行）
#找到accounting {}模块，注释掉radutmp(385行),注释掉去掉sql前面的#号(395行)
#找到session {}模块，注释掉radutmp（439行），去掉sql前面的#号（443行）
#找到post-auth {}模块，去掉sql前的#号（464行），去掉sql前的#号（552行）
sudo vi /etc/freeradius/sites-enabled/inner-tunnel
#找到authorize {}模块，注释掉files（124行），去掉sql前的#号（131行）
#找到session {}模块，注释掉radutmp（251行），去掉sql前面的#号（255行）
#找到post-auth {}模块，去掉sql前的#号（277行）,去掉sql前的#号（301行）
{% endcodeblock %}
接下来重启FreeRADIUS服务
{% codeblock lang:bash %}
sudo service freeradius restart
{% endcodeblock %}

C.开始加入VPN用户
{% codeblock lang:bash %}
mysql -uradius -p{radpass};
USE radius;
# 添加用户test，密码1234
INSERT INTO radcheck (username,attribute,op,VALUE) VALUES ('test','Cleartext-Password',':=','1234');
# 限制同时登陆人数
INSERT INTO radgroupcheck (groupname,attribute,op,VALUE) VALUES ('normal','Simultaneous-Use',':=','1');
# 将用户test加入VIP用户组
INSERT INTO radusergroup (username,groupname) VALUES ('test','VIP');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('VIP','Auth-Type',':=','Local');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('VIP','Service-Type',':=','Framed-User');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('VIP','Framed-Protocol',':=','PPP');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('VIP','Framed-MTU',':=','1500');
INSERT INTO radgroupreply (groupname,attribute,op,VALUE) VALUES ('VIP','Framed-Compression',':=','Van-Jacobson-TCP-IP');
{% endcodeblock %}

D.开启流量统计
{% codeblock lang:bash %}
sudo vi /etc/freeradius/radiusd.conf
#把如下这一行的注释去掉
$INCLUDE sql/mysql/counter.conf
#counter.conf里面提供了多种计数方式，可以进去详细了解下，这里我们使用按月统计的计数方式，对应的字段名是Max-Monthly-Session和Monthly-Traffic-Limit
sudo vi /etc/freeradius/sites-enabled/default
#在authorize段的末尾205行加入
monthlycounter
sudo vi /etc/freeradius/dictionary
#在文件末尾加入
ATTRIBUTE Max-Monthly-Session 3003 integer
ATTRIBUTE Monthly-Traffic-Limit 3004 integer
#接下来操作mysql加入如下内容
#每月最大流量1G
INSERT INTO radgroupcheck (groupname,attribute,op,VALUE) VALUES ('VIP','Max-Monthly-Session',':=','1073741824');
#流量更新间隔
INSERT INTO radgroupcheck (groupname,attribute,op,VALUE) VALUES ('VIP','Acct-Interim-Interval',':=','60');
{% endcodeblock %}

最后，重启一下freeradius服务，不出意外的话就可以使用用户名test密码1234来连接VPN服务了，而该用户每月可以使用1G的流量。还有其他的玩法就留给大家继续了。
