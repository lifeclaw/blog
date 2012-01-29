---
layout: post
title: "所有关于SSH的一切"
date: 2012-01-28 14:07
comments: true
categories: [Service, SSH]
---

　　SSH为Secure Shell的缩写，由IETF的网络工作小组（Network Working Group）所制定；SSH为创建在应用层和传输层基础上的安全协议。在维基百科上，有<a href="http://zh.wikipedia.org/wiki/SSH" target="_blank">关于SSH的详细词条</a>，但通俗点说，SSH能够让一个客户端安全的登录上一个服务器上进行管理操作。所以，忘掉FTP、POP和Telnet吧，专心来爱SSH。

　　让我们从最基础的部分开始，首先假定我们有台Macbook，然后想登录上一台Ubuntu服务器进行管理操作，那么首先要求Ubuntu服务器上 安装了SSH服务。SSH服务最早是由芬兰的一家公司开发，现在已经发展到SSH2版本，但由于版权和加密算法等因素的影响，很多人开始转用 OpenSSH，听这名字，就知道它是开源和免费的。
{% blockquote %}
以下所有操作都需要具备root权限的账号，通常我们不太建议在服务器上直接登录为root，所以一般会登录为普通用户，然后通过在命令前面加上sudo来获取root权限。
{% endblockquote %}
A.我们先惯例一下
{% codeblock lang:bash %}
sudo apt-get update sudo apt-get upgrade
{% endcodeblock %}
B.然后开始安装OpenSSH服务
{% codeblock lang:bash %}
sudo apt-get install openssh-server
{% endcodeblock %}
C.Ubuntu会帮我们解决一切依赖关系问题并且安装好OpenSSH服务，接下来可以做一些配置来实现更快更安全的目的具体的修改可以参见<a href="http://wiki.ubuntu.org.cn/OpenSSH%E9%AB%98%E7%BA%A7%E6%95%99%E7%A8%8B" target="_blank">Ubuntu OpenSSH高级教程</a>。

　　至此安装已经结束了，下面我们可以从Macbook上登录试试，假设Ubuntu上存在一个用户tester。在Macbook上选择应用程序 – 实用工具 – 终端，然后在打开的终端里面输入
{% codeblock lang:bash %}
#注意这里S_IP是服务器的真实IP地址
ssh tester@S_IP
{% endcodeblock %}
然后就会问你test的密码，输入密码就可以成功登录进行操作了。

　　每次都输入密码会很烦，而且也不安全，同时还有其他一些潜在的风险，所以SSH也提供基于密钥的认证机制，你必须为自己创建一对密钥，并把公钥放在 需要访问的服务器上。客户端软件会向服务器发出请求，请求用你的私匙进行安全验证。服务器收到请求之后，先在你在该服务器的用户根目录下寻找你的公钥，然 后把它和你发送过来的公钥进行比较。如果两个密钥一致，服务器就用公有密钥加密“质询”（challenge）并把它发送给客户端软件。从而避免被“中间 人”攻击。

　　由于之前所说的原因，会出现一种蛋疼的情况，有些公司还喜欢使用SSH2版本的SSH服务，SSH2和OpenSSH的加密算法是完全不一样的，他们所使用的的密钥对也不兼容，所以会出现下面4种组合

<ol>
<li>OpenSSH客户端对OpenSSH服务器</li>
<li>SSH2客户端对SSH2服务器</li>
<li>OpenSSH客户端对SSH2服务器</li>
<li>SSH2客户端对OpenSSH服务器</li>
</ol>
假设客户端C试图使用用户tester登录服务器S，我们来看看各种组合下如何使用密钥登录:
1.OpenSSH客户端对OpenSSH服务器，这是最简单和最常见的情况
首先在C上操作
{% codeblock lang:bash %}
ssh-keygen -t rsa
{% endcodeblock %}
生成的私钥保存在~/.ssh/id_rsa，(注意私钥一定要是这个名字，除非你更改C的ssh客户端配置)，然后将公钥id_rsa.pub上传到S上去
{% codeblock lang:bash %}
#这里S_IP是服务器的真实IP，并假定用户tester的主目录是/home/tester
scp ~/.ssh/id_rsa.pub tester@S_IP:/home/tester/.ssh/
{% endcodeblock %}
然后在服务器S上做如下操作
{% codeblock lang:bash %}
cd /home/tester/.ssh
cat id_rsa.pub >> authorized_keys
{% endcodeblock %}
退出服务器S，然后从C上重新登录一下
{% codeblock lang:bash %}
ssh tester@S_IP
{% endcodeblock %}
不出意外，你再也不用输入密码了。

2.SSH2客户端对SSH2服务器
这种情况也很简单，因为SSH2版本的ssh服务已经有了个新的工具ssh-keygen2。
首先在C上操作
{% codeblock lang:bash %}
ssh-keygen2 -t rsa
{% endcodeblock %}
注意，这将会在C上当前用户的目录的这个位置~/.ssh2/生成一对密钥id_rsa_2048_a和id_rsa_2048_a.pub
你必须在~/.ssh2/目录下建立一个文件identification，并通过它来指定私钥
{% codeblock lang:bash %}
cd ~/.ssh2/
vi identification
#输入如下内容
IdKey id_rsa_2048_a
#保存修改
{% endcodeblock %}
然后将公钥id_rsa_2048_a.pub传到服务器S上去
{% codeblock lang:bash %}
#这里S_IP是服务器的真实IP，并假定用户tester的主目录是/home/tester
scp ~/.ssh2/id_rsa_2048_a.pub tester@S_IP:/home/tester/.ssh2/
{% endcodeblock %}
然后在服务器S上做如下操作
{% codeblock lang:bash %}
cd /home/tester/.ssh2
vi authorization
#在里面新增一行
Key id_rsa_2048_a.pub
#保存修改
{% endcodeblock %}
退出服务器S，然后从C上重新登录一下
{% codeblock lang:bash %}
ssh tester@S_IP
{% endcodeblock %}
不出意外，这能够工作了。

3.OpenSSH客户端对SSH2服务器
这种情况是最复杂的一种，网络上很多的免密码登录SSH的文章都没有涉及到这种，下面具体介绍一下应该如何配置
首先在C上操作
{% codeblock lang:bash %}
ssh-keygen -t rsa
{% endcodeblock %}
生成的私钥保存在~/.ssh/id_rsa，注意私钥一定要是这个名字，除非你更改C的ssh客户端配置，然后你需要做一件事情，就是将公钥转换成为SSH2所兼容的模式，使用以下的指令
{% codeblock lang:bash %}
cd ~/.ssh/ ssh-keygen -e -f id_rsa.pub > id_rsa_2.pub
{% endcodeblock %}
然后将公钥id_rsa_2.pub上传到S上去
{% codeblock lang:bash %}
#这里S_IP是服务器的真实IP，并假定用户tester的主目录是/home/tester
scp ~/.ssh2/id_rsa_2.pub tester@S_IP:/home/tester/.ssh2/
{% endcodeblock %}
然后在服务器S上做如下操作
{% codeblock lang:bash %}
cd /home/tester/.ssh2
vi authorization
#在里面新增一行
Key id_rsa_2.pub
#保存修改
{% endcodeblock %}
退出服务器S，然后从C上重新登录一下
{% codeblock lang:bash %}
ssh tester@S_IP
{% endcodeblock %}
不出意外，这能够工作了。

4.SSH2客户端对OpenSSH服务器
这种情况是最蛋疼的，应该非常少见吧？这意味你将用一台商业授权的服务器去管理一台开源的服务器？希望你的工作不用这么纠结，虽然这种情况的配置是非常简单的，基本和1一致，因为SSH2原生也支持SSH1，所以就请大家参见1的配置了。

　　如果了解完了上面所说的一切，包括引用链接，你就完全够将SSH应用到工作的各个方面的，下面还会稍微透露一下，平时可能需要了解到的一些要点:
A.SSH2密钥和OpenSSH密钥的相互转换。
{% codeblock lang:bash %}
#OpenSSH转SSH2
ssh-keygen -e -f OpenSSH.pub > SSH2.pub
#SSH2转OpenSSH2
ssh-keygen -i -f SSH2.pub > SSH2.pub
{% endcodeblock %}
B.平时如果我们在Windows环境下，通常会使用SecureCRT，XShell以及Putty等优秀的SSH客户端软件，它们可以让SSH 的工作变得更轻松，但如果在Mac或者Linux环境下，命令行的SSH操作则更自然，那么你知道在命令行下的SSH如何使用代理嘛，当需要的时候？
下面以OpenSSH客户端为例，假设有两个服务器S1和S2，需要通过一个代理服务器P1的80端口才能够连接。
{% codeblock lang:bash %}
vi ~/.ssh/config
#修改如下内容
Host S1_IP S2_IP
    ProxyCommand nc -X connect -x P1:80 %h %p
    ServerAliveInterval 60
{% endcodeblock %}
C.此外，在使用scp都时候还有可能因为ssh和ssh2的问题出现如下错误：
{% codeblock lang:bash %}
"scp - FATAL: Executing ssh1 in compatibility mode failed (check that scp1 is in your PATH)."
{% endcodeblock %}
此种情况发生的场景一般是openssh作为client，要连接一个ssh2的server的时候。
网上对此有如下解释：
{% blockquote %}
Quote 1:

This problem is often quite perplexing, since a ssh -V trace may show that you're using SSH-2 - so what 
is a message about "ssh1 compatibility mode " doing in there?

What's happening is this:
  
  1. On the OpenSSH client, you run say, scp foo server:bar
  2. scp runs ssh in a subprocess to connnect to the server, and run the remote command scp -t bar. This 
     is intend to start an instance of the scp program on the server, and the two scp's will cooperate by
     speaking over the SSH connection, to retrieve the file.
  3. ssh connects to the server (using either protocol 1 or 2, it doesn't matter), and runs the remote scp
     command. However, the "scp" that gets run on the server is the SSH2 scp program (scp2), not the 
     OpenSSH one. The crux of the problem is: besides the name, these two scp's have exactly nothing in 
     common. scp2 cannot speak the file-transfer protocol that OpenSSH scp does. However, scp2 recognizes
     from the "-t" flag what's expected, and tries exec scp1 to service the connection (this is the extent
     of SSH2's SSH-1 compatibility; where OpenSSH has code for both protocols in a single set of programs,
     SSH2 expects to execute programs from a parallel SSH1 installation). It fails (presumably because
     you don't have SSH1 installed), and reports the problem.

The solution is to install either the OpenSSH or SSH1 version of scp on the server under the name "scp1",
somewhere in the sshd2's PATH.


Quote 2:

OpenSSH implements "scp" via RCP over an SSH channel.
ssh.com implement "scp" via FTP over an SSH channel.

OpenSSH's server has both implementations, but it's client only uses
the RCP version.

So if the client is OpenSSH, use "sftp" to get to an ssh.com server.
{% endblockquote %}
如果上述两种解决方案都觉得麻烦的话，可以通过tar来绕过这个问题：
{% codeblock lang:bash %}
scp2() {
    tar cf - -C $(dirname $1) $(basename $1)  | ssh user_name@server_ip -- "tar xmf - -C $2"
}
scp2r () {
    ssh user_name@server_ip -- "tar cf - -C $(dirname $1) $(basename $1)" | tar xmf - -C ${2:-.};
}
{% endcodeblock %}
