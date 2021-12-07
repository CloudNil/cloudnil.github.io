---
layout: post
title:  基于飞腾麒麟架构快速部署kubernetes(1.21.6,HA)
categories: [docker,Cloud]
date: 2021-11-16 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>当前版本的kubeadm已经原生支持部署HA模式集群，非常方便即可实现HA模式的kubernetes集群。本次部署基于飞腾麒麟的ARM架构，原则上来说，其他ARM架构也适配，系统版本是：麒麟V10，docker版本：18.09.8，kubernetes适用1.21.x版本，本文采用1.21.6。

### 1 环境准备

准备了六台机器作安装测试工作，机器信息如下: 

|      IP      |   Name   |       Role      |    OS   |
|--------------|----------|-----------------|---------|
| 172.16.2.1   | Master01 | Controller,etcd | 麒麟V10 |
| 172.16.2.2   | Master02 | Controller,etcd | 麒麟V10 |
| 172.16.2.3   | Master03 | Controller,etcd | 麒麟V10 |
| 172.16.2.11  | Node01   | Compute         | 麒麟V10 |
| 172.16.2.12  | Node02   | Compute         | 麒麟V10 |
| 172.16.2.13  | Node03   | Compute         | 麒麟V10 |
| 172.16.2.251 | Dns01    | DNS             | 麒麟V10 |
| 172.16.2.252 | Dns01    | DNS             | 麒麟V10 |

关闭防火墙： 

```
systemctl stop firewalld
systemctl disable firewalld
```

关闭selinux：

```
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
```

关闭swap： 

```
swapoff -a
vim /etc/fstab
```

流量转发

```
cat > /etc/sysctl.d/k8s.conf << EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

更新仓库源

```
cat > /etc/yum.repos.d/kylin_aarch64.repo << EOF 
[ks10-adv-os]
name = Kylin Linux Advanced Server 10 - Os
#baseurl = http://archive.kylinos.cn/yum/v10/pks/aarch64/os/
baseurl = http://update.cs2c.com.cn:8080/NS/V10/V10SP1/os/adv/lic/base/aarch64
gpgcheck = 0
enabled = 1
EOF

yum clean all
yum update
yum makecache
```

>注意：需要在/etc/hosts中配置本机的主机名解析，如Master01，添加：172.16.2.1 master01 。

### 2 安装docker

```
#wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sed -i 's/$releasever/7/g' /etc/yum.repos.d/docker-ce.repo

wget https://mirrors.tuna.tsinghua.edu.cn/centos-altarch/7/extras/aarch64/Packages/container-selinux-2.107-1.el7_6.noarch.rpm

yum install policycoreutils-python
rpm -ivh container-selinux-2.107-1.el7_6.noarch.rpm

yum install docker-ce-18.09.9 docker-ce-cli-18.09.9 containerd.io
systemctl enable docker
systemctl start docker
```

### 3 安装etcd集群

使用了docker-compose安装，当然，如果觉得麻烦，也可以直接docker run。

安装docker-compose

```
yum install docker-compose
```

Master01节点的ETCD的docker-compose.yml：

```yaml
etcd:
  image: quay.io/coreos/etcd:v3.3.13-arm64
  command: etcd --name etcd-srv1 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.1:2379,http://172.16.2.1:2380 --initial-advertise-peer-urls http://172.16.2.1:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.2:2380,etcd-srv3=http://172.16.2.3:2380" -initial-cluster-state new
  net: "host"
  restart: always
  stdin_open: true
  tty: true
  environment:
  - ETCDCTL_API=3
  - ETCD_UNSUPPORTED_ARCH=arm64
  volumes:
  - /store/etcd:/var/etcd
```

Master02节点的ETCD的docker-compose.yml：

```yaml
etcd:
  image: quay.io/coreos/etcd:v3.3.13-arm64
  command: etcd --name etcd-srv2 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.2:2379,http://172.16.2.2:2380 --initial-advertise-peer-urls http://172.16.2.2:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.2:2380,etcd-srv3=http://172.16.2.3:2380" -initial-cluster-state new
  net: "host"
  restart: always
  stdin_open: true
  tty: true
  environment:
  - ETCDCTL_API=3
  - ETCD_UNSUPPORTED_ARCH=arm64
  volumes:
  - /store/etcd:/var/etcd
```

Master03节点的ETCD的docker-compose.yml：

```yaml
etcd:
  image: quay.io/coreos/etcd:v3.3.13-arm64
  command: etcd --name etcd-srv3 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.3:2379,http://172.16.2.3:2380 --initial-advertise-peer-urls http://172.16.2.3:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.2:2380,etcd-srv3=http://172.16.2.3:2380" -initial-cluster-state new
  net: "host"
  restart: always
  stdin_open: true
  tty: true
  environment:
  - ETCDCTL_API=3
  - ETCD_UNSUPPORTED_ARCH=arm64
  volumes:
  - /store/etcd:/var/etcd
```

创建好docker-compose.yml文件后，使用命令`docker-compose up -d`部署。

### 4 安装k8s工具包

使用阿里云提供的源安装软件包

```
cat > /etc/yum.repos.d/kubernetes.repo << EOF 
[kubernetes] 
name=Kubernetes 
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-aarch64 
enabled=1 
gpgcheck=1 
repo_gpgcheck=1 
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg  
EOF

yum install -y kubelet-1.21.6-0 kubeadm-1.21.6-0 kubectl-1.21.6-0 --disableexcludes=kubernetes
systemctl enable kubelet
systemctl start kubelet
```

### 5 启用ipvs模块

本方案中采用ipvs作为kube-proxy的转发机制，效率比iptables高很多，开启ipvs模块支持。启用的ipvs相关模块重启机器后需要重启加载，为了避免麻烦，可以将加载模块配置在为开机启动（所有节点上都需要配置），华为鲲鹏的CentOS7.5其实默认已经开启，可以跳过本步骤：

```
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs
```

### 6 Api-Server负载均衡

配置负载均衡器对kube-apiserver进行负载均衡，可采用DNS轮询解析或者Haproxy（Nginx）反向代理实现负载均衡。

本文采用DNS轮询解析实现简单的负载均衡，在Dns01,Dns02节点上部署DNS。

1、添加`/etc/dnsmasq/hosts.dnsmasq`文件，添加域名解析

```
172.16.2.1 api.k8s.com
172.16.2.2 api.k8s.com
172.16.2.3 api.k8s.com
```

2、docker-compose部署dnsmasq服务：

```yaml
version: "3"
services:
  dnsmasq:
    image: gists/dnsmasq:2.85
    command: -q --log-facility=- --addn-hosts=/etc/hosts.dnsmasq
    network_mode: "host"
    volumes:
    - /root/dnsmasq/hosts.dnsmasq:/etc/hosts.dnsmasq
    cap_add:
    - NET_ADMIN
    restart: always
    stdin_open: true
    tty: true
```

3、在除了部署dnsmasq服务的其他所有节点上(包括Master和Node)，配置DNS（如果有默认启动的DNS服务，需要先将其停止）

```
cat <<EOF >/etc/resolv.conf
nameserver 172.16.2.251
nameserver 172.16.2.252
EOF
```

### 7 安装master节点

kubeadm配置文件kubeadm-config.yaml：

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
etcd:
  external:
    endpoints:
    - http://172.16.2.1:2379
    - http://172.16.2.2:2379
    - http://172.16.2.3:2379
networking:
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.68.0.0/16
kubernetesVersion: v1.21.6
controlPlaneEndpoint: api.k8s.com:6443
apiServer:
  certSANs:
  - api.k8s.com
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
#麒麟V10不支持systemd，显示配置使用cgroupfs驱动
cgroupDriver: cgroupfs
systemReserved: 
  cpu: "0.25"
  memory: 128Mi
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
ipvs:
  minSyncPeriod: 1s
  #rr-轮询  wrr-加权轮询  sh-地址哈希
  scheduler: rr
  syncPeriod: 10s
mode: ipvs
```

master01初始化指令：

```
kubeadm init --config=kubeadm-config.yaml --upload-certs
```
如果镜像已经提前下载，安装过程大概30秒，输出结果如下：

```
[init] Using Kubernetes version: v1.21.6
[preflight] Running pre-flight checks
  [WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
  [WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master01] and IPs [10.96.0.1 api.k8s.com]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] External etcd mode: Skipping etcd/ca certificate authority generation
[certs] External etcd mode: Skipping etcd/server certificate generation
[certs] External etcd mode: Skipping etcd/peer certificate generation
[certs] External etcd mode: Skipping etcd/healthcheck-client certificate generation
[certs] External etcd mode: Skipping apiserver-etcd-client certificate generation
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 16.004044 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.21" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
8a36d44089b5a08270e511834ebd3f2a39be6644a78a702fbb1b6f1fceb8ea1f
[mark-control-plane] Marking the node master01 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: vbcbuv.osci507haylbl00i
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join api.k8s.com:6443 --token vbcbuv.osci507haylbl00i \
  --discovery-token-ca-cert-hash sha256:87540f79ca94e422f3600a1e4d3e8669d0f4125d46877e296b790810a1f9aade \
  --control-plane --certificate-key 8a36d44089b5a08270e511834ebd3f2a39be6644a78a702fbb1b6f1fceb8ea1f

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join api.k8s.com:6443 --token vbcbuv.osci507haylbl00i \
  --discovery-token-ca-cert-hash sha256:87540f79ca94e422f3600a1e4d3e8669d0f4125d46877e296b790810a1f9aade
```

>说明：`certificate-key`用于其他master节点获取证书文件时验证，有小时间为2小时，超过2小时候需要重新生成：
```
kubeadm init phase upload-certs --upload-certs
```
`token`是使用指令`kubeadm token generate`生成的，执行过程如有异常，用命令`kubeadm reset`初始化后重试，生成的`token`有效时间为24小时，超过24小时后需要重新使用命令`kubeadm token create`创建新的`token`，`discovery-token-ca-cert-hash`的值可以使用命令生成：
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

加载管理员配置文件

方式一：

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

方式二：

```
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### 8 安装calico网络

网络组件选择很多，可以根据自己的需要选择calico、weave、flannel，calico性能最好，flannel的vxlan也不错，默认的UDP性能较差，weave的性能比较差，测试环境用下可以，生产环境不建议使用。calico的安装配置可以参考官方部署：[点击查看](https://docs.projectcalico.org/v3.15/getting-started/kubernetes/installation/calico)

calico-rbac.yml：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-node
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-kube-controllers
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-kube-controllers
rules:
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
      - serviceaccounts
    verbs:
      - watch
      - list
      - get
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-kube-controllers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-kube-controllers
subjects:
- kind: ServiceAccount
  name: calico-kube-controllers
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-node
rules:
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - endpoints
      - services
    verbs:
      - watch
      - list
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - nodes/status
    verbs:
      - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: calico-node
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-node
subjects:
- kind: ServiceAccount
  name: calico-node
  namespace: kube-system
```

calico.yml：

```yaml
---
# Source: calico/templates/calico-etcd-secrets.yaml
# The following contains k8s Secrets for use with a TLS enabled etcd cluster.
# For information on populating Secrets, see http://kubernetes.io/docs/user-guide/secrets/
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # Populate the following with etcd TLS configuration if desired, but leave blank if
  # not using TLS for etcd.
  # The keys below should be uncommented and the values populated with the base64
  # encoded contents of each file that would be associated with the TLS data.
  # Example command for encoding a file contents: cat <file> | base64 -w 0
  # etcd-key: null
  # etcd-cert: null
  # etcd-ca: null
---
# Source: calico/templates/calico-config.yaml
# This ConfigMap is used to configure a self-hosted Calico installation.
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  # Configure this with the location of your etcd cluster.
  etcd_endpoints: "http://172.16.2.1:2379,http://172.16.2.2:2379,http://172.16.2.3:2379"
  # If you're using TLS enabled etcd uncomment the following.
  # You must also populate the Secret below with these files.
  etcd_ca: ""   # "/calico-secrets/etcd-ca"
  etcd_cert: "" # "/calico-secrets/etcd-cert"
  etcd_key: ""  # "/calico-secrets/etcd-key"
  # Typha is disabled.
  typha_service_name: "none"
  # Configure the backend to use.
  calico_backend: "bird"
  # Configure the MTU to use for workload interfaces and tunnels.
  # - If Wireguard is enabled, set to your network MTU - 60
  # - Otherwise, if VXLAN or BPF mode is enabled, set to your network MTU - 50
  # - Otherwise, if IPIP is enabled, set to your network MTU - 20
  # - Otherwise, if not using any encapsulation, set to your network MTU.
  veth_mtu: "1440"

  # The CNI network configuration to install on each node. The special
  # values in this config will be automatically populated.
  cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "etcd_endpoints": "__ETCD_ENDPOINTS__",
          "etcd_key_file": "__ETCD_KEY_FILE__",
          "etcd_cert_file": "__ETCD_CERT_FILE__",
          "etcd_ca_cert_file": "__ETCD_CA_CERT_FILE__",
          "mtu": __CNI_MTU__,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "__KUBECONFIG_FILEPATH__"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        },
        {
          "type": "bandwidth",
          "capabilities": {"bandwidth": true}
        }
      ]
    }
---
# Source: calico/templates/calico-node.yaml
# This manifest installs the calico-node container, as well
# as the CNI plugins and network config on
# each master and worker node in a Kubernetes cluster.
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      hostNetwork: true
      tolerations:
        # Make sure calico-node gets scheduled on all nodes.
        - effect: NoSchedule
          operator: Exists
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - effect: NoExecute
          operator: Exists
      serviceAccountName: calico-node
      # Minimize downtime during a rolling upgrade or deletion; tell Kubernetes to do a "force
      # deletion": https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods.
      terminationGracePeriodSeconds: 0
      priorityClassName: system-node-critical
      initContainers:
        # This container installs the CNI binaries
        # and CNI network config file on each node.
        - name: install-cni
          image: calico/cni:v3.15.0
          command: ["/install-cni.sh"]
          env:
            # Name of the CNI config file to create.
            - name: CNI_CONF_NAME
              value: "10-calico.conflist"
            # The CNI network config to install on each node.
            - name: CNI_NETWORK_CONFIG
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: cni_network_config
            # The location of the etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # CNI MTU Config variable
            - name: CNI_MTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # Prevents the container from sleeping forever.
            - name: SLEEP
              value: "false"
          volumeMounts:
            - mountPath: /host/opt/cni/bin
              name: cni-bin-dir
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
            - mountPath: /calico-secrets
              name: etcd-certs
          securityContext:
            privileged: true
        # Adds a Flex Volume Driver that creates a per-pod Unix Domain Socket to allow Dikastes
        # to communicate with Felix over the Policy Sync API.
        - name: flexvol-driver
          image: calico/pod2daemon-flexvol:v3.15.0
          volumeMounts:
          - name: flexvol-driver-host
            mountPath: /host/driver
          securityContext:
            privileged: true
      containers:
        # Runs calico-node container on each Kubernetes node. This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: calico/node:v3.15.0
          env:
            # The location of the etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
            # Set noderef for node controller.
            - name: CALICO_K8S_NODE_REF
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            # Choose the backend to use.
            - name: CALICO_NETWORKING_BACKEND
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: calico_backend
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            # Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"
            # Enable IPIP
            - name: CALICO_IPV4POOL_IPIP
              value: "Always"
            # Enable or Disable VXLAN on the default IP pool.
            - name: CALICO_IPV4POOL_VXLAN
              value: "Never"
            # Set MTU for tunnel device used if ipip is enabled
            - name: FELIX_IPINIPMTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # Set MTU for the VXLAN tunnel device.
            - name: FELIX_VXLANMTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # Set MTU for the Wireguard tunnel device.
            - name: FELIX_WIREGUARDMTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # The default IPv4 pool to create on startup if none exists. Pod IPs will be
            # chosen from this range. Changing this value after installation will have
            # no effect. This should fall within `--cluster-cidr`.
            # - name: CALICO_IPV4POOL_CIDR
            #   value: "192.168.0.0/16"
            # Set MTU for the Wireguard tunnel device.
            - name: FELIX_WIREGUARDMTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            # Set Felix logging to "info"
            - name: FELIX_LOGSEVERITYSCREEN
              value: "info"
            - name: FELIX_HEALTHENABLED
              value: "true"
          securityContext:
            privileged: true
          resources:
            requests:
              cpu: 250m
          livenessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-live
              - -bird-live
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-ready
              - -bird-ready
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /run/xtables.lock
              name: xtables-lock
              readOnly: false
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
            - mountPath: /var/lib/calico
              name: var-lib-calico
              readOnly: false
            - mountPath: /calico-secrets
              name: etcd-certs
            - name: policysync
              mountPath: /var/run/nodeagent
      volumes:
        # Used by calico-node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        - name: var-lib-calico
          hostPath:
            path: /var/lib/calico
        - name: xtables-lock
          hostPath:
            path: /run/xtables.lock
            type: FileOrCreate
        # Used to install CNI.
        - name: cni-bin-dir
          hostPath:
            path: /opt/cni/bin
        - name: cni-net-dir
          hostPath:
            path: /etc/cni/net.d
        # Mount in the etcd TLS secrets with mode 400.
        # See https://kubernetes.io/docs/concepts/configuration/secret/
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets
            defaultMode: 0400
        # Used to create per-pod Unix Domain Sockets
        - name: policysync
          hostPath:
            type: DirectoryOrCreate
            path: /var/run/nodeagent
        # Used to install Flex Volume Driver
        - name: flexvol-driver-host
          hostPath:
            type: DirectoryOrCreate
            path: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
---
# Source: calico/templates/calico-kube-controllers.yaml
# See https://github.com/projectcalico/kube-controllers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calico-kube-controllers
  namespace: kube-system
  labels:
    k8s-app: calico-kube-controllers
spec:
  # The controllers can only have a single active instance.
  replicas: 1
  selector:
    matchLabels:
      k8s-app: calico-kube-controllers
  strategy:
    type: Recreate
  template:
    metadata:
      name: calico-kube-controllers
      namespace: kube-system
      labels:
        k8s-app: calico-kube-controllers
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      tolerations:
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      serviceAccountName: calico-kube-controllers
      priorityClassName: system-cluster-critical
      # The controllers must run in the host network namespace so that
      # it isn't governed by policy that would prevent it from working.
      hostNetwork: true
      containers:
        - name: calico-kube-controllers
          image: calico/kube-controllers:v3.15.0
          env:
            # The location of the etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
            # Choose which controllers to run.
            - name: ENABLED_CONTROLLERS
              value: policy,namespace,serviceaccount,workloadendpoint,node
          volumeMounts:
            # Mount in the etcd TLS secrets.
            - mountPath: /calico-secrets
              name: etcd-certs
          readinessProbe:
            exec:
              command:
              - /usr/bin/check-status
              - -r
      volumes:
        # Mount in the etcd TLS secrets with mode 400.
        # See https://kubernetes.io/docs/concepts/configuration/secret/
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets
            defaultMode: 0400

```

执行命令：

```
kubectl apply -f calico-rbac.yml
kubectl apply -f calico.yml
```

检查各节点组件运行状态：

```
root@master01:~# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master01   Ready    master   13m   v1.21.6
root@master01:~# kubectl get po -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-697d964cc4-p8jcn   1/1     Running   0          32s
calico-node-wg9l4                          1/1     Running   0          32s
coredns-89cc84847-l48q8                    1/1     Running   0          13m
coredns-89cc84847-mf5nr                    1/1     Running   0          13m
kube-apiserver-master01                    1/1     Running   0          12m
kube-controller-manager-master01           1/1     Running   0          12m
kube-proxy-9l287                           1/1     Running   0          13m
kube-scheduler-master01                    1/1     Running   0          12m
```

### 9 安装Master02和Master03节点

在master02，master03上执行加入集群命令：

```
kubeadm join api.k8s.com:6443 --token vbcbuv.osci507haylbl00i \
  --discovery-token-ca-cert-hash sha256:87540f79ca94e422f3600a1e4d3e8669d0f4125d46877e296b790810a1f9aade \
  --control-plane --certificate-key 8a36d44089b5a08270e511834ebd3f2a39be6644a78a702fbb1b6f1fceb8ea1f
```

可以查看下各节点及组件运行状态：

```
root@master01:~# kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
master01   Ready    master   25m     v1.21.6
master02   Ready    master   8m6s    v1.21.6
master03   Ready    master   7m33s   v1.21.6
root@master01:~# kubectl get po -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-697d964cc4-p8jcn   1/1     Running   0          12m
calico-node-lvnl8                          1/1     Running   0          7m35s
calico-node-p8h5z                          1/1     Running   0          8m9s
calico-node-wg9l4                          1/1     Running   0          12m
coredns-89cc84847-l48q8                    1/1     Running   0          25m
coredns-89cc84847-mf5nr                    1/1     Running   0          25m
kube-apiserver-master01                    1/1     Running   0          24m
kube-apiserver-master02                    1/1     Running   0          8m9s
kube-apiserver-master03                    1/1     Running   0          7m35s
kube-controller-manager-master01           1/1     Running   0          24m
kube-controller-manager-master02           1/1     Running   0          8m9s
kube-controller-manager-master03           1/1     Running   0          7m35s
kube-proxy-9l287                           1/1     Running   0          25m
kube-proxy-jmsfb                           1/1     Running   0          8m9s
kube-proxy-wzh62                           1/1     Running   0          7m35s
kube-scheduler-master01                    1/1     Running   0          24m
kube-scheduler-master02                    1/1     Running   0          8m9s
kube-scheduler-master03                    1/1     Running   0          7m35s
```

### 10 安装Node节点

Master节点安装好了Node节点就简单了,在各个Node节点上执行。

```
kubeadm join api.k8s.com:6443 --token vbcbuv.osci507haylbl00i \
  --discovery-token-ca-cert-hash sha256:87540f79ca94e422f3600a1e4d3e8669d0f4125d46877e296b790810a1f9aade
```

### 11 DNS集群部署

删除原单点coredns

```
kubectl delete deploy coredns -n kube-system
```

部署多实例的coredns集群，参考配置coredns.yml：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-dns
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      affinity:
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                  - key: k8s-app
                    operator: In
                    values:
                    - kube-dns
                topologyKey: kubernetes.io/hostname
      containers:
      - args:
        - -conf
        - /etc/coredns/Corefile
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/coredns
          name: config-volume
          readOnly: true
      dnsPolicy: Default
      nodeSelector:
        beta.kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: coredns
      serviceAccountName: coredns
      terminationGracePeriodSeconds: 30
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: Corefile
            path: Corefile
          name: coredns
        name: config-volume
```

### 12 部署Metrics-Server

kubernetesv1.11以后不再支持通过`heaspter`采集监控数据，支持新的监控数据采集组件`metrics-server`，比`heaspter`轻量很多，也不做数据的持久化存储，提供实时的监控数据查询还是很好用的。

获取部署文档并修改为阿里云镜像仓库，项目内容可以[点击这里](https://github.com/kubernetes-sigs/metrics-server)。

```
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -O metrics-server.yaml
sed -i 's/k8s.gcr.io\/metrics-server/registry.cn-hangzhou.aliyuncs.com\/google_containers/g' metrics-server.yaml
sed -i '/- --metric-resolution=15s/a\        - --kubelet-insecure-tls' metrics-server.yaml
```

执行部署命令：

```
kubectl apply -f metrics-server.yaml
```

查看监控数据：

```
root@master01:~# kubectl top nodes
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
master01   153m         8%     1748Mi          46%       
master02   108m         6%     1250Mi          33%       
master03   91m          5%     1499Mi          40%       
node01     256m         7%     1047Mi          13%       
node02     196m         5%     976Mi           10%       
node03     206m         5%     907Mi           12%       
```

### 13 部署Dashboard

推荐`k8dash`,比官方的`dashboard`顺眼多了，项目地址：`https://github.com/skooner-k8s/skooner`。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8dash-sa
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: k8dash-sa
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: k8dash-sa
  namespace: kube-system
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: k8dash
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: k8dash
  template:
    metadata:
      labels:
        k8s-app: k8dash
    spec:
      containers:
      - name: k8dash
        image: sharnoth/k8dash
        ports:
        - containerPort: 4654
        livenessProbe:
          httpGet:
            scheme: HTTP
            path: /
            port: 4654
          initialDelaySeconds: 30
          timeoutSeconds: 30
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
---
kind: Service
apiVersion: v1
metadata:
  name: k8dash-svc
  namespace: kube-system
spec:
  ports:
    - port: 80
      targetPort: 4654
  selector:
    k8s-app: k8dash
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k8dash-ing
  labels:
    k8s-app: k8dash
  namespace: kube-system
spec:
  rules:
  - host: console.cloudnil.com
    http:
      paths:
      - path: /
        pathType: "Prefix"
        backend:
          service:
            name: k8dash-svc
            port: 
              number: 80
```

登录时需要输入`Token`，查看命令：

```
kubectl get secrets -n kube-system |grep k8dash-sa-token|awk '{print $1}'| xargs kubectl describe secret -n kube-system
```

暴露本地端口到Master01访问测试：

```
#直接暴露Service端口到本地
kubectl port-forward  svc/skooner --address 0.0.0.0 12345:80 -n kube-system
```

访问地址：`http://172.16.2.1:12345`。

### 14 服务暴露到公网

kubernetes中的Service暴露到外部有三种方式，分别是：

- LoadBlancer Service
- NodePort Service
- Ingress

LoadBlancer Service是kubernetes深度结合云平台的一个组件；当使用LoadBlancer Service暴露服务时，实际上是通过向底层云平台申请创建一个负载均衡器来向外暴露服务；目前LoadBlancer Service支持的云平台已经相对完善，比如国外的GCE、DigitalOcean，国内的 阿里云，私有云 Openstack 等等，由于LoadBlancer Service深度结合了云平台，所以只能在一些云平台上来使用。

NodePort Service顾名思义，实质上就是通过在集群的每个node上暴露一个端口，然后将这个端口映射到某个具体的service来实现的，虽然每个node的端口有很多(0~65535)，但是由于安全性和易用性(服务多了就乱了，还有端口冲突问题)实际使用可能并不多。

Ingress可以实现使用nginx等开源的反向代理负载均衡器实现对外暴露服务，可以理解Ingress就是用于配置域名转发的一个东西，在nginx中就类似upstream，它与ingress-controller结合使用，通过ingress-controller监控到pod及service的变化，动态地将ingress中的转发信息写到诸如nginx、apache、haproxy等组件中实现方向代理和负载均衡。

### 15 部署Traefik-ingress-controller

`Traefik-ingress-controller`是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。 它支持多种后台 (Docker, Swarm, Kubernetes, Marathon, Mesos, Consul, Etcd, Zookeeper, BoltDB, Rest API, file…) 来自动化、动态的应用它的配置文件设置。

相比起`Nginx`，`Traefik`更轻量，速度更快，配置更简单，不过功能及拓展性不如Nginx丰富多样，各位可根据实际情况选择。

本次部署中，将Traefik-ingress部署到`master01、master02、master03`上，监听宿主机的`80`端口:

Namespace 和 RBAC角色配置：

```
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-deploy
  namespace: traefik
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-deploy
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
      - ingressroutes
      - traefikservices
      - ingressroutetcps
      - ingressrouteudps
      - tlsoptions
      - tlsstores
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-deploy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-deploy
subjects:
- kind: ServiceAccount
  name: traefik-ingress-deploy
  namespace: traefik
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: tlsstores.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSStore
    plural: tlsstores
    singular: tlsstore
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: ingressrouteudps.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteUDP
    plural: ingressrouteudps
    singular: ingressrouteudp
  scope: Namespaced
```

Configmap和Deployment：

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: traefik-config
  namespace: traefik
  labels:
    app: traefik
data:
  traefik.yaml: |-
    ping: ""                    ## 启用 Ping
    serversTransport:
      insecureSkipVerify: true  ## Traefik 忽略验证代理服务的 TLS 证书
    api:
      insecure: true            ## 允许 HTTP 方式访问 API
      dashboard: true           ## 启用 Dashboard
      debug: false              ## 启用 Debug 调试模式
    # metrics:
    #   prometheus: ""            ## 配置 Prometheus 监控指标数据，并使用默认配置
    entryPoints:
      web:
        address: ":80"          ## 配置 80 端口，并设置入口名称为 web
      websecure:
        address: ":443"         ## 配置 443 端口，并设置入口名称为 websecure
    providers:
      kubernetesCRD: ""         ## 启用 Kubernetes CRD 方式来配置路由规则
      kubernetesIngress: ""     ## 启动 Kubernetes Ingress 方式来配置路由规则
    log:
      filePath: ""              ## 设置调试日志文件存储路径，如果为空则输出到控制台
      level: INFO               ## 设置调试日志级别
      format: common            ## 设置调试日志格式
    accessLog:
      filePath: ""              ## 设置访问日志文件存储路径，如果为空则输出到控制台
      format: common            ## 设置访问调试日志格式
      bufferingSize: 100        ## 设置访问日志缓存行数
      filters:
        statusCodes: ["403","413","500","502","503","504"]    ## 设置只保留指定状态码范围内的访问日志
        retryAttempts: true     ## 设置代理访问重试失败时，保留访问日志
        minDuration: 20         ## 设置保留请求时间超过指定持续时间的访问日志
      fields:                   ## 设置访问日志中的字段是否保留（keep 保留、drop 不保留）
        defaultMode: keep       ## 设置默认保留访问日志字段
        names:                  ## 针对访问日志特别字段特别配置保留模式
          ClientUsername: drop  
        headers:                ## 设置 Header 中字段是否保留
          defaultMode: keep     ## 设置默认保留 Header 中字段
          names:                ## 针对 Header 中特别字段特别配置保留模式
            User-Agent: redact
            Authorization: drop
            Content-Type: keep
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traefik-ingress-deploy
  namespace: traefik
  labels:
    app: traefik
spec:
  replicas: 3
  selector:
    matchLabels:
      app: traefik
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      name: traefik
      labels:
        app: traefik
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cluster.role
                operator: In
                values:
                - proxy
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/name
                    operator: In
                    values: 
                    - traefik-ingress-deploy
              topologyKey: "kubernetes.io/hostname"
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      serviceAccountName: traefik-ingress-deploy
      terminationGracePeriodSeconds: 1
      hostNetwork: true
      containers:
      - image: traefik:2.3.6
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        - name: admin
          containerPort: 8080
        resources:
          limits:
            cpu: 2
            memory: 2048Mi
          requests:
            cpu: 0.5
            memory: 1024Mi
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --configfile=/config/traefik.yaml
        volumeMounts:
        - mountPath: /config
          name: config
        readinessProbe:
          httpGet:
            path: /ping
            port: 8080
          failureThreshold: 2
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 5
        livenessProbe:
          httpGet:
            path: /ping
            port: 8080
          failureThreshold: 3
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5    
      volumes:
      - name: config
        configMap:
          name: traefik-config
```

部署完成之后，可以访问Traefik的控制台（只能看Ingress规则）：`MasterIP：8080`，更多配置可参考Traefik官网：`https://docs.traefik.io`。

### 16 Dashboard开启TLS访问

针对单个应用开启TLS安全访问的步骤：

- 1.Traefik配置443端口监听
- 2.创建证书Secret
- 3.Ingress上配置证书及使用的`Traefik Entrypoint`

修改Traefik Configmap配置文件中的entryPoints，修改更新Configmap后需要重建Traefik容器以便快速应用新的配置。

```
    entryPoints:
      web:
        address: ":80"          ## 配置 80 端口，并设置入口名称为 web
      websecure:
        address: ":443"         ## 配置 443 端口，并设置入口名称为 websecure
```

创建证书Secret可以使用自建证书或购买证书，自建证书为不受信任证书，地址栏访问时会提示不安全。

```
#生成证书
openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/cloudnil.com.key -x509 -days 365 -out certs/cloudnil.com.crt

#创建证书Secret
kubectl create secret tls tls-cert --cert=cloudnil.com.crt --key=cloudnil.com.key -n kube-system
```

修改k8dash的Ingress配置，可以使用原生Ingress或者Traefik拓展的`IngressRoute`（二选一）。

```yaml
#原生Ingress配置，推荐
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k8dash-ing
  labels:
    k8s-app: k8dash
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  tls:
  - hosts:
    - console.cloudnil.com
    secretName: tls-cert
  rules:
  - host: console.cloudnil.com
    http:
      paths:
      - path: /
        pathType: "Prefix"
        backend:
          service:
            name: k8dash-svc
            port: 
              number: 80
---
#Traefik IngressRoute配置
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: k8dash-ingroute
  labels:
    k8s-app: k8dash
  namespace: kube-system
spec:
  entryPoints:
  - websecure
  routes:
  - match: Host(`console.cloudnil.com`) && PathPrefix(`/`)
    kind: Rule
    services:
    - name: earth-web-svc
      port: 80
  tls:
    certResolver: tls-cert
```

### 17 结语

`Traefik-ingress-controller`部署完成之后，解析相关域名如`console.cloudnil.com`到master01、master02、master03的外网IP，就可以使用`console.cloudnil.com`访问dashboard，其他应用类似。

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2021/11/16/Deploy-kubernetes(1.21.6)-HA-with-kubeadm-on-kylin.md/。