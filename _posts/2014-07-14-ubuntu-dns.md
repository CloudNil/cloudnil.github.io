---
layout: post
title:  Ubuntu DNS服务器配置
categories: [Linux]
date: 2014-07-14 10:58:30 +0800
keywords: [Linux,DNS]
---

>基于Ubuntu 14.04 上配置连接DNS服务器，用于内网域名通讯。

### 1 环境说明

* 服务器IP     10.68.19.61
* 操作系统     Ubuntu 13.04
* DNS程序      Bind9
* 测试域名     mycloud.com
* 目标IP       10.68.19.134

### 2 安装配置BIND9

```Bash
apt-get install bind9
```

总共需要编辑2个文件，新增2个文件，如下：
修改/etc/bind/named.conf.options，去掉forwarders的注释，其中的IP为网络营运商提供的DNS服务器，这里我们使用google的DNS。

```Bash
forwarders { 
       8.8.8.8; 
       8.8.4.4; 
}; 
```

修改/etc/bind/named.conf.local，在最后增加增加双向解析代码：

```Bash
zone "mycloud.com" { 
     type master; 
     file "/etc/bind/db.mycloud.com"; 
}; 
   
zone "19.68.10.in-addr.arpa" { 
     type master; 
     file "/etc/bind/db.10.68.19"; 
}; 
```

>注意：其中的19.68.10是目标IP10.68.19.134的前三段，表示一个IP地址段。

新增域名（mycloud.com）解析文件/etc/bind/db.mycloud.com，内容如下：

```Bash
; 
; BIND data file for dev sites 
; 
$TTL    604800 
@       IN      SOA     mycloud.com. root.mycloud.com. ( 
                              1         ; Serial 
                         604800         ; Refresh 
                          86400         ; Retry 
                        2419200         ; Expire 
                         604800 )       ; Negative Cache TTL 
; 
@       IN      NS      mycloud.com. 
@       IN      A       10.68.19.134 
*.mycloud.com.  14400   IN      A       10.68.19.134 
```

新增IP地址反向解析文件/etc/bind/db.10.68.19，内容如下：

```Bash
; 
; BIND reverse data file for dev domains 
; 
$TTL    604800 
@       IN      SOA     dev. root.dev. ( 
                              1         ; Serial 
                         604800         ; Refresh 
                          86400         ; Retry 
                        2419200         ; Expire 
                         604800 )       ; Negative Cache TTL 
; 
@        IN      NS      mycloud.com. 
134      IN      PTR     mycloud.com. 
```

### 3 重启BIND9服务

```Bash
service bind9 restart
```

### 4 修改本机配置

修改每一台需要使用该DNS服务器的dns配置文件

```Bash
sudo vi /etc/resolv.conf
```

修改nameserver为上边配置好的DNS服务器IP

```Bash
nameserver 10.68.19.61
```

此修改在每次重启服务器后都会赔覆盖，可以修改配置文件

```Bash
sudo vi /etc/resolvconf/resolv.conf.d/base 
```

在其中增加一条

```Bash
nameserver 10.68.19.61
```

这样重启服务器后DNS配置依然有效，然后重启networking服务，刷新DNS缓存。

```Bash
service networking restart
```

### 5 测试效果

```Bash
root@controller:/etc/bind# nslookup 
> baidu.com 
Server:         10.68.19.61 
Address:        10.68.19.61#53 
   
Non-authoritative answer: 
Name:   baidu.com 
Address: 220.181.111.86 
Name:   baidu.com 
Address: 123.125.114.144 
Name:   baidu.com 
Address: 220.181.111.85 
> mycloud.com 
Server:         10.68.19.61 
Address:        10.68.19.61#53 
   
Name:   mycloud.com 
Address: 10.68.19.134 
> uaa.mycloud.com 
Server:         10.68.19.61 
Address:        10.68.19.61#53 
   
Name:   uaa.mycloud.com 
Address: 10.68.19.134
```

解析情况为，域名：baidu.com，在本地DNS中没有找到匹配，通过DNS：8.8.8.8解析，mycloud.com在本地DNS中有匹配，解析到10.68.19.134.

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2014/07/14/ubuntu-dns/。