---
layout: post
title: "Everything About SSH Service"
date: 2012-01-28 14:07
comments: true
categories: [Service, SSH]
---

SSH为Secure Shell的缩写，由IETF的网络工作小组（Network Working Group）所制定；SSH为创建在应用层和传输层基础上的安全协议。在维基百科上，有<a href="http://zh.wikipedia.org/wiki/SSH" target="_blank">关于SSH的详细词条</a>，但通俗点说，SSH能够让一个客户端安全的登录上一个服务器上进行管理操作。所以，忘掉FTP、POP和Telnet吧，专心来爱SSH。

让我们从最基础的部分开始，首先假定我们有台Macbook，然后想登录上一台Ubuntu服务器进行管理操作，那么首先要求Ubuntu服务器上 安装了SSH服务。SSH服务最早是由芬兰的一家公司开发，现在已经发展到SSH2版本，但由于版权和加密算法等因素的影响，很多人开始转用 OpenSSH，听这名字，就知道它是开源和免费的。
{% blockquote %}
以下所有操作都需要具备root权限的账号，通常我们不太建议在服务器上直接登录为root，所以一般会登录为普通用户，然后通过在命令前面加上sudo来获取root权限。
{% endblockquote %}
1.我们先惯例一下
{% codeblock lang:bash %}
sudo apt-get update sudo apt-get upgrade
{% endcodeblock %}
2.然后开始安装OpenSSH服务
{% codeblock lang:bash %}
sudo apt-get install openssh-server
{% endcodeblock %}
3.Ubuntu会帮我们解决一切依赖关系问题并且安装好OpenSSH服务，接下来可以做一些配置来实现更快更安全的目的具体的修改可以参见<a href="http://wiki.ubuntu.org.cn/OpenSSH%E9%AB%98%E7%BA%A7%E6%95%99%E7%A8%8B" target="_blank">Ubuntu OpenSSH高级教程</a>。

至此安装已经结束了，下面我们可以从Macbook上登录试试，假设Ubuntu上存在一个用户tester。在Macbook上选择应用程序 – 实用工具 – 终端，然后在打开的终端里面输入

#注意这里S_IP是服务器的真实IP地址 ssh tester@S_IP
然后就会问你test的密码，输入密码就可以成功登录进行操作了。

每次都输入密码会很烦，而且也不安全，同时还有其他一些潜在的风险，所以SSH也提供基于密钥的认证机制，你必须为自己创建一对密钥，并把公钥放在 需要访问的服务器上。客户端软件会向服务器发出请求，请求用你的私匙进行安全验证。服务器收到请求之后，先在你在该服务器的用户根目录下寻找你的公钥，然 后把它和你发送过来的公钥进行比较。如果两个密钥一致，服务器就用公有密钥加密“质询”（challenge）并把它发送给客户端软件。从而避免被“中间 人”攻击。

由于之前所说的原因，会出现一种蛋疼的情况，有些公司还喜欢使用SSH2版本的SSH服务，SSH2和OpenSSH的加密算法是完全不一样的，他们所使用的的密钥对也不兼容，所以会出现下面4种组合
1. OpenSSH客户端对OpenSSH服务器
2. SSH2客户端对SSH2服务器
3. OpenSSH客户端对SSH2服务器
4. SSH2客户端对OpenSSH服务器
假设客户端C试图使用用户tester登录服务器S，我们来看看各种组合下如何使用密钥登录
