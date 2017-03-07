---
layout: post
title:  kubeadm快速部署kubernetes1.5.2
categories: [docker,Cloud]
date: 2017-01-17 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>Kubernetes 1.5.2发布，调整部署文档。

### 1 环境准备

准备了三台机器作安装测试工作，机器信息如下: 

|      IP      |   Name   |    Role    |     OS    |
|--------------|----------|------------|-----------|
| 172.16.1.101 | Master01 | Controller | Centos7.2 |
| 172.16.1.106 | Minion01 | Compute    | Centos7.2 |
| 172.16.1.107 | Minino02 | Compute    | Centos7.2 |

### 2 安装docker

```bash
tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

yum update -y && yum upgrade -y
yum install docker-engine -y
systemctl start docker
systemctl enable docker.service
```

### 3 安装k8s工具包

三种方式：官方源安装、非官方源安装和release工程编译，yum方式因为不能直接使用google提供的源，非官方源中提供的版本比较老（mritd提供的源很不错，版本很新），如果要使用新版本，可以尝试release工程编译的方式。

**博主提供**

一些比较懒得同学:-D，可以直接从博主提供的位置下载RPM工具包安装，[下载地址](https://github.com/CloudNil/kubernetes-library/tree/master/rpm_x86_64/For_kubelet_1.5.2)。

```bash
yum install -y socat

rpm -ivh kubeadm-1.6.0-0.alpha.0.2074.a092d8e0f95f52.x86_64.rpm  kubectl-1.5.1-0.x86_64.rpm  kubelet-1.5.1-0.x86_64.rpm  kubernetes-cni-0.3.0.1-0.07a8a2.x86_64.rpm

systemctl enable kubelet.service
```

**官方源安装**

跨越GFW方式不细说，你懂的。

建议使用`yumdownloader`下载rpm包，不然那下载速度，会让各位对玩k8s失去兴趣的。

```bash
yum install -y yum-utils

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

yumdownloader kubelet kubeadm kubectl kubernetes-cni
rpm -ivh *.rpm
systemctl enable kubelet.service && systemctl start kubelet
```

**非官方源安装**

```bash
#感谢mritd维护了一个yum源
tee /etc/yum.repos.d/mritd.repo << EOF
[mritdrepo]
name=Mritd Repository
baseurl=https://rpm.mritd.me/centos/7/x86_64
enabled=1
gpgcheck=1
gpgkey=https://cdn.mritd.me/keys/rpm.public.key
EOF
yum makecache
yum install -y kubelet kubectl kubernetes-cni kubeadm
systemctl enable kubelet && systemctl start kubelet
```

**relese编译**

```bash
git clone https://github.com/kubernetes/release.git
cd release/rpm
./docker-build.sh
```

编译完成后生成rpm包到：`/output/x86_64`，进入到该目录后安装rpm包，注意选择amd64的包（相信大多数同学都是64bit环境，如果是32bit或者arm架构请自行选择安装）。

### 4 下载docker镜像

kubeadm方式安装kubernetes集群需要的镜像在docker官方镜像中并未提供，只能去google的官方镜像库：`gcr.io` 中下载，GFW咋办？翻墙!也可以使用docker hub做跳板自己构建，这里针对k8s-1.5.2我已经做好镜像，各位可以直接下载，dashboard的版本并未紧跟kubelet主线版本，用哪个版本都可以，本文使用kubernetes-dashboard-amd64:v1.5.0。

kubernetes-1.5.2所需要的镜像：

- etcd-amd64:2.2.5
- kubedns-amd64:1.9
- kube-dnsmasq-amd64:1.4 
- dnsmasq-metrics-amd64:1.0
- exechealthz-amd64:1.2 
- pause-amd64:3.0
- kube-discovery-amd64:1.0
- kube-proxy-amd64:v1.5.2 
- kube-scheduler-amd64:v1.5.2
- kube-controller-manager-amd64:v1.5.2 
- kube-apiserver-amd64:v1.5.2 
- kubernetes-dashboard-amd64:v1.5.0

偷下懒吧，直接执行以下脚本：

```bash
#!/bin/bash
images=(kube-proxy-amd64:v1.5.2 kube-discovery-amd64:1.0 kubedns-amd64:1.9 kube-scheduler-amd64:v1.5.2 kube-controller-manager-amd64:v1.5.2 kube-apiserver-amd64:v1.5.2 etcd-amd64:2.2.5 kube-dnsmasq-amd64:1.4 dnsmasq-metrics-amd64:1.0 exechealthz-amd64:1.2 pause-amd64:3.0 kubernetes-dashboard-amd64:v1.5.0 nginx-ingress-controller:0.8.3)
for imageName in ${images[@]} ; do
  docker pull cloudnil/$imageName
  docker tag cloudnil/$imageName gcr.io/google_containers/$imageName
  docker rmi cloudnil/$imageName
done
```

### 5 安装master节点

由于kubeadm和kubelet安装过程中会生成`/etc/kubernetes`目录，而`kubeadm init`会先检测该目录是否存在，所以我们先使用kubeadm初始化环境。

```bash
kubeadm reset && systemctl start kubelet
kubeadm init --api-advertise-addresses=172.16.1.101 --use-kubernetes-version v1.5.2
#如果使用外部etcd集群:
kubeadm init --api-advertise-addresses=172.16.1.101 --use-kubernetes-version v1.5.2 --external-etcd-endpoints http://172.16.1.107:2379,http://172.16.1.107:4001
```

>说明：如果打算使用flannel网络，请加上：`--pod-network-cidr=10.244.0.0/16`。如果有多网卡的，请根据实际情况配置`--api-advertise-addresses=<ip-address>`，单网卡情况可以省略。

如果出现`ebtables not found in system path`的错误，要先安装`ebtables`包，我安装的过程中未提示，该包系统已经自带了。
```bash
yum install -y ebtables
```

安装过程大概2-3分钟，输出结果如下：

```bash
[kubeadm] WARNING: kubeadm is in alpha, please do not use it for production clusters.
[preflight] Running pre-flight checks
[init] Using Kubernetes version: v1.5.2
[tokens] Generated token: "064158.548b9ddb1d3fad3e"
[certificates] Generated Certificate Authority key and certificate.
[certificates] Generated API Server key and certificate
[certificates] Generated Service Account signing keys
[certificates] Created keys and certificates in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 61.317580 seconds
[apiclient] Waiting for at least one node to register and become ready
[apiclient] First node is ready after 6.556101 seconds
[apiclient] Creating a test deployment
[apiclient] Test deployment succeeded
[token-discovery] Created the kube-discovery deployment, waiting for it to become ready
[token-discovery] kube-discovery is ready after 6.020980 seconds
[addons] Created essential addon: kube-proxy
[addons] Created essential addon: kube-dns

Your Kubernetes master has initialized successfully!

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node:

kubeadm join --token=de3d61.504a049ec342e135 172.16.1.101
```

### 6 安装minion节点

Master节点安装好了Minoin节点就简单了。

```bash
kubeadm reset && systemctl start kubelet
kubeadm join --token=de3d61.504a049ec342e135 172.16.1.101
```

输出结果如下：

```bash
[kubeadm] WARNING: kubeadm is in alpha, please do not use it for production clusters.
[preflight] Running pre-flight checks
[preflight] Starting the kubelet service
[tokens] Validating provided token
[discovery] Created cluster info discovery client, requesting info from "http://172.16.1.101:9898/cluster-info/v1/?token-id=f11877"
[discovery] Cluster info object received, verifying signature using given token
[discovery] Cluster info signature and contents are valid, will use API endpoints [https://172.16.1.101:6443]
[bootstrap] Trying to connect to endpoint https://172.16.1.101:6443
[bootstrap] Detected server version: v1.5.2
[bootstrap] Successfully established connection with endpoint "https://172.16.1.101:6443"
[csr] Created API client to obtain unique certificate for this node, generating keys and certificate signing request
[csr] Received signed certificate from the API server:
Issuer: CN=kubernetes | Subject: CN=system:node:yournode | CA: false
Not before: 2016-12-15 19:44:00 +0000 UTC Not After: 2017-12-15 19:44:00 +0000 UTC
[csr] Generating kubelet configuration
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"

Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```

安装完成后可以查看下状态：
```bash
[root@master ~]# kubectl get nodes
NAME       STATUS         AGE
master     Ready,master   6m
minion01   Ready          2m
minion02   Ready          2m
```

### 7 安装Calico网络

网络组件选择很多，可以根据自己的需要选择calico、weave、flannel，calico性能最好，weave和flannel差不多。[Addons](http://kubernetes.io/docs/admin/addons/)中有配置好的yaml，部署环境使用的阿里云的VPC，官方提供的flannel.yaml创建的flannel网络有问题，所以本文中尝试calico网络，。

```bash
kubectl apply -f http://docs.projectcalico.org/v2.0/getting-started/kubernetes/installation/hosted/kubeadm/calico.yaml
```

如果使用了外部etcd，去掉其中以下内容，并修改`etcd_endpoints: [ETCD_ENDPOINTS]`：

```yaml
---

# This manifest installs the Calico etcd on the kubeadm master.  This uses a DaemonSet
# to force it to run on the master even when the master isn't schedulable, and uses
# nodeSelector to ensure it only runs on the master.
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: calico-etcd
  namespace: kube-system
  labels:
    k8s-app: calico-etcd
spec:
  template:
    metadata:
      labels:
        k8s-app: calico-etcd
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        scheduler.alpha.kubernetes.io/tolerations: |
          [{"key": "dedicated", "value": "master", "effect": "NoSchedule" },
           {"key":"CriticalAddonsOnly", "operator":"Exists"}]
    spec:
      # Only run this pod on the master.
      nodeSelector:
        kubeadm.alpha.kubernetes.io/role: master
      hostNetwork: true
      containers:
        - name: calico-etcd
          image: gcr.io/google_containers/etcd:2.2.1
          env:
            - name: CALICO_ETCD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          command: ["/bin/sh","-c"]
          args: ["/usr/local/bin/etcd --name=calico --data-dir=/var/etcd/calico-data --advertise-client-urls=http://$CALICO_ETCD_IP:6666 --listen-client-urls=http://0.0.0.0:6666 --listen-peer-urls=http://0.0.0.0:6667"]
          volumeMounts:
            - name: var-etcd
              mountPath: /var/etcd
      volumes:
        - name: var-etcd
          hostPath:
            path: /var/etcd

---

# This manfiest installs the Service which gets traffic to the Calico
# etcd.
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: calico-etcd
  name: calico-etcd
  namespace: kube-system
spec:
  # Select the calico-etcd pod running on the master.
  selector:
    k8s-app: calico-etcd
  # This ClusterIP needs to be known in advance, since we cannot rely
  # on DNS to get access to etcd.
  clusterIP: 10.96.232.136
  ports:
    - port: 6666
```

检查各节点组件运行状态：

```bash
[root@master work]# kubectl get po -n=kube-system -o wide
NAME                                       READY     STATUS    RESTARTS   AGE       IP                NODE
calico-node-0jkjn                          2/2       Running   0          25m       172.16.1.101      master
calico-node-w1kmx                          2/2       Running   2          25m       172.16.1.106      minion01
calico-node-xqch6                          2/2       Running   0          25m       172.16.1.107      minion02
calico-policy-controller-807063459-d7z47   1/1       Running   0          11m       172.16.1.107      minion02
dummy-2088944543-qw3vr                     1/1       Running   0          29m       172.16.1.101      master
kube-apiserver-master                      1/1       Running   0          28m       172.16.1.101      master
kube-controller-manager-master             1/1       Running   0          29m       172.16.1.101      master
kube-discovery-1769846148-lzlff            1/1       Running   0          29m       172.16.1.101      master
kube-dns-2924299975-jfvrd                  4/4       Running   0          29m       192.168.228.193   master
kube-proxy-6bk7n                           1/1       Running   0          28m       172.16.1.107      minion02
kube-proxy-6pgqz                           1/1       Running   1          29m       172.16.1.106      minion01
kube-proxy-7ms6m                           1/1       Running   0          29m       172.16.1.101      master
kube-scheduler-master                      1/1       Running   0          28m       172.16.1.101      master
```

>说明：kube-dns需要等calico配置完成后才是running状态。

### 8 部署Dashboard

下载kubernetes-dashboard.yaml

```bash
curl -O https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

修改配置内容，#------内是修改的内容，调整目的：部署kubernetes-dashboard到default-namespaces，不暴露端口到HostNode，调整版本为1.5.0，imagePullPolicy调整为IfNotPresent。

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  labels:
    app: kubernetes-dashboard
  name: kubernetes-dashboard
#----------
#  namespace: kube-system
#----------
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubernetes-dashboard
  template:
    metadata:
      labels:
        app: kubernetes-dashboard
      annotations:
        scheduler.alpha.kubernetes.io/tolerations: |
          [
            {
              "key": "dedicated",
              "operator": "Equal",
              "value": "master",
              "effect": "NoSchedule"
            }
          ]
    spec:
      containers:
      - name: kubernetes-dashboard
        #----------
        image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.5.0
        imagePullPolicy: IfNotPresent
        #----------
        ports:
        - containerPort: 9090
          protocol: TCP
        args:
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
        livenessProbe:
          httpGet:
            path: /
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: kubernetes-dashboard
  name: kubernetes-dashboard
#----------
#  namespace: kube-system
#----------
spec:
#----------
#  type: NodePort
#----------
  ports:
  - port: 80
    targetPort: 9090
  selector:
    app: kubernetes-dashboard
```

### 9 Dashboard服务暴露到公网

kubernetes中的Service暴露到外部有三种方式，分别是：

- LoadBlancer Service
- NodePort Service
- Ingress

LoadBlancer Service是kubernetes深度结合云平台的一个组件；当使用LoadBlancer Service暴露服务时，实际上是通过向底层云平台申请创建一个负载均衡器来向外暴露服务；目前LoadBlancer Service支持的云平台已经相对完善，比如国外的GCE、DigitalOcean，国内的 阿里云，私有云 Openstack 等等，由于LoadBlancer Service深度结合了云平台，所以只能在一些云平台上来使用。

NodePort Service顾名思义，实质上就是通过在集群的每个node上暴露一个端口，然后将这个端口映射到某个具体的service来实现的，虽然每个node的端口有很多(0~65535)，但是由于安全性和易用性(服务多了就乱了，还有端口冲突问题)实际使用可能并不多。

Ingress可以实现使用nginx等开源的反向代理负载均衡器实现对外暴露服务，可以理解Ingress就是用于配置域名转发的一个东西，在nginx中就类似upstream，它与ingress-controller结合使用，通过ingress-controller监控到pod及service的变化，动态地将ingress中的转发信息写到诸如nginx、apache、haproxy等组件中实现方向代理和负载均衡。

#### 9.1 部署Nginx-ingress-controller

`Nginx-ingress-controller`是kubernetes官方提供的集成了Ingress-controller和Nginx的一个docker镜像。

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-ingress-controller
  labels:
    k8s-app: nginx-ingress-lb
spec:
  replicas: 1
  selector:
    k8s-app: nginx-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: nginx-ingress-lb
        name: nginx-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      #本环境中的minion02节点有外网IP，并且有label定义：External-IP=true
      nodeSelector:
        External-IP: true
      containers:
      - image: gcr.io/google_containers/nginx-ingress-controller:0.8.3
        name: nginx-ingress-lb
        imagePullPolicy: IfNotPresent
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          timeoutSeconds: 1
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        args:
        - /nginx-ingress-controller
        - --default-backend-service=$(POD_NAMESPACE)/kubernetes-dashboard
```

#### 9.2 部署Ingress

```bash
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: k8s-dashboard
spec:
  rules:
  - host: dashboard.cloudnil.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 80

```

部署完Ingress后，解析域名`dashboard.cloudnil.com`到minion02的外网IP，就可以使用`dashboard.cloudnil.com`访问dashboard。

### 10 注意事项

kubeadm目前还在开发测试阶段，不建议在生产环境中使用kubeadm部署kubernetes环境。此外，使用kubeadm是需要注意以下几点：

#### 10.1 单点故障

当前版本的kubeadm暂且不能部署真正高可用的kubernetes环境，只具有单点的master环境，如采用内置etcd，那etcd也是单节点，若master节点故障，可能存在数据丢失的情况，所以建议采用外部的etcd集群，这样即使master节点故障，那只要重启即可，数据不会丢失，高可用的部署功能据说正在开发中，很快就可以发布使用。

#### 10.2 暴露主机端口

POD实例配置中的HostPort和HostIP参数无法用于使用了CNI网络插件的kubernetes集群环境，如果需要暴露容器到主机端口，可以使用NodePort或者HostNetwork。

#### 10.3 CentOS环境路由错误

RHEL/CentOS7 环境中iptables的策略关系，会导致路由通讯错误，需要手动调整iptables的桥接设置：

```bash
# cat /etc/sysctl.d/k8s.conf
 net.bridge.bridge-nf-call-ip6tables = 1
 net.bridge.bridge-nf-call-iptables = 1
```

#### 10.4 Token丢失

Master节点部署完成之后，会输出一个token用于minion节点的配置链接，不过这个token没有很方便的查看方式，导致此日志输出关闭后，没有token无法join minion节点，可以通过下述方式查看token：

```bash
kubectl -n kube-system get secret clusterinfo -o yaml | grep token-map | awk '{print $2}' | base64 --decode | sed "s|{||g;s|}||g;s|:|.|g;s/\"//g;" | xargs echo
```

建议提前使用`kubeadm token`命令生成token，然后在执行`kubeadm init`和`kubeadm join`的使用通过`--token`指定token。

#### 10.5 Vagrant中主机名的问题

如果使用Vagrant虚拟化环境部署kubernetes，首先得确保`hostname -i`能够获取正确的通讯IP，默认情况下，如果`/etc/hosts`中未配置主机名与IP的对应关系，kubelet会取第一个非lo网卡作为通讯入口，若这个网卡不做了NAT桥接的网卡，那安装就会出现问题。