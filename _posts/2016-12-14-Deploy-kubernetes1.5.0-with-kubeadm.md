---
layout: post
title:  kubeadm快速部署kubernetes1.5.0
categories: [docker,Cloud]
date: 2016-12-14 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>Kubernetes 1.5.0刚刚发布，添加了众多的新特性，我们的云平台也计划从Mesos和Rancher迁移到kubernetes，所以迫不及待地想尝试一下，Google出品，必属精品，无奈GFW的层层阻挡，让原本简单的部署步骤变得异常复杂，所以写下此文，供各位参考。

### 1 环境准备

准备了三台机器作安装测试工作，机器信息如下: 

|      IP      |   Name   |    Role    |     OS    |
|--------------|----------|------------|-----------|
| 172.16.1.101 | Master01 | Controller | Centos7.2 |
| 172.16.1.106 | Minion01 | Compute    | Centos7.2 |
| 172.16.1.107 | Minino02 | Compute    | Centos7.2 |

### 2 安装docker

```Bash
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

**官方源安装**

跨越GFW方式不细说，你懂的。

建议使用`yumdownloader`下载rpm包，不然那下载速度，会让各位对玩k8s失去兴趣的。

```Bash
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
systemctl enable kubelet && systemctl start kubelet
```

**非官方源安装**

```Bash
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

```Bash
git clone https://github.com/kubernetes/release.git
cd release/rpm
./docker-build.sh
```

编译完成后生成rpm包到：`/output/x86_64`，进入到该目录后安装rpm包，注意选择amd64的包（相信大多数同学都是64bit环境，如果是32bit或者arm架构请自行选择安装）。

### 4 下载docker镜像

kubeadm方式安装kubernetes集群需要的镜像在docker官方镜像中并未提供，只能去google的官方镜像库：`gcr.io` 中下载，GFW咋办？翻墙!也可以使用docker hub做跳板自己构建，这里针对k8s-1.5.0我已经做好镜像，各位可以直接下载。

kubernetes-1.5.0所需要的镜像：

- etcd-amd64:2.2.5
- kubedns-amd64:1.7
- kube-dnsmasq-amd64:1.3 
- exechealthz-amd64:1.1 
- pause-amd64:3.0
- kube-discovery-amd64:1.0
- kube-proxy-amd64:v1.5.0 
- kube-scheduler-amd64:v1.5.0
- kube-controller-manager-amd64:v1.5.0 
- kube-apiserver-amd64:v1.5.0 
- kubernetes-dashboard-amd64:v1.5.0

偷下懒吧，直接执行以下脚本：

```Bash
#!/bin/bash
images=(kube-proxy-amd64:v1.5.0 kube-discovery-amd64:1.0 kubedns-amd64:1.7 kube-scheduler-amd64:v1.5.0 kube-controller-manager-amd64:v1.5.0 kube-apiserver-amd64:v1.5.0 etcd-amd64:2.2.5 kube-dnsmasq-amd64:1.3 exechealthz-amd64:1.1 pause-amd64:3.0 kubernetes-dashboard-amd64:v1.5.0 
nginx-ingress-controller:0.8.3)
for imageName in ${images[@]} ; do
  docker pull cloudnil/$imageName
  docker tag cloudnil/$imageName gcr.io/google_containers/$imageName
  docker rmi cloudnil/$imageName
done
```

### 5 安装master节点

由于kubeadm和kubelet安装过程中会生成`/etc/kubernetes`目录，而`kubeadm init`会先检测该目录是否存在，所以我们先使用kubeadm初始化环境。

```Bash
kubeadm reset && systemctl start kubelet
kubeadm init --use-kubernetes-version v1.5.0 --pod-network-cidr=10.244.0.0/16
```

>说明：`--pod-network-cidr=10.244.0.0/16`是Flannel使用的网段，如果不打算使用Flannel的可以去掉该属性。如果有多网卡的，请根据实际情况配置`--api-advertise-addresses=<ip-address>`。

如果出现`ebtables not found in system path`的错误，要先安装`ebtables`包，我安装的过程中未提示，该包系统已经自带了。
```Bash
yum install -y ebtables
```

安装过程大概2-3分钟，输出结果如下：

```Bash
[kubeadm] WARNING: kubeadm is in alpha, please do not use it for production clusters.
[preflight] Running pre-flight checks
[init] Using Kubernetes version: v1.5.0
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

```Bash
kubeadm reset && systemctl start kubelet
kubeadm join --token=de3d61.504a049ec342e135 172.16.1.101
```

输出结果如下：

```Bash
[kubeadm] WARNING: kubeadm is in alpha, please do not use it for production clusters.
[preflight] Running pre-flight checks
[preflight] Starting the kubelet service
[tokens] Validating provided token
[discovery] Created cluster info discovery client, requesting info from "http://172.16.1.101:9898/cluster-info/v1/?token-id=f11877"
[discovery] Cluster info object received, verifying signature using given token
[discovery] Cluster info signature and contents are valid, will use API endpoints [https://172.16.1.101:6443]
[bootstrap] Trying to connect to endpoint https://172.16.1.101:6443
[bootstrap] Detected server version: v1.5.0
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
```Bash
[root@master ~]# kubectl get nodes
NAME       STATUS         AGE
master     Ready,master   6m
minion01   Ready          2m
minion02   Ready          2m
```

### 7 安装Flannel网络

网络组件选择很多，可以根据自己的需要选择calico、weave、flannel，calico性能最好，weave和flannel差不多。[Addons](http://kubernetes.io/docs/admin/addons/)中有配置好的yaml。

```Bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

如果使用的物理机器是阿里的VPC，请使用下边这个脚本创建网络，原生的flannel镜像不支持VPC，创建后网络是正常，但是会创建多个虚拟网卡，重启节点后网络失效。

```Bash
kubectl apply -f http://k8s.oss-cn-shanghai.aliyuncs.com/kube/flannel-vpc.yml
```

>说明：注意配置其中的Network与kubeadm init命令中的pod-network-cidr一致、ACCESS_KEY_ID、ACCESS_KEY_SECRET是阿里云的KEY。

检查各节点组件运行状态：

```Bash
[root@master ~]# kubectl get po -n=kube-system -o wide
NAME                                    READY     STATUS    RESTARTS   AGE       IP             NODE
dummy-2088944543-md22p                  1/1       Running   0          17m        172.16.1.101   master
etcd-master                             1/1       Running   0          19m        172.16.1.101   master
kube-apiserver-master                   1/1       Running   0          19m        172.16.1.101   master
kube-controller-manager-master          1/1       Running   0          17m        172.16.1.101   master
kube-discovery-2220406887-jqsdz         1/1       Running   0          17m        172.16.1.101   master
kube-dns-654381707-sxjsj                3/3       Running   0          17m        10.244.0.3     master
kube-flannel-ds-8dmw4                   2/2       Running   0          10m        172.16.1.106   minion01
kube-flannel-ds-gljhj                   2/2       Running   0          10m        172.16.1.107   minion02
kube-flannel-ds-kpwjd                   2/2       Running   0          10m        172.16.1.101   master
kube-proxy-sx5l9                        1/1       Running   0          17m        172.16.1.106   minion01
kube-proxy-59x41                        1/1       Running   0          17m        172.16.1.101   master
kube-proxy-pn1j9                        1/1       Running   0          17m        172.16.1.107   minion02
kube-scheduler-master                   1/1       Running   0          17m        172.16.1.101   master
```
>说明：kube-dns需要等flannel配置完成后才是running状态。

### 8 部署Dashboard

下载kubernetes-dashboard.yaml

```Bash
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

```Bash
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

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2016/12/14/Deploy-kubernetes1.5.0-with-kubeadm/。