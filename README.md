# Local-DNS-Attack

本地DNS攻击一般可以划分为三种方式：

1. 直接修改受害者的host，但要求较高，成功的前提是黑客能够入侵并控制受害者的电脑；

2. 在受害者查询域名IP时，仿冒DNS应答包，并且先于本地DNS服务器发送给受害者，使得受害者接收错误的信息；

3. 直接对本地DNS服务器进行投毒，使得本地DNS服务器中存储的域名-IP地址条目本身就是错误的，如此，该局于网下的所有机器都会成为受害者，在向本地DNS服务器请求域名IP时得到错误的应答；

**注意**！由于本地DNS攻击要求能够偷听到受害者或本地DNS服务器的数据包，所以攻击者的机器必须和它们处于同一局域网中。

## 配置

首先配置好本次实验的实验环境：

软件：Vmware Workstation 12 Player

系统：SEEDUbuntu12.04 * 3 (分别作为本地DNS服务器，客户端和攻击者)

网络：NAT模式

注：这里的**客户端也即受害者**。

NAT（网络地址转换）模式是让虚拟机借助NAT的功能，通过宿主机所在的网络来访问公网。这种模式下宿主机成为双网卡主机，同时参与现有的宿主局域网和虚拟局域网，但由于加设了一个虚拟的NAT服务器，使得虚拟局域网内的虚拟机在对外访问时，使用的是宿主机的IP地址，这样从外部网络来看，只能看到宿主机，完全看不到新建的虚拟局域网。

### 网络配置

因为攻击者要监听到局域网内所有的数据包以捕捉到客户端和本地DNS服务器的包，所以攻击者主机的网卡要设置为混杂模式。实验中用到的网卡是eth0，使用下面语句可以把该网卡设置为混杂模式(重启后需再次配置)：

![混杂模式](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image2.png)

在正式开始实验前，我们需要将三台虚拟机分别配置为本地DNS服务器，客户端和攻击者。先设置IP地址，分别设置为192.168.153.10, 192.168.153.100, 192.168.153.200。设置方式如下(重启后需再次配置)：

![配置IP](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image3.png)

不想重启后再次配置可以直接修改配置文件：

![配置IP](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image4.png)

注意要根据NAT联网方式来设置，

![配置IP](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image5.png)

![配置IP](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image6.png)

宿主机使用的DNS服务器是10.8.8.8和10.8.4.4，VMware采用NAT联网时，用的是VMnet8网卡，我们可以使用的IP地址就从192.168.153.0/24这个子网中抽取。在interfaces文件中加入虚拟机eth0网卡的配置信息：

![配置IP](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image7.png)

在配置好之后重启虚拟机或者重启一下网络即可生效，检查一下此时能不能互相ping通，能不能连外网，都成功就算配置好了。

![配置IP](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image8.png)

直接设置静态IP后就不需要每次重启都重复配置了。

### 配置本地DNS服务器

第一步：安装bind9服务器。

![bind9](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image9.png)

第二步：修改配置文件named.conf.options，instruction中说的是创建，但是这里已经有这个文件了，所以直接打开把语句加在里面就好。

![bind9](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image10.png)

![bind9](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image11.png)

其中dump.db是用来转储DNS服务器的缓存的。

第三步：创建zone，修改同一目录下的named.conf文件，添加以下内容：

![bind9](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image12.png)

这里假设我们拥有example.com这个域名，则我们要对关于这个域名的请求提供回答。所以要在DNS服务器中为其创建zone。

第四步：配置zone文件

![bind9](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image13.png)

在/var/cache/bind路径下创建example.com.db文件，并且写入下面的内容：

![bind9](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image14.png)

这个文件就是我们在named.conf添加的内容中file关键字对应的文件，真正的DNS解析(resolution)是放在这个zone文件中的。

这里的@作为一个特殊符号，表示named.conf中的源，这里也即example.com。而IN表示的则是internet。SOA是Start of Authority的缩写。这个zone文件包含有7条资源记录(RRs, Resource Records)，即SOA RR, NS(Name Server)RR, MX(Mail Exchange)RR 以及四条A(host Address)RR。

然后还要配置DNS逆向lookup文件，在同一目录下增加一个名为192.168.153的文件用于example.com域名，这个文件已经提供好了，复制到路径下就好。

第五步：启动DNS服务器

![bind9](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image15.png)

或者使用sudo service bind9 restart 语句也可以，至此本地DNS服务器配置完毕。

### 配置客户端

在客户端我们需要配置使得本地DNS服务器192.168.153.10成为客户端192.168.153.100的默认DNS服务器，通过修改客户端的DNS配置文件来实现。

第一步：修改etc文件夹下的resolv.conf文件，加入红框内的内容。

![victim](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image16.png)

注意，因为Ubuntu系统中，resolv.conf会被DHCP客户端覆写，这样我们加入的内容就会被消除掉，为了避免这个状况我们要禁止DHCP。

第二步：禁止DHCP

![victim](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image17.png)

点击All Settings，然后点击Network，在Wired选项卡点击Options按钮，然后在IPv4 Settings选项卡中把Method更改为Automatic(DHCP) addresses only，然后把DNS servers更改为前面设置的本地DNS服务器的ip地址。

查看一下此时的DNS服务器，确实就我们刚刚配置的，这样就成功了。

![victim](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image18.png)

事实上，在实际配置我修改了IP地址后无法按实验指导书中的步骤禁止DHCP，所以重启以后就失效了。所以索性直接在interfaces文件中指定DNS服务器就可以了，这和在resolv.conf中指定是一样的，只是interfaces的优先级更高。重启之后测试一下能连外网能互相ping通并且DNS服务器的设置生效就成功了。

### 配置攻击者

攻击者没有什么需要配置的，就按前面配好固定IP地址就可以了。需要用到的Netwat工具和Wireshark都已经装好了。

### 测试配置状况

![test](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image19.png)

在配置好的客户端dig域名可以看到使用的服务器是本地DNS服务器，并且能得到正确的ANSWER。

## Attackers have already compromised the victim’s machine

第一种攻击方式是指黑客已经入侵到客户端，可以修改客户端的文件。这里直接修改客户端的hosts文件，把一个域名重定向到另一个我们指定的IP地址就可以了。使用sudo gedit /etc/hosts打开hosts文件，这里把www.baidu.com解析为1.2.3.4，可以看到ping的时候这个篡改确实生效了。

![attack1](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image20.png)

Dig和nslookup都不会受影响，因为它们都会忽略掉hosts文件，只有ping命令和在浏览器打开网站是受影响的。

![attack1](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image21.png)

## Directly Spoof Response to User

假设攻击者和客户端处在同一个局域网，当客户端的用户使用域名请求页面时，攻击者可以监听到客户端发出的DNS请求，然后抢在本地DNS服务器之前发回一个假的DNS应答，使得客户端收到一个假的IP地址，从而欺骗客户端。

这里我们使用netwag的105工具包来实现，打开netwag后界面如下图所示，双击105号工具，然后到Form选项卡进行设置。

![attack2](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image22.png)

![attack2](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image23.png)

在Form界面只需要设置好要攻击的域名以及想要更换的IP地址就可以了，其他参数可以帮助我们更精确地实现，滑动滑条还会看到一个filter参数，可以使用过滤表达式来限定接收的包，比方说我们只想修改客户端一台机器，这样就可以设置一个过滤表达式ip host 192.168.153.100。设置好后点击Run按钮就会进行Running界面开始监听DNS包，并且在监听到符合条件的包之后按照设定返回DNS应答。

![attack2](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image24.png)

这时再在客户端dig example.com可以发现DNS应答已被篡改为恶意IP。**注意**！因为不能保证每次都是攻击者伪造的DNS应答早于正确DNS应答返回到客户端，所以客户端打开example.com时可能到达正确的页面，也可能变为访问我们伪造的恶意IP，不是百分百能成功的。

## DNS Server Cache Poisoning

前面我们采用抢先本地DNS服务器返回DNS应答的策略，但这个策略不是百分百能成功的。但是我们还可以利用一个漏洞，如果本地DNS服务器没有缓存客户端查询的域名-IP对，那么它就需要向再上级的DNS服务器提交查询，等待对方返回然后缓存到本地，下次客户端再问就可以直接返回了。如果我们在本地DNS服务器向上级查询的时候直接返回一个伪造的DNS应答包给本地DNS服务器，那么它就会把我们伪造包中的恶意IP当作是客户端请求域名对应的IP地址，存入到缓存中。这样以后每次客户端请求都会得到一个错误的IP地址，这就叫做DNS服务器缓存毒化，这种攻击方式要比前面一种好很多。 为了保证效果，我们可以先清空本地DNS服务器的缓存：

![attack3](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image25.png)

这里我们同样使用netwag的105工具包来实现：

![attack3](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image26.png)

![attack3](https://raw.githubusercontent.com/familyld/Local-DNS-Attack/master/graph/image27.png)

因为本地DNS服务器的缓存被毒化了，所以每一次请求bing.com，客户端都会被重定向到1.2.3.4，直到我们关闭netwag，并且在本地DNS服务器的对应条目TTL减少到0，条目被清除，需要再次向上级DNS服务器发出请求。

##小结

[http://www.cnblogs.com/lanxuezaipiao/p/3613497.html](http://www.cnblogs.com/lanxuezaipiao/p/3613497.html) 提供了Ubuntu下修改静态IP地址以及DNS服务器的教程。需要注意配置IP地址时在用VMnet8局域网的可用IP地址，修改DNS服务器后为了避免被覆盖，要关闭DHCP服务。

在后两种攻击方式中中，因为有netwag的帮忙，我们不需要自己构造伪造的DNS应答包，但是需要满足的条件还是要知道的，什么样的DNS回复能够被客户端视为正确的并接收呢？要符合以下9个条件：

1. 源IP地址必须与DNS请求被送往的IP地址相匹配
2. 目的IP地址必须与DNS请求送来的IP地址相匹配；
3. 源端口号（UDP端口）必须与DNS请求被送达（通常是53号端口）的端口号相匹配；
4. 目的端口号必须与DNS请求送来的端口号相匹配；
5. UDP校验和的计算必须正确无误；
6. 传输ID必须与DNS的传输ID相匹配；
7. 答复询问部分的域名必须与请求询问部分的域名匹配；
8. 答复部分的域名必须与DNS请求询问部分的域名匹配；
9. 用户电脑必须在收到合法的DNS应答之前收到攻击者的DNS应答。

[http://www.geek521.com/?p=4952](http://www.geek521.com/?p=4952) 的教程讲得还是比较仔细的，由于条件9不是每次都能满足，所以第二种攻击方式存在失败的几率。

至于第三种攻击方式，它和第二种的不同在于：首先要满足本地DNS服务器没有缓存客户端请求的域名，因为我们的攻击对象从客户端转变为了本地DNS服务器；并且我们可以通过设定ttl来限定伪造的应答包能够在本地DNS服务器的缓存保持多长时间，比方说设定ttl=600，则可以保持600秒，共10分钟；

特别地，我们需要把spoofip字段设置为raw，否则netwag105会尝试嗅探root DNS服务器的MAC地址，因为root DNS服务器往往不在同一个LAN段中，所以要收到ARP回复需要一段时间，如果这段时间中本地DNS服务器得到了正确的IP应答那么攻击就失败了，这是需要注意的地方。设置为raw之后，netwag就不会尝试嗅探MAC地址了。
