---
layout: post
title:  Mesos+Marathon+docker集群部署
categories: [docker,Cloud]
date: 2015-12-14 10:58:30 +0800
keywords: [docker,云计算,mesos]
---

>基于Centos7.2 Mesos+Marathon集群部署，服务发现使用bamboo组件。

### 1 整体架构

|     节点名称    | 节点类型 |      IP      |            组件            |
| --------------- | -------- | ------------ | -------------------------- |
| master101       | master   | 192.168.2.71 | mesos、marathon、zookpeer  |
| master102       | master   | 192.168.2.72 | mesos、marathon、zookpeer  |
| master103       | master   | 192.168.2.73 | mesos、marathon、zookpeer  |
| slave101        | slave    | 192.168.2.61 | mesos、docker              |
| slave102        | slave    | 192.168.2.62 | mesos、docker              |
| slave103        | slave    | 192.168.2.63 | mesos、docker              |
| bamboo101       | 负载均衡 | 192.168.2.91 | haproxy、bamboo、keeplived |
| bamboo102       | 负载均衡 | 192.168.2.92 | haproxy、bamboo、keeplived |
| bamboo103       | 负载均衡 | 192.168.2.93 | haproxy、bamboo、keeplived |

>说明：集群模式部署，master节点应该是奇数，最少为3个节点，便于leader选举

### 2 环境准备

>操作系统：Centos7.2 Minimal  
>Mesos版本：0.24.1  
>Marathon版本：0.11.0  
>Docker版本：1.7.1

- 关闭selinux（重启）

```Bash
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

- 关闭防火墙

```Bash
systemctl disable firewalld.service
```

- 清空iptables

```Bash
iptables -F
```

- 升级centos包

```Bash
yum update
```

- 安装mesosphere仓库

```Bash
rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
yum clean all
yum makecache
```

>说明：以上部署所有节点执行

### 3 Master节点安装

#### 3.1 组件安装  
```Bash
yum -y install mesos marathon mesosphere-zookeeper
```

#### 3.2 配置zookeeper

```Bash
#master101
echo 1 > /var/lib/zookeeper/myid
#master102
echo 2 > /var/lib/zookeeper/myid
#master103
echo 3 > /var/lib/zookeeper/myid
```

编辑文件/etc/zookeeper/conf/zoo.cfg，追加以下内容

```Bash
server.1=192.168.2.71:2888:3888
server.2=192.168.2.72:2888:3888
server.3=192.168.2.73:2888:3888
```

重启zookeeper服务

```Bash
systemctl start zookeeper
```

#### 3.3 配置mesos

配置mesos使用zookeeper

```Bash
echo zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/mesos > /etc/mesos/zk
echo 2 > /etc/mesos-master/quorum
```

配置mesos-master组件的hostname和ip参数

```Bash
#master101
echo 192.168.2.71 > /etc/mesos-master/hostname
echo 192.168.2.71 > /etc/mesos-master/ip
#master102
echo 192.168.2.72 > /etc/mesos-master/hostname
echo 192.168.2.72 > /etc/mesos-master/ip
#master103
echo 192.168.2.73 > /etc/mesos-master/hostname
echo 192.168.2.73 > /etc/mesos-master/ip
```

> 说明：hostname可以不配置，默认使用机器名

停用masrer节点上的mesos-slave服务

```Bash
systemctl stop mesos-slave.service && systemctl disable mesos-slave.service
```

重启mesos-master服务

```Bash
systemctl restart mesos-master.service
```

#### 3.4 配置marathon

```Bash
mkdir -p /etc/marathon/conf
```

直接复制mesos的hostname文件

```Bash
cp /etc/mesos-master/hostname /etc/marathon/conf
echo http_callback > /etc/marathon/conf/event_subscriber
```

#### 3.5 配置marathon使用zookeeper（可选）

默认情况下marathon会自动获取本机的mesos的zk配置，并且会根据zookeeper的配置。为自己添加marathon的配置同步，手动添加以下配置文件是为了便于管理，同样，其他的marathon参数，都可以以参数名称命名一个文件，存放在/etc/marathon/conf目录下，然后在其中设置参数值的形式为marathon作进一步配置
提示：在本例中，默认的marathon启动命令为：

```Bash
marathon: run_jar --hostname 192.168.2.71 --zk zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/marathon --master zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/mesos
```

第一步：连接mesos的zookeeper

```Bash
cp /etc/mesos/zk /etc/marathon/conf/master
```

第二步：配置marathon使用的zookeeper

```Bash
echo zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/marathon /etc/marathon/conf/zk
```  
#### 启动服务及检测

重启marathon服务

```Bash
systemctl restart marathon
```

检查端口监听情况

>说明：如果没有lsof命令可以下载安装`yum install -y lsof`

zookeeper端口：lsof -i :2181 -n

```Bash
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java      2893 root   23u  IPv6  23787      0t0  TCP *:eforward (LISTEN)
java      2893 root   27u  IPv6  25757      0t0  TCP 192.168.2.71:eforward->192.168.2.71:35808 (ESTABLISHED)
java      2893 root   30u  IPv6  24204      0t0  TCP 192.168.2.71:eforward->192.168.2.71:35812 (ESTABLISHED)
java      2893 root   31u  IPv6  25810      0t0  TCP 192.168.2.71:eforward->192.168.2.71:35852 (ESTABLISHED)
java      2893 root   32u  IPv6  25803      0t0  TCP 192.168.2.71:eforward->192.168.2.72:57387 (ESTABLISHED)
java      2957 root   15u  IPv4  25761      0t0  TCP 192.168.2.71:35812->192.168.2.71:eforward (ESTABLISHED)
java      2957 root   31u  IPv6  25157      0t0  TCP 192.168.2.71:35808->192.168.2.71:eforward (ESTABLISHED)
java      2957 root   37u  IPv6  25758      0t0  TCP 192.168.2.71:54372->192.168.2.72:eforward (ESTABLISHED)
mesos-mas 3342 root   17u  IPv4  25809      0t0  TCP 192.168.2.71:35852->192.168.2.71:eforward (ESTABLISHED)
mesos-mas 3342 root   18u  IPv4  26920      0t0  TCP 192.168.2.71:54419->192.168.2.72:eforward (ESTABLISHED)
mesos-mas 3342 root   22u  IPv4  25817      0t0  TCP 192.168.2.71:54420->192.168.2.72:eforward (ESTABLISHED)
mesos-mas 3342 root   23u  IPv4  25816      0t0  TCP 192.168.2.71:54418->192.168.2.72:eforward (ESTABLISHED)
```

mesos端口：lsof -i :5050 -n

```Bash
java      2957 root   27u  IPv4  25326      0t0  TCP 192.168.2.71:52901->192.168.2.72:mmcc (ESTABLISHED)
mesos-mas 3342 root    5u  IPv4  24286      0t0  TCP 192.168.2.71:mmcc (LISTEN)
mesos-mas 3342 root   24u  IPv4  26921      0t0  TCP 192.168.2.71:52898->192.168.2.72:mmcc (ESTABLISHED)
mesos-mas 3342 root   25u  IPv4  26967      0t0  TCP 192.168.2.71:60935->192.168.2.73:mmcc (ESTABLISHED)
mesos-mas 3342 root   26u  IPv4  25818      0t0  TCP 192.168.2.71:mmcc->192.168.2.72:45845 (ESTABLISHED)
mesos-mas 3342 root   27u  IPv4  25823      0t0  TCP 192.168.2.71:mmcc->192.168.2.73:51978 (ESTABLISHED)
```

其他端口

```Bash
mesos通信端口：lsof -i :2888 -n
mesos选举端口：lsof -i :3888 -n
marathon端口：lsof -i :8080 -n
```

服务重启命令

```Bash
systemctl restart zookeeper
systemctl restart mesos-master
systemctl restart marathon
```

配置开机启动

```Bash
chkconfig zookeeper on
chkconfig mesos-master on
chkconfig marathon on
```

### 4 Salve节点安装

#### 4.1 组件安装

```Bash
yum -y install mesos docker
```

#### 4.2 配置mesos，与master一致

```Bash
echo zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/mesos > /etc/mesos/zk
```

#### 4.3 配置mesos-slave

```Bash
#slave101
echo 192.168.2.61 > /etc/mesos-slave/hostname
echo 192.168.2.61 > /etc/mesos-slave/ip
```

```Bash
#slave102
echo 192.168.2.62 > /etc/mesos-slave/hostname
echo 192.168.2.62 > /etc/mesos-slave/ip
```

```Bash
#slave103
echo 192.168.2.63 > /etc/mesos-slave/hostname
echo 192.168.2.63 > /etc/mesos-slave/ip
```

>hostname可以不配置，默认使用机器名

#### 4.4 配置mesos-slave使用docker容器

```Bash
echo 'docker,mesos' > /etc/mesos-slave/containerizers
echo '5mins' > /etc/mesos-slave/executor_registration_timeout
```

如果使用本地docker仓库，需要配置docker

```Bash
sed -i "s/^OPTIONS='--selinux-enabled'/OPTIONS='--selinux-enabled --insecure-registry 192.168.2.98:5000'/g" /etc/sysconfig/docker
```

>说明：`192.168.2.98:5000`是本环境中部署的docker registry仓库地址

#### 4.5 启动服务

停用slave节点上的mesos-master服务

```Bash
systemctl stop mesos-master.service && systemctl disable mesos-master.service
```

服务重启命令

```Bash
systemctl restart docker
systemctl restart mesos-slave
```

配置开机启动

```Bash
chkconfig docker on
chkconfig mesos-slave on
```

### 5 Mesos简单使用

#### 5.1 Mesos控制台

Mesos的控制台地址：`http://192.168.2.71:5050`
Mesos的控制台上可以查看的当前的资源实用情况、Slave节点状态、当前运行的Task、完成的Task、可以切换到Framework（如Marathon）或者Slave。

#### 5.2 Marathon控制台

Marathon控制台地址：`http://192.168.2.71:8080`
Marathon控制台上可以查看当前应用的运行状态，可以发布新应用、调整当前应用的实例数等。  
发布应用：

```Bash
ID：hello
CPUs：0.1
Memory：16
Disk Space：0
Instances：1
Command：echo hello world!;sleep 10;
```

上述例子是使用Mesos默认容器进行创建，并执行任务；如果需要使用docker容器，可以在`Docker container settings`中填写具体参数。
应用创建后，会输出`hello world!`，并且每10秒钟重新deploy一次。

>说明：Mesos+Marathon框架发布的Task会在执行完毕后进行销毁，并重新发布，若是需要发布web app一类的长应用，必须保证运行程序独占console，如Tomcat，可以执行`catalina.sh run`进行启动并保持长运行状态，后台运行的程序Marathon无法保持Task长运行。

可以通过Marathon的RestAPI发布应用

```Bash
curl -X POST http://192.168.2.71:8080/v2/apps -d@inky.json -H "Content-type: application/json"
```

inky.json文件示例

```json
{
    "id": "inky4",
    "container": {
        "docker": {
            "image": "192.168.2.98:5000/tomcat-jdk1.7",
            "network":"BRIDGE",
            "portMappings":[{"containerPort":8080,"hostPort":0,"protocol": "tcp"}]
        },
        "type": "DOCKER",
        "volumes": []
    },
    "args": [],
    "cpus": 1,
    "mem": 512,
    "instances": 1
}
```

### 6 Docker私有仓库

#### 6.1 仓库搭建

方式一：通过docker image构建

`docker run -d -p 5000:5000 -v /opt/data/registry:/tmp/registry:rw registry`

>说明：`/tmp/registry`:容器中镜像的存储位置
`/opt/data/registry`：挂载`/tmp/registry`到本地宿主机上的目录

方式二：通过本地安装docker-registry构建

```Bash
yum install -y python-devel libevent-devel python-pip gcc xz-devel
python-pip install docker-registry
```

存储路径修改：

`cp config/config_sample.yml config/config.yml`

修改其中的：`storage_path`参数，启动docker-registry的web服务

`gunicorn -c contrib/gunicorn.py docker_registry.wsgi:application`

#### 6.2 上传、下载镜像

1. 从公有创库中获取基础镜像（docker pull）
2. 对基础镜像做出修改（docker commit）
3. 标记本地镜像（docker tag）
4. 推送镜像到私有仓库（docker push）
5. 获取私有仓库镜像（docker pull）

### 7 负载均衡与服务发现

#### 7.1 Haproxy组件

```Bash
yum install -y haproxy
```

配置忽略VIP及开启转发

```Bash
echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
echo net.ipv4.ip_nonlocal_bind=1 >> /etc/sysctl.conf
```

重启系统或者执行以下命令

```Bash
sysctl -e net.ipv4.ip_forward=1
sysctl -e net.ipv4.ip_nonlocal_bind=1
```

重启haproxy服务

```Bash
chkconfig haproxy on
systemctl restart haproxy
```

#### 7.2 Bamboo组件

通过脚本自动安装

```Bash
curl https://raw.githubusercontent.com/VFT/bamboo/master/install.sh | sh
mkdir /var/bamboo
cp /opt/bamboo/config/haproxy_template.cfg /var/bamboo/
cp /opt/bamboo/config/production.example.json /var/bamboo/production.json
```

启动Bamboo服务

```Bash
nohup /opt/bamboo/bamboo -haproxy_check -config="/var/bamboo/production.json" &
```

停止Bamboo服务

```Bash
kill $(lsof -i:8000 |awk '{print $2}' | tail -n 2)
```

>说明：可以用命令`netstat -ntpl`或者`lsof -i :5050 -n`查看进程

#### 7.3 Bamboo组件（源码编译安装）

直接获取rpm包

```Bash
wget https://github.com/VFT/mesos-docker/blob/master/package/bamboo-0.2.16_1-1.x86_64.rpm
```

或者自主编译rpm二进制包

```Bash
# build dependencies
sudo yum install -y golang rpm-build rubygems ruby-devel
sudo gem install  fpm  --no-ri --no-rdoc
# setup a go build tree
sudo yum install -y git mercurial
export GOPATH=~/gopath
mkdir $GOPATH
go get github.com/tools/godep
go install github.com/tools/godep
# build the binary
# get newest source code
go get github.com/QubitProducts/bamboo
cd ${GOPATH}/src/github.com/QubitProducts/bamboo
# or get a specil version source code，such as：
# wget https://github.com/QubitProducts/bamboo/archive/v0.2.15.tar.gz
# tar xzvf v0.2.15.tar.gz
# cd bamboo-0.2.15/
go build

# edit builder/build.after-install
sed -i '10,15s/configure)/*)/g' builder/build.after-install

# edit builder/build.sh
sed -i 's/version=${_BAMBOO_VERSION:-"1.0.0"}/version=${_BAMBOO_VERSION:-"0.2.15"}/g' builder/build.sh
sed -i 's/arch="all"/arch="x86_64"/g' builder/build.sh
sed -i 's/pkgtype=${_PKGTYPE:-"deb"}/pkgtype=${_PKGTYPE:-"rpm"}/g' builder/build.sh

#运行命令生成rpm包，输出到output目录
./builder/build.sh

#安装rpm包
rpm -ivh bamboo-0.2.15_1-1.x86_64.rpm
```

安装start-stop-daemon：
方式一直接获取

```Bash
wget https://github.com/VFT/mesos-docker/blob/master/script/init.d-bamboo-server
```

方式二自主编译

```Bash
yum install -y gcc wget
wget http://developer.axis.com/download/distribution/apps-sys-utils-start-stop-daemon-IR1_9_18-2.tar.gz
tar zxf apps-sys-utils-start-stop-daemon-IR1_9_18-2.tar.gz
cd apps/sys-utils/start-stop-daemon-IR1_9_18-2/
gcc start-stop-daemon.c -o start-stop-daemon
cp start-stop-daemon /usr/bin
```

配置bamboo-server启动（脚本位于源码包中）

```Bash
wget https://github.com/QubitProducts/bamboo/archive/v0.2.14.tar.gz
tar xzvf v0.2.14.tar.gz
cd bamboo-0.2.14/
cp builder/init.d-bamboo-server /etc/init.d/bamboo-server
chown root:root /etc/init.d/bamboo-server
chmod 755 /etc/init.d/bamboo-server
chkconfig bamboo-server on
```

修改/var/bamboo/production.json

```json
{
  "Marathon": {
    "Endpoint": "http://192.168.2.71:8080,http://192.168.2.72:8080,http://192.168.2.73:8080",
    "UseEventStream": true
  },

  "Bamboo": {
    "Endpoint": "http://192.168.2.93:8000",
    "Zookeeper": {
      "Host": "192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181",
      "Path": "/marathon-haproxy/state",
      "ReportingDelay": 5
    }
  },

  "HAProxy": {
    "TemplatePath": "/var/bamboo/haproxy_template.cfg",
    "OutputPath": "/etc/haproxy/haproxy.cfg",
    "ReloadCommand": "haproxy -f /etc/haproxy/haproxy.cfg -p /var/run/haproxy.pid -D -sf $(cat /var/run/haproxy.pid)",
    "ReloadValidationCommand": "haproxy -c -f {{.}}"
  },

  "StatsD": {
    "Enabled": false,
    "Host": "localhost:8125",
    "Prefix": "bamboo-server.production."
  }
}
```

>说明：Marathon.Endpoint：Marathon服务的访问地址，Bamboo.Host：Bamboo服务控制台地址，Bamboo.Zookeeper.Host：Zookeeper服务访问地址

修改/var/bamboo/haproxy_template.cfg

```Bash
stats socket /run/haproxy/admin.sock mode 660 level admin  >>   stats socket /var/lib/haproxy/stats
```

启动Bamboo服务

`systemctl start bamboo-server`

停止Bamboo服务

`systemctl stop bamboo-server`

#### 7.4 Bamboo简单使用

按上述配置Bamboo安装后以后，Bamboo监听`8000`端口，可以通过`http://<IP>:8000`来访问bamboo的控制台

![bamboo console](https://raw.githubusercontent.com/VFT/imageStore/master/bamboo01.png)

添加转发规则

![bamboo edit](https://raw.githubusercontent.com/VFT/imageStore/master/bamboo02.png)

现在，可以通过`http://<IP>`，默认端口：`80`来访问`inky1`这个app。

### 8 疑难杂症

#### 8.1 内存不足

- 表现：无法发布app应用，marathon日志中关键提示`mem NOT SATISFIED`
- 分析：free命令查看宿主机内存使用率
```Bash
[root@slave101 ~]# free -h
              total        used        free      shared  buff/cache   available
Mem:           1.8G        339M        261M         24M        1.2G        1.2G
Swap:          2.0G         76K        2.0G
```

可以看出，空闲内存不足，基本都是被cache占用，而marathon识别内存时是从free中识别

- 解决：手动释放内存
```Bash
sync
echo 3 > /proc/sys/vm/drop_caches
sysctl -p
```

查看内存情况：

```Bash
[root@salve102 ~]# free -h
              total        used        free      shared  buff/cache   available
Mem:           1.8G        441M        1.2G         24M        167M        1.2G
Swap:          2.0G        108K        2.0G
```