---
layout: post
title:  kubeadm快速部署kubernetes(1.26.4,HA)
categories: [docker,Cloud]
date: 2023-04-18 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>当前版本的kubeadm已经原生支持部署HA模式集群，非常方便即可实现HA模式的kubernetes集群。本次部署基于Ubuntu20.04，并使用最新的docker版本：20.10.24，kubernetes适用1.26.x版本，本文采用1.26.4。

### 1 环境准备

准备了六台机器作安装测试工作，机器信息如下: 

|      IP      |   Name   |       Role      |      OS     |
|--------------|----------|-----------------|-------------|
| 172.16.2.1   | Master01 | Controller,etcd | Ubuntu20.04 |
| 172.16.2.2   | Master02 | Controller,etcd | Ubuntu20.04 |
| 172.16.2.3   | Master03 | Controller,etcd | Ubuntu20.04 |
| 172.16.2.11  | Node01   | Compute         | Ubuntu20.04 |
| 172.16.2.12  | Node02   | Compute         | Ubuntu20.04 |
| 172.16.2.13  | Node03   | Compute         | Ubuntu20.04 |
| 172.16.2.251 | Dns01    | DNS             | Ubuntu20.04 |
| 172.16.2.252 | Dns01    | DNS             | Ubuntu20.04 |

>注意：需要在/etc/hosts中配置本机的主机名解析，如Master01，添加：172.16.2.1 master01 。

关闭Ubuntu20.04自带的dns解析编辑`/etc/systemd/resolved.conf`，修改配置如下：
```
[Resolve]
DNS=114.114.114.114 8.8.8.8
#FallbackDNS=
#Domains=
#LLMNR=no
#MulticastDNS=no
#DNSSEC=no
#DNSOverTLS=no
#Cache=no-negative
DNSStubListener=no
#ReadEtcHosts=yes
```

创建软链接并删除现有的`resolved.conf`，重启服务：

```
ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf

systemctl restart systemd-resolved.service
```

### 2 安装docker

```
apt update

apt install ca-certificates curl

install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update

VERSION_STRING=5:20.10.24~3-0~ubuntu-focal
apt install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin
```

配置docker使用systemd驱动，相比默认的cgroups更稳定。

```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload && systemctl restart docker
```

Kubernetes自v1.24移除了对docker-shim的支持，而Docker默认又不支持CRI规范，因而二者将无法直接完成整合。为此，Mirantis和Docker联合创建了cri-dockerd项目，用于为Docker Engine提供一个能够支持到CRI规范的垫片，从而能够让Kubernetes基于CRI控制Docker。

```
curl -LO https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd_0.3.1.3-0.ubuntu-focal_amd64.deb 

apt install ./cri-dockerd_0.3.1.3-0.ubuntu-focal_amd64.deb

##确认服务启动情况
systemctl status cri-docker.service
```

配置cri-dockerd，编辑文件`/usr/lib/systemd/system/cri-docker.service`，修改配置参数：
```
ExecStart=/usr/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9
```

重启服务
```
systemctl daemon-reload && systemctl restart cri-docker.service
```

### 3 安装etcd集群

使用了docker-compose安装，当然，如果觉得麻烦，也可以直接docker run。

Master01节点的ETCD的docker-compose.yml：

```yaml
version: "3.7"
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.6
    command: etcd --name etcd-srv1 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.1:2379,http://172.16.2.1:2380 --initial-advertise-peer-urls http://172.16.2.1:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.2:2380,etcd-srv3=http://172.16.2.3:2380" -initial-cluster-state new
    network_mode: "host"
    restart: always
    stdin_open: true
    tty: true
    environment:
    - ETCDCTL_API=3
    volumes:
    - /store/etcd:/var/etcd
```

Master02节点的ETCD的docker-compose.yml：

```yaml
version: "3.7"
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.6
    command: etcd --name etcd-srv2 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.2:2379,http://172.16.2.2:2380 --initial-advertise-peer-urls http://172.16.2.2:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.2:2380,etcd-srv3=http://172.16.2.3:2380" -initial-cluster-state new
    network_mode: "host"
    restart: always
    stdin_open: true
    tty: true
    environment:
    - ETCDCTL_API=3
    volumes:
    - /store/etcd:/var/etcd
```

Master03节点的ETCD的docker-compose.yml：

```yaml
version: "3.7"
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.6
    command: etcd --name etcd-srv3 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.3:2379,http://172.16.2.3:2380 --initial-advertise-peer-urls http://172.16.2.3:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.2:2380,etcd-srv3=http://172.16.2.3:2380" -initial-cluster-state new
    network_mode: "host"
    restart: always
    stdin_open: true
    tty: true
    environment:
    - ETCDCTL_API=3
    volumes:
    - /store/etcd:/var/etcd
```

创建好docker-compose.yml文件后，使用命令`docker compose up -d`部署。

>关于docker-compose的使用，可以参考：[docker compose安装文档](https://docs.docker.com/compose/install/#alternative-install-options)。

### 4 安装k8s工具包

**阿里源安装**

```
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

apt update
apt install -y kubelet=1.26.4-00 kubeadm=1.26.4-00 kubectl=1.26.4-00 ipvsadm ipset
apt-mark hold kubelet kubeadm kubectl ipvsadm docker-ce
```

### 5 启用ipvs模块

本方案中采用ipvs作为kube-proxy的转发机制，效率比iptables高很多，开启ipvs模块支持。

```
modprobe ip_vs && modprobe ip_vs_rr && modprobe ip_vs_wrr && modprobe ip_vs_sh
```

启用的ipvs相关模块重启机器后需要重启加载，为了避免麻烦，可以将加载模块配置在为开机启动（所有节点上都需要配置）：

```
root@master01:~# vi /etc/modules

# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
ip_vs_rr
ip_vs_wrr
ip_vs_sh
ip_vs
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
    image: cloudnil/dnsmasq:2.76
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
>说明：Ubuntu18.04 默认会开启本地DNS服务，要先关闭默认DNS服务并配置禁止启动：`systemctl disable systemd-resolved`，`systemctl stop systemd-resolved`，同时修改节点DNS配置文件`/etc/resolv.conf`中的`nameserver`为公网DNS服务器IP。

3、在除了部署dnsmasq服务的其他所有节点上(包括Master和Node)，配置DNS（同时也要先停用本机默认启动的DNS服务）

```
cat <<EOF >/etc/resolv.conf
nameserver 172.16.2.251
nameserver 172.16.2.252
EOF
```

### 7 安装master节点

kubeadm配置文件kubeadm-config.yaml：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
imageRepository: registry.aliyuncs.com/google_containers
etcd:
  external:
    endpoints:
    - http://172.16.2.1:2379
    - http://172.16.2.2:2379
    - http://172.16.2.3:2379
networking:
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.68.0.0/16
kubernetesVersion: v1.26.4
controlPlaneEndpoint: api.k8s.com:6443
apiServer:
  certSANs:
  - api.k8s.com
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
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

>说明：因为k8s.io在墙外，导致镜像无法获取，感谢阿里云提供了镜像仓库：`registry.aliyuncs.com/google_containers`，镜像下载需要点时间，也可以提前下载镜像：`kubeadm config images pull --config kubeadm-config.yaml`。

相关镜像：
```
registry.aliyuncs.com/google_containers/kube-apiserver:v1.26.4
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.26.4
registry.aliyuncs.com/google_containers/kube-scheduler:v1.26.4
registry.aliyuncs.com/google_containers/kube-proxy:v1.26.4
registry.aliyuncs.com/google_containers/pause:3.9
registry.aliyuncs.com/google_containers/coredns:v1.9.3
```

master01初始化指令：

```
kubeadm init --config=kubeadm-config.yaml --upload-certs
```
如果镜像已经提前下载，安装过程大概30秒，输出结果如下：

```
[init] Using Kubernetes version: v1.26.4
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local yny2023141] and IPs [10.96.0.1 172.16.2.1]
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
[apiclient] All control plane components are healthy after 7.507426 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node yny2023141 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node yny2023141 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: xn39cs.mjlg0er3cqr8t5jd
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join api.k8s.com:6443 --token obrxsw.2wzhv0z6nfnc4nyr \
    --discovery-token-ca-cert-hash sha256:312a5bab2f699a4e701fb08ce65218b01aff7521fc962f35ad56e5d3214d3e6f \
    --control-plane --certificate-key fbf4fd608065cb4478a6269de8ca7402ceec7640209df73dbdef254e3d02efbd

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join api.k8s.com:6443 --token obrxsw.2wzhv0z6nfnc4nyr \
    --discovery-token-ca-cert-hash sha256:312a5bab2f699a4e701fb08ce65218b01aff7521fc962f35ad56e5d3214d3e6f
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

网络组件选择很多，可以根据自己的需要选择calico、weave、flannel，calico性能最好，flannel的vxlan也不错，默认的UDP性能较差，weave的性能比较差，测试环境用下可以，生产环境不建议使用。calico的安装配置可以参考官方部署：[点击查看](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-etcd-datastore)

calico.yml：

```yaml
---
# Source: calico/templates/calico-kube-controllers.yaml
# This manifest creates a Pod Disruption Budget for Controller to allow K8s Cluster Autoscaler to evict

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: calico-kube-controllers
  namespace: kube-system
  labels:
    k8s-app: calico-kube-controllers
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: calico-kube-controllers
---
# Source: calico/templates/calico-kube-controllers.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-kube-controllers
  namespace: kube-system
---
# Source: calico/templates/calico-node.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-node
  namespace: kube-system
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
  etcd_endpoints: "http://172.16.7.141:2379"
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
  # By default, MTU is auto-detected, and explicitly setting this field should not be required.
  # You can override auto-detection by providing a non-zero value.
  veth_mtu: "0"

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
          "log_file_path": "/var/log/calico/cni/cni.log",
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
# Source: calico/templates/calico-kube-controllers-rbac.yaml
# Include a clusterrole for the kube-controllers component,
# and bind it to the calico-kube-controllers serviceaccount.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-kube-controllers
rules:
  # Pods are monitored for changing labels.
  # The node controller monitors Kubernetes nodes.
  # Namespace and serviceaccount labels are used for policy.
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
  # Watch for changes to Kubernetes NetworkPolicies.
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
---
# Source: calico/templates/calico-node-rbac.yaml
# Include a clusterrole for the calico-node DaemonSet,
# and bind it to the calico-node serviceaccount.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-node
rules:
  # Used for creating service account tokens to be used by the CNI plugin
  - apiGroups: [""]
    resources:
      - serviceaccounts/token
    resourceNames:
      - calico-node
    verbs:
      - create
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  # EndpointSlices are used for Service-based network policy rule
  # enforcement.
  - apiGroups: ["discovery.k8s.io"]
    resources:
      - endpointslices
    verbs:
      - watch 
      - list
  - apiGroups: [""]
    resources:
      - endpoints
      - services
    verbs:
      # Used to discover service IPs for advertisement.
      - watch
      - list
  # Pod CIDR auto-detection on kubeadm needs access to config maps.
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - nodes/status
    verbs:
      # Needed for clearing NodeNetworkUnavailable flag.
      - patch
---
# Source: calico/templates/calico-kube-controllers-rbac.yaml
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
# Source: calico/templates/calico-node-rbac.yaml
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
          image: docker.io/calico/cni:v3.25.1
          imagePullPolicy: IfNotPresent
          command: ["/opt/cni/bin/install"]
          envFrom:
          - configMapRef:
              # Allow KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT to be overridden for eBPF mode.
              name: kubernetes-services-endpoint
              optional: true
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
        # This init container mounts the necessary filesystems needed by the BPF data plane
        # i.e. bpf at /sys/fs/bpf and cgroup2 at /run/calico/cgroup. Calico-node initialisation is executed
        # in best effort fashion, i.e. no failure for errors, to not disrupt pod creation in iptable mode.
        - name: "mount-bpffs"
          image: docker.io/calico/node:v3.25.1
          imagePullPolicy: IfNotPresent
          command: ["calico-node", "-init", "-best-effort"]
          volumeMounts:
            - mountPath: /sys/fs
              name: sys-fs
              # Bidirectional is required to ensure that the new mount we make at /sys/fs/bpf propagates to the host
              # so that it outlives the init container.
              mountPropagation: Bidirectional
            - mountPath: /var/run/calico
              name: var-run-calico
              # Bidirectional is required to ensure that the new mount we make at /run/calico/cgroup propagates to the host
              # so that it outlives the init container.
              mountPropagation: Bidirectional
            # Mount /proc/ from host which usually is an init program at /nodeproc. It's needed by mountns binary,
            # executed by calico-node, to mount root cgroup2 fs at /run/calico/cgroup to attach CTLB programs correctly.
            - mountPath: /nodeproc
              name: nodeproc
              readOnly: true
          securityContext:
            privileged: true
      containers:
        # Runs calico-node container on each Kubernetes node. This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: docker.io/calico/node:v3.25.1
          imagePullPolicy: IfNotPresent
          envFrom:
          - configMapRef:
              # Allow KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT to be overridden for eBPF mode.
              name: kubernetes-services-endpoint
              optional: true
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
            # Enable or Disable VXLAN on the default IPv6 IP pool.
            - name: CALICO_IPV6POOL_VXLAN
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
            - name: CALICO_IPV4POOL_CIDR
              value: "10.68.0.0/16"
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            - name: FELIX_HEALTHENABLED
              value: "true"
          securityContext:
            privileged: true
          resources:
            requests:
              cpu: 250m
          lifecycle:
            preStop:
              exec:
                command:
                - /bin/calico-node
                - -shutdown
          livenessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-live
              - -bird-live
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
            timeoutSeconds: 10
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-ready
              - -bird-ready
            periodSeconds: 10
            timeoutSeconds: 10
          volumeMounts:
            # For maintaining CNI plugin API credentials.
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
              readOnly: false
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
            # For eBPF mode, we need to be able to mount the BPF filesystem at /sys/fs/bpf so we mount in the
            # parent directory.
            - name: bpffs
              mountPath: /sys/fs/bpf
            - name: cni-log-dir
              mountPath: /var/log/calico/cni
              readOnly: true
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
        - name: sys-fs
          hostPath:
            path: /sys/fs/
            type: DirectoryOrCreate
        - name: bpffs
          hostPath:
            path: /sys/fs/bpf
            type: Directory
        # mount /proc at /nodeproc to be used by mount-bpffs initContainer to mount root cgroup2 fs.
        - name: nodeproc
          hostPath:
            path: /proc
        # Used to install CNI.
        - name: cni-bin-dir
          hostPath:
            path: /opt/cni/bin
        - name: cni-net-dir
          hostPath:
            path: /etc/cni/net.d
        # Used to access CNI logs.
        - name: cni-log-dir
          hostPath:
            path: /var/log/calico/cni
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
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      serviceAccountName: calico-kube-controllers
      priorityClassName: system-cluster-critical
      # The controllers must run in the host network namespace so that
      # it isn't governed by policy that would prevent it from working.
      hostNetwork: true
      containers:
        - name: calico-kube-controllers
          image: docker.io/calico/kube-controllers:v3.25.1
          imagePullPolicy: IfNotPresent
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
          livenessProbe:
            exec:
              command:
              - /usr/bin/check-status
              - -l
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
            timeoutSeconds: 10
          readinessProbe:
            exec:
              command:
              - /usr/bin/check-status
              - -r
            periodSeconds: 10
      volumes:
        # Mount in the etcd TLS secrets with mode 400.
        # See https://kubernetes.io/docs/concepts/configuration/secret/
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets
            defaultMode: 0440
```
执行命令：

```
kubectl apply -f calico.yml
```

检查各节点组件运行状态：

```
root@master01:~# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master01   Ready    master   13m   v1.26.4

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
kubeadm join api.k8s.com:6443 --token obrxsw.2wzhv0z6nfnc4nyr \
    --discovery-token-ca-cert-hash sha256:312a5bab2f699a4e701fb08ce65218b01aff7521fc962f35ad56e5d3214d3e6f \
    --control-plane --certificate-key fbf4fd608065cb4478a6269de8ca7402ceec7640209df73dbdef254e3d02efbd
```

可以查看下各节点及组件运行状态：

```
root@master01:~# kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
master01   Ready    master   25m     v1.26.4
master02   Ready    master   8m6s    v1.26.4
master03   Ready    master   7m33s   v1.26.4
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
kubeadm join api.k8s.com:6443 --token obrxsw.2wzhv0z6nfnc4nyr \
    --discovery-token-ca-cert-hash sha256:312a5bab2f699a4e701fb08ce65218b01aff7521fc962f35ad56e5d3214d3e6f \
    --cri-socket unix:///run/cri-dockerd.sock
```

### 11 DNS集群部署

删除原单点coredns

```
kubectl delete deploy coredns -n kube-system
```

部署多实例的coredns集群，参考配置coredns.yml：

```yaml
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
        image: registry.aliyuncs.com/google_containers/coredns:1.9.3
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

获取部署文档，[点击这里](https://github.com/kubernetes-sigs/metrics-server)。

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: system:aggregated-metrics-reader
rules:
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: metrics-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        image: registry.aliyuncs.com/google_containers/metrics-server:v0.6.3
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 4443
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
```

执行部署命令：

```
kubectl apply -f components.yaml
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

推荐`k8dash`,比官方的`dashboard`顺眼多了，项目地址：`https://github.com/herbrandson/k8dash`。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8dash-sa
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
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
        image: herbrandson/k8dash:latest
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
#直接暴露Pod端口到本地
kubectl port-forward  pod/k8dash-fc78cd558-thdrv --address 0.0.0.0 12345:4654 -n kube-system

#直接暴露Service端口到本地
kubectl port-forward  svc/k8dash-svc --address 0.0.0.0 12345:80 -n kube-system
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

### 15 部署Nginx-ingress-controller

>说明：Nginx-controller和Traefik-controller二选一。

`Nginx-ingress-controller`是kubernetes官方提供的集成了Ingress-controller和Nginx的一个docker镜像。

配置可参考Nginx-ingress-controller官网：`https://kubernetes.github.io/ingress-nginx`。

### 16 部署Traefik-ingress-controller

>说明：Nginx-ingress-controller和Traefik-ingress-controller二选一。

`Traefik-ingress-controller`是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。 它支持多种后台 (Docker, Swarm, Kubernetes, Marathon, Mesos, Consul, Etcd, Zookeeper, BoltDB, Rest API, file…) 来自动化、动态的应用它的配置文件设置。

相比起`Nginx`，`Traefik`更轻量，速度更快，配置更简单，不过功能及拓展性不如Nginx丰富多样，各位可根据实际情况选择。

本次部署中，将Traefik-ingress部署到`master01、master02、master03`上，监听宿主机的`80`端口:

Namespace 和 RBAC角色配置：

```yaml
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
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
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
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
      - middlewaretcps
      - ingressroutes
      - traefikservices
      - ingressroutetcps
      - ingressrouteudps
      - tlsoptions
      - tlsstores
      - serverstransports
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
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
```

CRD配置：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.6.2
  creationTimestamp: null
  name: ingressroutes.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: IngressRoute
    listKind: IngressRouteList
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: IngressRoute is the CRD implementation of a Traefik HTTP Router.
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: IngressRouteSpec defines the desired state of IngressRoute.
            properties:
              entryPoints:
                description: 'EntryPoints defines the list of entry point names to
                  bind to. Entry points have to be configured in the static configuration.
                  More info: https://doc.traefik.io/traefik/v2.9/routing/entrypoints/
                  Default: all.'
                items:
                  type: string
                type: array
              routes:
                description: Routes defines the list of routes.
                items:
                  description: Route holds the HTTP route configuration.
                  properties:
                    kind:
                      description: Kind defines the kind of the route. Rule is the
                        only supported kind.
                      enum:
                      - Rule
                      type: string
                    match:
                      description: 'Match defines the router''s rule. More info: https://doc.traefik.io/traefik/v2.9/routing/routers/#rule'
                      type: string
                    middlewares:
                      description: 'Middlewares defines the list of references to
                        Middleware resources. More info: https://doc.traefik.io/traefik/v2.9/routing/providers/kubernetes-crd/#kind-middleware'
                      items:
                        description: MiddlewareRef is a reference to a Middleware
                          resource.
                        properties:
                          name:
                            description: Name defines the name of the referenced Middleware
                              resource.
                            type: string
                          namespace:
                            description: Namespace defines the namespace of the referenced
                              Middleware resource.
                            type: string
                        required:
                        - name
                        type: object
                      type: array
                    priority:
                      description: 'Priority defines the router''s priority. More
                        info: https://doc.traefik.io/traefik/v2.9/routing/routers/#priority'
                      type: integer
                    services:
                      description: Services defines the list of Service. It can contain
                        any combination of TraefikService and/or reference to a Kubernetes
                        Service.
                      items:
                        description: Service defines an upstream HTTP service to proxy
                          traffic to.
                        properties:
                          kind:
                            description: Kind defines the kind of the Service.
                            enum:
                            - Service
                            - TraefikService
                            type: string
                          name:
                            description: Name defines the name of the referenced Kubernetes
                              Service or TraefikService. The differentiation between
                              the two is specified in the Kind field.
                            type: string
                          namespace:
                            description: Namespace defines the namespace of the referenced
                              Kubernetes Service or TraefikService.
                            type: string
                          passHostHeader:
                            description: PassHostHeader defines whether the client
                              Host header is forwarded to the upstream Kubernetes
                              Service. By default, passHostHeader is true.
                            type: boolean
                          port:
                            anyOf:
                            - type: integer
                            - type: string
                            description: Port defines the port of a Kubernetes Service.
                              This can be a reference to a named port.
                            x-kubernetes-int-or-string: true
                          responseForwarding:
                            description: ResponseForwarding defines how Traefik forwards
                              the response from the upstream Kubernetes Service to
                              the client.
                            properties:
                              flushInterval:
                                description: 'FlushInterval defines the interval,
                                  in milliseconds, in between flushes to the client
                                  while copying the response body. A negative value
                                  means to flush immediately after each write to the
                                  client. This configuration is ignored when ReverseProxy
                                  recognizes a response as a streaming response; for
                                  such responses, writes are flushed to the client
                                  immediately. Default: 100ms'
                                type: string
                            type: object
                          scheme:
                            description: Scheme defines the scheme to use for the
                              request to the upstream Kubernetes Service. It defaults
                              to https when Kubernetes Service port is 443, http otherwise.
                            type: string
                          serversTransport:
                            description: ServersTransport defines the name of ServersTransport
                              resource to use. It allows to configure the transport
                              between Traefik and your servers. Can only be used on
                              a Kubernetes Service.
                            type: string
                          sticky:
                            description: 'Sticky defines the sticky sessions configuration.
                              More info: https://doc.traefik.io/traefik/v2.9/routing/services/#sticky-sessions'
                            properties:
                              cookie:
                                description: Cookie defines the sticky cookie configuration.
                                properties:
                                  httpOnly:
                                    description: HTTPOnly defines whether the cookie
                                      can be accessed by client-side APIs, such as
                                      JavaScript.
                                    type: boolean
                                  name:
                                    description: Name defines the Cookie name.
                                    type: string
                                  sameSite:
                                    description: 'SameSite defines the same site policy.
                                      More info: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite'
                                    type: string
                                  secure:
                                    description: Secure defines whether the cookie
                                      can only be transmitted over an encrypted connection
                                      (i.e. HTTPS).
                                    type: boolean
                                type: object
                            type: object
                          strategy:
                            description: Strategy defines the load balancing strategy
                              between the servers. RoundRobin is the only supported
                              value at the moment.
                            type: string
                          weight:
                            description: Weight defines the weight and should only
                              be specified when Name references a TraefikService object
                              (and to be precise, one that embeds a Weighted Round
                              Robin).
                            type: integer
                        required:
                        - name
                        type: object
                      type: array
                  required:
                  - kind
                  - match
                  type: object
                type: array
              tls:
                description: 'TLS defines the TLS configuration. More info: https://doc.traefik.io/traefik/v2.9/routing/routers/#tls'
                properties:
                  certResolver:
                    description: 'CertResolver defines the name of the certificate
                      resolver to use. Cert resolvers have to be configured in the
                      static configuration. More info: https://doc.traefik.io/traefik/v2.9/https/acme/#certificate-resolvers'
                    type: string
                  domains:
                    description: 'Domains defines the list of domains that will be
                      used to issue certificates. More info: https://doc.traefik.io/traefik/v2.9/routing/routers/#domains'
                    items:
                      description: Domain holds a domain name with SANs.
                      properties:
                        main:
                          description: Main defines the main domain name.
                          type: string
                        sans:
                          description: SANs defines the subject alternative domain
                            names.
                          items:
                            type: string
                          type: array
                      type: object
                    type: array
                  options:
                    description: 'Options defines the reference to a TLSOption, that
                      specifies the parameters of the TLS connection. If not defined,
                      the `default` TLSOption is used. More info: https://doc.traefik.io/traefik/v2.9/https/tls/#tls-options'
                    properties:
                      name:
                        description: 'Name defines the name of the referenced TLSOption.
                          More info: https://doc.traefik.io/traefik/v2.9/routing/providers/kubernetes-crd/#kind-tlsoption'
                        type: string
                      namespace:
                        description: 'Namespace defines the namespace of the referenced
                          TLSOption. More info: https://doc.traefik.io/traefik/v2.9/routing/providers/kubernetes-crd/#kind-tlsoption'
                        type: string
                    required:
                    - name
                    type: object
                  secretName:
                    description: SecretName is the name of the referenced Kubernetes
                      Secret to specify the certificate details.
                    type: string
                  store:
                    description: Store defines the reference to the TLSStore, that
                      will be used to store certificates. Please note that only `default`
                      TLSStore can be used.
                    properties:
                      name:
                        description: 'Name defines the name of the referenced TLSStore.
                          More info: https://doc.traefik.io/traefik/v2.9/routing/providers/kubernetes-crd/#kind-tlsstore'
                        type: string
                      namespace:
                        description: 'Namespace defines the namespace of the referenced
                          TLSStore. More info: https://doc.traefik.io/traefik/v2.9/routing/providers/kubernetes-crd/#kind-tlsstore'
                        type: string
                    required:
                    - name
                    type: object
                type: object
            required:
            - routes
            type: object
        required:
        - metadata
        - spec
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []

---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.6.2
  creationTimestamp: null
  name: ingressroutetcps.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: IngressRouteTCP
    listKind: IngressRouteTCPList
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: IngressRouteTCP is the CRD implementation of a Traefik TCP Router.
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: IngressRouteTCPSpec defines the desired state of IngressRouteTCP.
            properties:
              entryPoints:
                description: 'EntryPoints defines the list of entry point names to
                  bind to. Entry points have to be configured in the static configuration.
                  More info: https://doc.traefik.io/traefik/v2.9/routing/entrypoints/
                  Default: all.'
                items:
                  type: string
                type: array
              routes:
                description: Routes defines the list of routes.
                items:
                  description: RouteTCP holds the TCP route configuration.
                  properties:
                    match:
                      description: 'Match defines the router''s rule. More info: https://doc.traefik.io/traefik/v2.9/routing/routers/#rule_1'
                      type: string
                    middlewares:
                      description: Middlewares defines the list of references to MiddlewareTCP
                        resources.
                      items:
                        description: ObjectReference is a generic reference to a Traefik
                          resource.
                        properties:
                          name:
                            description: Name defines the name of the referenced Traefik
                              resource.
                            type: string
                          namespace:
                            description: Namespace defines the namespace of the referenced
                              Traefik resource.
                            type: string
                        required:
                        - name
                        type: object
                      type: array
                    priority:
                      description: 'Priority defines the router''s priority. More
                        info: https://doc.traefik.io/traefik/v2.9/routing/routers/#priority_1'
                      type: integer
                    services:
                      description: Services defines the list of TCP services.
                      items:
                        description: ServiceTCP defines an upstream TCP service to
                          proxy traffic to.
                        properties:
                          name:
                            description: Name defines the name of the referenced Kubernetes
                              Service.
                            type: string
                          namespace:
                            description: Namespace defines the namespace of the referenced
                              Kubernetes Service.
                            type: string
                          port:
                            anyOf:
                            - type: integer
                            - type: string
                            description: Port defines the port of a Kubernetes Service.
                              This can be a reference to a named port.
                            x-kubernetes-int-or-string: true
                          proxyProtocol:
                            description: 'ProxyProtocol defines the PROXY protocol
                              configuration. More info: https://doc.traefik.io/traefik/v2.9/routing/services/#proxy-protocol'
                            properties:
                              version:
                                description: Version defines the PROXY Protocol version
                                  to use.
                                type: integer
                            type: object
                          terminationDelay:
                            description: TerminationDelay defines the deadline that
                              the proxy sets, after one of its connected peers indicates
                              it has closed the writing capability of its connection,
                              to close the reading capability as well, hence fully
                              terminating the connection. It is a duration in milliseconds,
                              defaulting to 100. A negative value means an infinite
                              deadline (i.e. the reading capability is never closed).
                            type: integer
                          weight:
                            description: Weight defines the weight used when balancing
                              requests between multiple Kubernetes Service.
                            type: integer
                        required:
                        - name
                        - port
                        type: object
                      type: array
                  required:
                  - match
                  type: object
                type: array
              tls:
                description: 'TLS defines the TLS configuration on a layer 4 / TCP
                  Route. More info: https://doc.traefik.io/traefik/v2.9/routing/routers/#tls_1'
                properties:
                  certResolver:
                    description: 'CertResolver defines the name of the certificate
                      resolver to use. Cert resolvers have to be configured in the
                      static configuration. More info: https://doc.traefik.io/traefik/v2.9/https/acme/#certificate-resolvers'
                    type: string
                  domains:
                    description: 'Domains defines the list of domains that will be
                      used to issue certificates. More info: https://doc.traefik.io/traefik/v2.9/routing/routers/#domains'
                    items:
                      description: Domain holds a domain name with SANs.
                      properties:
                        main:
                          description: Main defines the main domain name.
                          type: string
                        sans:
                          description: SANs defines the subject alternative domain
                            names.
                          items:
                            type: string
                          type: array
                      type: object
                    type: array
                  options:
                    description: 'Options defines the reference to a TLSOption, that
                      specifies the parameters of the TLS connection. If not defined,
                      the `default` TLSOption is used. More info: https://doc.traefik.io/traefik/v2.9/https/tls/#tls-options'
                    properties:
                      name:
                        description: Name defines the name of the referenced Traefik
                          resource.
                        type: string
                      namespace:
                        description: Namespace defines the namespace of the referenced
                          Traefik resource.
                        type: string
                    required:
                    - name
                    type: object
                  passthrough:
                    description: Passthrough defines whether a TLS router will terminate
                      the TLS connection.
                    type: boolean
                  secretName:
                    description: SecretName is the name of the referenced Kubernetes
                      Secret to specify the certificate details.
                    type: string
                  store:
                    description: Store defines the reference to the TLSStore, that
                      will be used to store certificates. Please note that only `default`
                      TLSStore can be used.
                    properties:
                      name:
                        description: Name defines the name of the referenced Traefik
                          resource.
                        type: string
                      namespace:
                        description: Namespace defines the namespace of the referenced
                          Traefik resource.
                        type: string
                    required:
                    - name
                    type: object
                type: object
            required:
            - routes
            type: object
        required:
        - metadata
        - spec
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []

---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.6.2
  creationTimestamp: null
  name: ingressrouteudps.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: IngressRouteUDP
    listKind: IngressRouteUDPList
    plural: ingressrouteudps
    singular: ingressrouteudp
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: IngressRouteUDP is a CRD implementation of a Traefik UDP Router.
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: IngressRouteUDPSpec defines the desired state of a IngressRouteUDP.
            properties:
              entryPoints:
                description: 'EntryPoints defines the list of entry point names to
                  bind to. Entry points have to be configured in the static configuration.
                  More info: https://doc.traefik.io/traefik/v2.9/routing/entrypoints/
                  Default: all.'
                items:
                  type: string
                type: array
              routes:
                description: Routes defines the list of routes.
                items:
                  description: RouteUDP holds the UDP route configuration.
                  properties:
                    services:
                      description: Services defines the list of UDP services.
                      items:
                        description: ServiceUDP defines an upstream UDP service to
                          proxy traffic to.
                        properties:
                          name:
                            description: Name defines the name of the referenced Kubernetes
                              Service.
                            type: string
                          namespace:
                            description: Namespace defines the namespace of the referenced
                              Kubernetes Service.
                            type: string
                          port:
                            anyOf:
                            - type: integer
                            - type: string
                            description: Port defines the port of a Kubernetes Service.
                              This can be a reference to a named port.
                            x-kubernetes-int-or-string: true
                          weight:
                            description: Weight defines the weight used when balancing
                              requests between multiple Kubernetes Service.
                            type: integer
                        required:
                        - name
                        - port
                        type: object
                      type: array
                  type: object
                type: array
            required:
            - routes
            type: object
        required:
        - metadata
        - spec
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []

---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.6.2
  creationTimestamp: null
  name: middlewares.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: Middleware
    listKind: MiddlewareList
    plural: middlewares
    singular: middleware
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: 'Middleware is the CRD implementation of a Traefik Middleware.
          More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/overview/'
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: MiddlewareSpec defines the desired state of a Middleware.
            properties:
              addPrefix:
                description: 'AddPrefix holds the add prefix middleware configuration.
                  This middleware updates the path of a request before forwarding
                  it. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/addprefix/'
                properties:
                  prefix:
                    description: Prefix is the string to add before the current path
                      in the requested URL. It should include a leading slash (/).
                    type: string
                type: object
              basicAuth:
                description: 'BasicAuth holds the basic auth middleware configuration.
                  This middleware restricts access to your services to known users.
                  More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/basicauth/'
                properties:
                  headerField:
                    description: 'HeaderField defines a header field to store the
                      authenticated user. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/basicauth/#headerfield'
                    type: string
                  realm:
                    description: 'Realm allows the protected resources on a server
                      to be partitioned into a set of protection spaces, each with
                      its own authentication scheme. Default: traefik.'
                    type: string
                  removeHeader:
                    description: 'RemoveHeader sets the removeHeader option to true
                      to remove the authorization header before forwarding the request
                      to your service. Default: false.'
                    type: boolean
                  secret:
                    description: Secret is the name of the referenced Kubernetes Secret
                      containing user credentials.
                    type: string
                type: object
              buffering:
                description: 'Buffering holds the buffering middleware configuration.
                  This middleware retries or limits the size of requests that can
                  be forwarded to backends. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/buffering/#maxrequestbodybytes'
                properties:
                  maxRequestBodyBytes:
                    description: 'MaxRequestBodyBytes defines the maximum allowed
                      body size for the request (in bytes). If the request exceeds
                      the allowed size, it is not forwarded to the service, and the
                      client gets a 413 (Request Entity Too Large) response. Default:
                      0 (no maximum).'
                    format: int64
                    type: integer
                  maxResponseBodyBytes:
                    description: 'MaxResponseBodyBytes defines the maximum allowed
                      response size from the service (in bytes). If the response exceeds
                      the allowed size, it is not forwarded to the client. The client
                      gets a 500 (Internal Server Error) response instead. Default:
                      0 (no maximum).'
                    format: int64
                    type: integer
                  memRequestBodyBytes:
                    description: 'MemRequestBodyBytes defines the threshold (in bytes)
                      from which the request will be buffered on disk instead of in
                      memory. Default: 1048576 (1Mi).'
                    format: int64
                    type: integer
                  memResponseBodyBytes:
                    description: 'MemResponseBodyBytes defines the threshold (in bytes)
                      from which the response will be buffered on disk instead of
                      in memory. Default: 1048576 (1Mi).'
                    format: int64
                    type: integer
                  retryExpression:
                    description: 'RetryExpression defines the retry conditions. It
                      is a logical combination of functions with operators AND (&&)
                      and OR (||). More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/buffering/#retryexpression'
                    type: string
                type: object
              chain:
                description: 'Chain holds the configuration of the chain middleware.
                  This middleware enables to define reusable combinations of other
                  pieces of middleware. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/chain/'
                properties:
                  middlewares:
                    description: Middlewares is the list of MiddlewareRef which composes
                      the chain.
                    items:
                      description: MiddlewareRef is a reference to a Middleware resource.
                      properties:
                        name:
                          description: Name defines the name of the referenced Middleware
                            resource.
                          type: string
                        namespace:
                          description: Namespace defines the namespace of the referenced
                            Middleware resource.
                          type: string
                      required:
                      - name
                      type: object
                    type: array
                type: object
              circuitBreaker:
                description: CircuitBreaker holds the circuit breaker configuration.
                properties:
                  checkPeriod:
                    anyOf:
                    - type: integer
                    - type: string
                    description: CheckPeriod is the interval between successive checks
                      of the circuit breaker condition (when in standby state).
                    x-kubernetes-int-or-string: true
                  expression:
                    description: Expression is the condition that triggers the tripped
                      state.
                    type: string
                  fallbackDuration:
                    anyOf:
                    - type: integer
                    - type: string
                    description: FallbackDuration is the duration for which the circuit
                      breaker will wait before trying to recover (from a tripped state).
                    x-kubernetes-int-or-string: true
                  recoveryDuration:
                    anyOf:
                    - type: integer
                    - type: string
                    description: RecoveryDuration is the duration for which the circuit
                      breaker will try to recover (as soon as it is in recovering
                      state).
                    x-kubernetes-int-or-string: true
                type: object
              compress:
                description: 'Compress holds the compress middleware configuration.
                  This middleware compresses responses before sending them to the
                  client, using gzip compression. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/compress/'
                properties:
                  excludedContentTypes:
                    description: ExcludedContentTypes defines the list of content
                      types to compare the Content-Type header of the incoming requests
                      and responses before compressing.
                    items:
                      type: string
                    type: array
                  minResponseBodyBytes:
                    description: 'MinResponseBodyBytes defines the minimum amount
                      of bytes a response body must have to be compressed. Default:
                      1024.'
                    type: integer
                type: object
              contentType:
                description: ContentType holds the content-type middleware configuration.
                  This middleware exists to enable the correct behavior until at least
                  the default one can be changed in a future version.
                properties:
                  autoDetect:
                    description: AutoDetect specifies whether to let the `Content-Type`
                      header, if it has not been set by the backend, be automatically
                      set to a value derived from the contents of the response. As
                      a proxy, the default behavior should be to leave the header
                      alone, regardless of what the backend did with it. However,
                      the historic default was to always auto-detect and set the header
                      if it was nil, and it is going to be kept that way in order
                      to support users currently relying on it.
                    type: boolean
                type: object
              digestAuth:
                description: 'DigestAuth holds the digest auth middleware configuration.
                  This middleware restricts access to your services to known users.
                  More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/digestauth/'
                properties:
                  headerField:
                    description: 'HeaderField defines a header field to store the
                      authenticated user. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/basicauth/#headerfield'
                    type: string
                  realm:
                    description: 'Realm allows the protected resources on a server
                      to be partitioned into a set of protection spaces, each with
                      its own authentication scheme. Default: traefik.'
                    type: string
                  removeHeader:
                    description: RemoveHeader defines whether to remove the authorization
                      header before forwarding the request to the backend.
                    type: boolean
                  secret:
                    description: Secret is the name of the referenced Kubernetes Secret
                      containing user credentials.
                    type: string
                type: object
              errors:
                description: 'ErrorPage holds the custom error middleware configuration.
                  This middleware returns a custom page in lieu of the default, according
                  to configured ranges of HTTP Status codes. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/errorpages/'
                properties:
                  query:
                    description: Query defines the URL for the error page (hosted
                      by service). The {status} variable can be used in order to insert
                      the status code in the URL.
                    type: string
                  service:
                    description: 'Service defines the reference to a Kubernetes Service
                      that will serve the error page. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/errorpages/#service'
                    properties:
                      kind:
                        description: Kind defines the kind of the Service.
                        enum:
                        - Service
                        - TraefikService
                        type: string
                      name:
                        description: Name defines the name of the referenced Kubernetes
                          Service or TraefikService. The differentiation between the
                          two is specified in the Kind field.
                        type: string
                      namespace:
                        description: Namespace defines the namespace of the referenced
                          Kubernetes Service or TraefikService.
                        type: string
                      passHostHeader:
                        description: PassHostHeader defines whether the client Host
                          header is forwarded to the upstream Kubernetes Service.
                          By default, passHostHeader is true.
                        type: boolean
                      port:
                        anyOf:
                        - type: integer
                        - type: string
                        description: Port defines the port of a Kubernetes Service.
                          This can be a reference to a named port.
                        x-kubernetes-int-or-string: true
                      responseForwarding:
                        description: ResponseForwarding defines how Traefik forwards
                          the response from the upstream Kubernetes Service to the
                          client.
                        properties:
                          flushInterval:
                            description: 'FlushInterval defines the interval, in milliseconds,
                              in between flushes to the client while copying the response
                              body. A negative value means to flush immediately after
                              each write to the client. This configuration is ignored
                              when ReverseProxy recognizes a response as a streaming
                              response; for such responses, writes are flushed to
                              the client immediately. Default: 100ms'
                            type: string
                        type: object
                      scheme:
                        description: Scheme defines the scheme to use for the request
                          to the upstream Kubernetes Service. It defaults to https
                          when Kubernetes Service port is 443, http otherwise.
                        type: string
                      serversTransport:
                        description: ServersTransport defines the name of ServersTransport
                          resource to use. It allows to configure the transport between
                          Traefik and your servers. Can only be used on a Kubernetes
                          Service.
                        type: string
                      sticky:
                        description: 'Sticky defines the sticky sessions configuration.
                          More info: https://doc.traefik.io/traefik/v2.9/routing/services/#sticky-sessions'
                        properties:
                          cookie:
                            description: Cookie defines the sticky cookie configuration.
                            properties:
                              httpOnly:
                                description: HTTPOnly defines whether the cookie can
                                  be accessed by client-side APIs, such as JavaScript.
                                type: boolean
                              name:
                                description: Name defines the Cookie name.
                                type: string
                              sameSite:
                                description: 'SameSite defines the same site policy.
                                  More info: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite'
                                type: string
                              secure:
                                description: Secure defines whether the cookie can
                                  only be transmitted over an encrypted connection
                                  (i.e. HTTPS).
                                type: boolean
                            type: object
                        type: object
                      strategy:
                        description: Strategy defines the load balancing strategy
                          between the servers. RoundRobin is the only supported value
                          at the moment.
                        type: string
                      weight:
                        description: Weight defines the weight and should only be
                          specified when Name references a TraefikService object (and
                          to be precise, one that embeds a Weighted Round Robin).
                        type: integer
                    required:
                    - name
                    type: object
                  status:
                    description: Status defines which status or range of statuses
                      should result in an error page. It can be either a status code
                      as a number (500), as multiple comma-separated numbers (500,502),
                      as ranges by separating two codes with a dash (500-599), or
                      a combination of the two (404,418,500-599).
                    items:
                      type: string
                    type: array
                type: object
              forwardAuth:
                description: 'ForwardAuth holds the forward auth middleware configuration.
                  This middleware delegates the request authentication to a Service.
                  More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/forwardauth/'
                properties:
                  address:
                    description: Address defines the authentication server address.
                    type: string
                  authRequestHeaders:
                    description: AuthRequestHeaders defines the list of the headers
                      to copy from the request to the authentication server. If not
                      set or empty then all request headers are passed.
                    items:
                      type: string
                    type: array
                  authResponseHeaders:
                    description: AuthResponseHeaders defines the list of headers to
                      copy from the authentication server response and set on forwarded
                      request, replacing any existing conflicting headers.
                    items:
                      type: string
                    type: array
                  authResponseHeadersRegex:
                    description: 'AuthResponseHeadersRegex defines the regex to match
                      headers to copy from the authentication server response and
                      set on forwarded request, after stripping all headers that match
                      the regex. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/forwardauth/#authresponseheadersregex'
                    type: string
                  tls:
                    description: TLS defines the configuration used to secure the
                      connection to the authentication server.
                    properties:
                      caOptional:
                        type: boolean
                      caSecret:
                        description: CASecret is the name of the referenced Kubernetes
                          Secret containing the CA to validate the server certificate.
                          The CA certificate is extracted from key `tls.ca` or `ca.crt`.
                        type: string
                      certSecret:
                        description: CertSecret is the name of the referenced Kubernetes
                          Secret containing the client certificate. The client certificate
                          is extracted from the keys `tls.crt` and `tls.key`.
                        type: string
                      insecureSkipVerify:
                        description: InsecureSkipVerify defines whether the server
                          certificates should be validated.
                        type: boolean
                    type: object
                  trustForwardHeader:
                    description: 'TrustForwardHeader defines whether to trust (ie:
                      forward) all X-Forwarded-* headers.'
                    type: boolean
                type: object
              headers:
                description: 'Headers holds the headers middleware configuration.
                  This middleware manages the requests and responses headers. More
                  info: https://doc.traefik.io/traefik/v2.9/middlewares/http/headers/#customrequestheaders'
                properties:
                  accessControlAllowCredentials:
                    description: AccessControlAllowCredentials defines whether the
                      request can include user credentials.
                    type: boolean
                  accessControlAllowHeaders:
                    description: AccessControlAllowHeaders defines the Access-Control-Request-Headers
                      values sent in preflight response.
                    items:
                      type: string
                    type: array
                  accessControlAllowMethods:
                    description: AccessControlAllowMethods defines the Access-Control-Request-Method
                      values sent in preflight response.
                    items:
                      type: string
                    type: array
                  accessControlAllowOriginList:
                    description: AccessControlAllowOriginList is a list of allowable
                      origins. Can also be a wildcard origin "*".
                    items:
                      type: string
                    type: array
                  accessControlAllowOriginListRegex:
                    description: AccessControlAllowOriginListRegex is a list of allowable
                      origins written following the Regular Expression syntax (https://golang.org/pkg/regexp/).
                    items:
                      type: string
                    type: array
                  accessControlExposeHeaders:
                    description: AccessControlExposeHeaders defines the Access-Control-Expose-Headers
                      values sent in preflight response.
                    items:
                      type: string
                    type: array
                  accessControlMaxAge:
                    description: AccessControlMaxAge defines the time that a preflight
                      request may be cached.
                    format: int64
                    type: integer
                  addVaryHeader:
                    description: AddVaryHeader defines whether the Vary header is
                      automatically added/updated when the AccessControlAllowOriginList
                      is set.
                    type: boolean
                  allowedHosts:
                    description: AllowedHosts defines the fully qualified list of
                      allowed domain names.
                    items:
                      type: string
                    type: array
                  browserXssFilter:
                    description: BrowserXSSFilter defines whether to add the X-XSS-Protection
                      header with the value 1; mode=block.
                    type: boolean
                  contentSecurityPolicy:
                    description: ContentSecurityPolicy defines the Content-Security-Policy
                      header value.
                    type: string
                  contentTypeNosniff:
                    description: ContentTypeNosniff defines whether to add the X-Content-Type-Options
                      header with the nosniff value.
                    type: boolean
                  customBrowserXSSValue:
                    description: CustomBrowserXSSValue defines the X-XSS-Protection
                      header value. This overrides the BrowserXssFilter option.
                    type: string
                  customFrameOptionsValue:
                    description: CustomFrameOptionsValue defines the X-Frame-Options
                      header value. This overrides the FrameDeny option.
                    type: string
                  customRequestHeaders:
                    additionalProperties:
                      type: string
                    description: CustomRequestHeaders defines the header names and
                      values to apply to the request.
                    type: object
                  customResponseHeaders:
                    additionalProperties:
                      type: string
                    description: CustomResponseHeaders defines the header names and
                      values to apply to the response.
                    type: object
                  featurePolicy:
                    description: 'Deprecated: use PermissionsPolicy instead.'
                    type: string
                  forceSTSHeader:
                    description: ForceSTSHeader defines whether to add the STS header
                      even when the connection is HTTP.
                    type: boolean
                  frameDeny:
                    description: FrameDeny defines whether to add the X-Frame-Options
                      header with the DENY value.
                    type: boolean
                  hostsProxyHeaders:
                    description: HostsProxyHeaders defines the header keys that may
                      hold a proxied hostname value for the request.
                    items:
                      type: string
                    type: array
                  isDevelopment:
                    description: IsDevelopment defines whether to mitigate the unwanted
                      effects of the AllowedHosts, SSL, and STS options when developing.
                      Usually testing takes place using HTTP, not HTTPS, and on localhost,
                      not your production domain. If you would like your development
                      environment to mimic production with complete Host blocking,
                      SSL redirects, and STS headers, leave this as false.
                    type: boolean
                  permissionsPolicy:
                    description: PermissionsPolicy defines the Permissions-Policy
                      header value. This allows sites to control browser features.
                    type: string
                  publicKey:
                    description: PublicKey is the public key that implements HPKP
                      to prevent MITM attacks with forged certificates.
                    type: string
                  referrerPolicy:
                    description: ReferrerPolicy defines the Referrer-Policy header
                      value. This allows sites to control whether browsers forward
                      the Referer header to other sites.
                    type: string
                  sslForceHost:
                    description: 'Deprecated: use RedirectRegex instead.'
                    type: boolean
                  sslHost:
                    description: 'Deprecated: use RedirectRegex instead.'
                    type: string
                  sslProxyHeaders:
                    additionalProperties:
                      type: string
                    description: 'SSLProxyHeaders defines the header keys with associated
                      values that would indicate a valid HTTPS request. It can be
                      useful when using other proxies (example: "X-Forwarded-Proto":
                      "https").'
                    type: object
                  sslRedirect:
                    description: 'Deprecated: use EntryPoint redirection or RedirectScheme
                      instead.'
                    type: boolean
                  sslTemporaryRedirect:
                    description: 'Deprecated: use EntryPoint redirection or RedirectScheme
                      instead.'
                    type: boolean
                  stsIncludeSubdomains:
                    description: STSIncludeSubdomains defines whether the includeSubDomains
                      directive is appended to the Strict-Transport-Security header.
                    type: boolean
                  stsPreload:
                    description: STSPreload defines whether the preload flag is appended
                      to the Strict-Transport-Security header.
                    type: boolean
                  stsSeconds:
                    description: STSSeconds defines the max-age of the Strict-Transport-Security
                      header. If set to 0, the header is not set.
                    format: int64
                    type: integer
                type: object
              inFlightReq:
                description: 'InFlightReq holds the in-flight request middleware configuration.
                  This middleware limits the number of requests being processed and
                  served concurrently. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/inflightreq/'
                properties:
                  amount:
                    description: Amount defines the maximum amount of allowed simultaneous
                      in-flight request. The middleware responds with HTTP 429 Too
                      Many Requests if there are already amount requests in progress
                      (based on the same sourceCriterion strategy).
                    format: int64
                    type: integer
                  sourceCriterion:
                    description: 'SourceCriterion defines what criterion is used to
                      group requests as originating from a common source. If several
                      strategies are defined at the same time, an error will be raised.
                      If none are set, the default is to use the requestHost. More
                      info: https://doc.traefik.io/traefik/v2.9/middlewares/http/inflightreq/#sourcecriterion'
                    properties:
                      ipStrategy:
                        description: 'IPStrategy holds the IP strategy configuration
                          used by Traefik to determine the client IP. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/ipwhitelist/#ipstrategy'
                        properties:
                          depth:
                            description: Depth tells Traefik to use the X-Forwarded-For
                              header and take the IP located at the depth position
                              (starting from the right).
                            type: integer
                          excludedIPs:
                            description: ExcludedIPs configures Traefik to scan the
                              X-Forwarded-For header and select the first IP not in
                              the list.
                            items:
                              type: string
                            type: array
                        type: object
                      requestHeaderName:
                        description: RequestHeaderName defines the name of the header
                          used to group incoming requests.
                        type: string
                      requestHost:
                        description: RequestHost defines whether to consider the request
                          Host as the source.
                        type: boolean
                    type: object
                type: object
              ipWhiteList:
                description: 'IPWhiteList holds the IP whitelist middleware configuration.
                  This middleware accepts / refuses requests based on the client IP.
                  More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/ipwhitelist/'
                properties:
                  ipStrategy:
                    description: 'IPStrategy holds the IP strategy configuration used
                      by Traefik to determine the client IP. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/ipwhitelist/#ipstrategy'
                    properties:
                      depth:
                        description: Depth tells Traefik to use the X-Forwarded-For
                          header and take the IP located at the depth position (starting
                          from the right).
                        type: integer
                      excludedIPs:
                        description: ExcludedIPs configures Traefik to scan the X-Forwarded-For
                          header and select the first IP not in the list.
                        items:
                          type: string
                        type: array
                    type: object
                  sourceRange:
                    description: SourceRange defines the set of allowed IPs (or ranges
                      of allowed IPs by using CIDR notation).
                    items:
                      type: string
                    type: array
                type: object
              passTLSClientCert:
                description: 'PassTLSClientCert holds the pass TLS client cert middleware
                  configuration. This middleware adds the selected data from the passed
                  client TLS certificate to a header. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/passtlsclientcert/'
                properties:
                  info:
                    description: Info selects the specific client certificate details
                      you want to add to the X-Forwarded-Tls-Client-Cert-Info header.
                    properties:
                      issuer:
                        description: Issuer defines the client certificate issuer
                          details to add to the X-Forwarded-Tls-Client-Cert-Info header.
                        properties:
                          commonName:
                            description: CommonName defines whether to add the organizationalUnit
                              information into the issuer.
                            type: boolean
                          country:
                            description: Country defines whether to add the country
                              information into the issuer.
                            type: boolean
                          domainComponent:
                            description: DomainComponent defines whether to add the
                              domainComponent information into the issuer.
                            type: boolean
                          locality:
                            description: Locality defines whether to add the locality
                              information into the issuer.
                            type: boolean
                          organization:
                            description: Organization defines whether to add the organization
                              information into the issuer.
                            type: boolean
                          province:
                            description: Province defines whether to add the province
                              information into the issuer.
                            type: boolean
                          serialNumber:
                            description: SerialNumber defines whether to add the serialNumber
                              information into the issuer.
                            type: boolean
                        type: object
                      notAfter:
                        description: NotAfter defines whether to add the Not After
                          information from the Validity part.
                        type: boolean
                      notBefore:
                        description: NotBefore defines whether to add the Not Before
                          information from the Validity part.
                        type: boolean
                      sans:
                        description: Sans defines whether to add the Subject Alternative
                          Name information from the Subject Alternative Name part.
                        type: boolean
                      serialNumber:
                        description: SerialNumber defines whether to add the client
                          serialNumber information.
                        type: boolean
                      subject:
                        description: Subject defines the client certificate subject
                          details to add to the X-Forwarded-Tls-Client-Cert-Info header.
                        properties:
                          commonName:
                            description: CommonName defines whether to add the organizationalUnit
                              information into the subject.
                            type: boolean
                          country:
                            description: Country defines whether to add the country
                              information into the subject.
                            type: boolean
                          domainComponent:
                            description: DomainComponent defines whether to add the
                              domainComponent information into the subject.
                            type: boolean
                          locality:
                            description: Locality defines whether to add the locality
                              information into the subject.
                            type: boolean
                          organization:
                            description: Organization defines whether to add the organization
                              information into the subject.
                            type: boolean
                          organizationalUnit:
                            description: OrganizationalUnit defines whether to add
                              the organizationalUnit information into the subject.
                            type: boolean
                          province:
                            description: Province defines whether to add the province
                              information into the subject.
                            type: boolean
                          serialNumber:
                            description: SerialNumber defines whether to add the serialNumber
                              information into the subject.
                            type: boolean
                        type: object
                    type: object
                  pem:
                    description: PEM sets the X-Forwarded-Tls-Client-Cert header with
                      the certificate.
                    type: boolean
                type: object
              plugin:
                additionalProperties:
                  x-kubernetes-preserve-unknown-fields: true
                description: 'Plugin defines the middleware plugin configuration.
                  More info: https://doc.traefik.io/traefik/plugins/'
                type: object
              rateLimit:
                description: 'RateLimit holds the rate limit configuration. This middleware
                  ensures that services will receive a fair amount of requests, and
                  allows one to define what fair is. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/ratelimit/'
                properties:
                  average:
                    description: Average is the maximum rate, by default in requests/s,
                      allowed for the given source. It defaults to 0, which means
                      no rate limiting. The rate is actually defined by dividing Average
                      by Period. So for a rate below 1req/s, one needs to define a
                      Period larger than a second.
                    format: int64
                    type: integer
                  burst:
                    description: Burst is the maximum number of requests allowed to
                      arrive in the same arbitrarily small period of time. It defaults
                      to 1.
                    format: int64
                    type: integer
                  period:
                    anyOf:
                    - type: integer
                    - type: string
                    description: 'Period, in combination with Average, defines the
                      actual maximum rate, such as: r = Average / Period. It defaults
                      to a second.'
                    x-kubernetes-int-or-string: true
                  sourceCriterion:
                    description: SourceCriterion defines what criterion is used to
                      group requests as originating from a common source. If several
                      strategies are defined at the same time, an error will be raised.
                      If none are set, the default is to use the request's remote
                      address field (as an ipStrategy).
                    properties:
                      ipStrategy:
                        description: 'IPStrategy holds the IP strategy configuration
                          used by Traefik to determine the client IP. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/ipwhitelist/#ipstrategy'
                        properties:
                          depth:
                            description: Depth tells Traefik to use the X-Forwarded-For
                              header and take the IP located at the depth position
                              (starting from the right).
                            type: integer
                          excludedIPs:
                            description: ExcludedIPs configures Traefik to scan the
                              X-Forwarded-For header and select the first IP not in
                              the list.
                            items:
                              type: string
                            type: array
                        type: object
                      requestHeaderName:
                        description: RequestHeaderName defines the name of the header
                          used to group incoming requests.
                        type: string
                      requestHost:
                        description: RequestHost defines whether to consider the request
                          Host as the source.
                        type: boolean
                    type: object
                type: object
              redirectRegex:
                description: 'RedirectRegex holds the redirect regex middleware configuration.
                  This middleware redirects a request using regex matching and replacement.
                  More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/redirectregex/#regex'
                properties:
                  permanent:
                    description: Permanent defines whether the redirection is permanent
                      (301).
                    type: boolean
                  regex:
                    description: Regex defines the regex used to match and capture
                      elements from the request URL.
                    type: string
                  replacement:
                    description: Replacement defines how to modify the URL to have
                      the new target URL.
                    type: string
                type: object
              redirectScheme:
                description: 'RedirectScheme holds the redirect scheme middleware
                  configuration. This middleware redirects requests from a scheme/port
                  to another. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/redirectscheme/'
                properties:
                  permanent:
                    description: Permanent defines whether the redirection is permanent
                      (301).
                    type: boolean
                  port:
                    description: Port defines the port of the new URL.
                    type: string
                  scheme:
                    description: Scheme defines the scheme of the new URL.
                    type: string
                type: object
              replacePath:
                description: 'ReplacePath holds the replace path middleware configuration.
                  This middleware replaces the path of the request URL and store the
                  original path in an X-Replaced-Path header. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/replacepath/'
                properties:
                  path:
                    description: Path defines the path to use as replacement in the
                      request URL.
                    type: string
                type: object
              replacePathRegex:
                description: 'ReplacePathRegex holds the replace path regex middleware
                  configuration. This middleware replaces the path of a URL using
                  regex matching and replacement. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/replacepathregex/'
                properties:
                  regex:
                    description: Regex defines the regular expression used to match
                      and capture the path from the request URL.
                    type: string
                  replacement:
                    description: Replacement defines the replacement path format,
                      which can include captured variables.
                    type: string
                type: object
              retry:
                description: 'Retry holds the retry middleware configuration. This
                  middleware reissues requests a given number of times to a backend
                  server if that server does not reply. As soon as the server answers,
                  the middleware stops retrying, regardless of the response status.
                  More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/retry/'
                properties:
                  attempts:
                    description: Attempts defines how many times the request should
                      be retried.
                    type: integer
                  initialInterval:
                    anyOf:
                    - type: integer
                    - type: string
                    description: InitialInterval defines the first wait time in the
                      exponential backoff series. The maximum interval is calculated
                      as twice the initialInterval. If unspecified, requests will
                      be retried immediately. The value of initialInterval should
                      be provided in seconds or as a valid duration format, see https://pkg.go.dev/time#ParseDuration.
                    x-kubernetes-int-or-string: true
                type: object
              stripPrefix:
                description: 'StripPrefix holds the strip prefix middleware configuration.
                  This middleware removes the specified prefixes from the URL path.
                  More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/stripprefix/'
                properties:
                  forceSlash:
                    description: 'ForceSlash ensures that the resulting stripped path
                      is not the empty string, by replacing it with / when necessary.
                      Default: true.'
                    type: boolean
                  prefixes:
                    description: Prefixes defines the prefixes to strip from the request
                      URL.
                    items:
                      type: string
                    type: array
                type: object
              stripPrefixRegex:
                description: 'StripPrefixRegex holds the strip prefix regex middleware
                  configuration. This middleware removes the matching prefixes from
                  the URL path. More info: https://doc.traefik.io/traefik/v2.9/middlewares/http/stripprefixregex/'
                properties:
                  regex:
                    description: Regex defines the regular expression to match the
                      path prefix from the request URL.
                    items:
                      type: string
                    type: array
                type: object
            type: object
        required:
        - metadata
        - spec
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []

---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.6.2
  creationTimestamp: null
  name: middlewaretcps.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: MiddlewareTCP
    listKind: MiddlewareTCPList
    plural: middlewaretcps
    singular: middlewaretcp
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: 'MiddlewareTCP is the CRD implementation of a Traefik TCP middleware.
          More info: https://doc.traefik.io/traefik/v2.9/middlewares/overview/'
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: MiddlewareTCPSpec defines the desired state of a MiddlewareTCP.
            properties:
              inFlightConn:
                description: InFlightConn defines the InFlightConn middleware configuration.
                properties:
                  amount:
                    description: Amount defines the maximum amount of allowed simultaneous
                      connections. The middleware closes the connection if there are
                      already amount connections opened.
                    format: int64
                    type: integer
                type: object
              ipWhiteList:
                description: IPWhiteList defines the IPWhiteList middleware configuration.
                properties:
                  sourceRange:
                    description: SourceRange defines the allowed IPs (or ranges of
                      allowed IPs by using CIDR notation).
                    items:
                      type: string
                    type: array
                type: object
            type: object
        required:
        - metadata
        - spec
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []

---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.6.2
  creationTimestamp: null
  name: serverstransports.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: ServersTransport
    listKind: ServersTransportList
    plural: serverstransports
    singular: serverstransport
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: 'ServersTransport is the CRD implementation of a ServersTransport.
          If no serversTransport is specified, the default@internal will be used.
          The default@internal serversTransport is created from the static configuration.
          More info: https://doc.traefik.io/traefik/v2.9/routing/services/#serverstransport_1'
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: ServersTransportSpec defines the desired state of a ServersTransport.
            properties:
              certificatesSecrets:
                description: CertificatesSecrets defines a list of secret storing
                  client certificates for mTLS.
                items:
                  type: string
                type: array
              disableHTTP2:
                description: DisableHTTP2 disables HTTP/2 for connections with backend
                  servers.
                type: boolean
              forwardingTimeouts:
                description: ForwardingTimeouts defines the timeouts for requests
                  forwarded to the backend servers.
                properties:
                  dialTimeout:
                    anyOf:
                    - type: integer
                    - type: string
                    description: DialTimeout is the amount of time to wait until a
                      connection to a backend server can be established.
                    x-kubernetes-int-or-string: true
                  idleConnTimeout:
                    anyOf:
                    - type: integer
                    - type: string
                    description: IdleConnTimeout is the maximum period for which an
                      idle HTTP keep-alive connection will remain open before closing
                      itself.
                    x-kubernetes-int-or-string: true
                  pingTimeout:
                    anyOf:
                    - type: integer
                    - type: string
                    description: PingTimeout is the timeout after which the HTTP/2
                      connection will be closed if a response to ping is not received.
                    x-kubernetes-int-or-string: true
                  readIdleTimeout:
                    anyOf:
                    - type: integer
                    - type: string
                    description: ReadIdleTimeout is the timeout after which a health
                      check using ping frame will be carried out if no frame is received
                      on the HTTP/2 connection.
                    x-kubernetes-int-or-string: true
                  responseHeaderTimeout:
                    anyOf:
                    - type: integer
                    - type: string
                    description: ResponseHeaderTimeout is the amount of time to wait
                      for a server's response headers after fully writing the request
                      (including its body, if any).
                    x-kubernetes-int-or-string: true
                type: object
              insecureSkipVerify:
                description: InsecureSkipVerify disables SSL certificate verification.
                type: boolean
              maxIdleConnsPerHost:
                description: MaxIdleConnsPerHost controls the maximum idle (keep-alive)
                  to keep per-host.
                type: integer
              peerCertURI:
                description: PeerCertURI defines the peer cert URI used to match against
                  SAN URI during the peer certificate verification.
                type: string
              rootCAsSecrets:
                description: RootCAsSecrets defines a list of CA secret used to validate
                  self-signed certificate.
                items:
                  type: string
                type: array
              serverName:
                description: ServerName defines the server name used to contact the
                  server.
                type: string
            type: object
        required:
        - metadata
        - spec
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []

---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.6.2
  creationTimestamp: null
  name: tlsoptions.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: TLSOption
    listKind: TLSOptionList
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: 'TLSOption is the CRD implementation of a Traefik TLS Option,
          allowing to configure some parameters of the TLS connection. More info:
          https://doc.traefik.io/traefik/v2.9/https/tls/#tls-options'
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: TLSOptionSpec defines the desired state of a TLSOption.
            properties:
              alpnProtocols:
                description: 'ALPNProtocols defines the list of supported application
                  level protocols for the TLS handshake, in order of preference. More
                  info: https://doc.traefik.io/traefik/v2.9/https/tls/#alpn-protocols'
                items:
                  type: string
                type: array
              cipherSuites:
                description: 'CipherSuites defines the list of supported cipher suites
                  for TLS versions up to TLS 1.2. More info: https://doc.traefik.io/traefik/v2.9/https/tls/#cipher-suites'
                items:
                  type: string
                type: array
              clientAuth:
                description: ClientAuth defines the server's policy for TLS Client
                  Authentication.
                properties:
                  clientAuthType:
                    description: ClientAuthType defines the client authentication
                      type to apply.
                    enum:
                    - NoClientCert
                    - RequestClientCert
                    - RequireAnyClientCert
                    - VerifyClientCertIfGiven
                    - RequireAndVerifyClientCert
                    type: string
                  secretNames:
                    description: SecretNames defines the names of the referenced Kubernetes
                      Secret storing certificate details.
                    items:
                      type: string
                    type: array
                type: object
              curvePreferences:
                description: 'CurvePreferences defines the preferred elliptic curves
                  in a specific order. More info: https://doc.traefik.io/traefik/v2.9/https/tls/#curve-preferences'
                items:
                  type: string
                type: array
              maxVersion:
                description: 'MaxVersion defines the maximum TLS version that Traefik
                  will accept. Possible values: VersionTLS10, VersionTLS11, VersionTLS12,
                  VersionTLS13. Default: None.'
                type: string
              minVersion:
                description: 'MinVersion defines the minimum TLS version that Traefik
                  will accept. Possible values: VersionTLS10, VersionTLS11, VersionTLS12,
                  VersionTLS13. Default: VersionTLS10.'
                type: string
              preferServerCipherSuites:
                description: 'PreferServerCipherSuites defines whether the server
                  chooses a cipher suite among his own instead of among the client''s.
                  It is enabled automatically when minVersion or maxVersion is set.
                  Deprecated: https://github.com/golang/go/issues/45430'
                type: boolean
              sniStrict:
                description: SniStrict defines whether Traefik allows connections
                  from clients connections that do not specify a server_name extension.
                type: boolean
            type: object
        required:
        - metadata
        - spec
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []

---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.6.2
  creationTimestamp: null
  name: tlsstores.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: TLSStore
    listKind: TLSStoreList
    plural: tlsstores
    singular: tlsstore
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: 'TLSStore is the CRD implementation of a Traefik TLS Store. For
          the time being, only the TLSStore named default is supported. This means
          that you cannot have two stores that are named default in different Kubernetes
          namespaces. More info: https://doc.traefik.io/traefik/v2.9/https/tls/#certificates-stores'
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: TLSStoreSpec defines the desired state of a TLSStore.
            properties:
              certificates:
                description: Certificates is a list of secret names, each secret holding
                  a key/certificate pair to add to the store.
                items:
                  description: Certificate holds a secret name for the TLSStore resource.
                  properties:
                    secretName:
                      description: SecretName is the name of the referenced Kubernetes
                        Secret to specify the certificate details.
                      type: string
                  required:
                  - secretName
                  type: object
                type: array
              defaultCertificate:
                description: DefaultCertificate defines the default certificate configuration.
                properties:
                  secretName:
                    description: SecretName is the name of the referenced Kubernetes
                      Secret to specify the certificate details.
                    type: string
                required:
                - secretName
                type: object
              defaultGeneratedCert:
                description: DefaultGeneratedCert defines the default generated certificate
                  configuration.
                properties:
                  domain:
                    description: Domain is the domain definition for the DefaultCertificate.
                    properties:
                      main:
                        description: Main defines the main domain name.
                        type: string
                      sans:
                        description: SANs defines the subject alternative domain names.
                        items:
                          type: string
                        type: array
                    type: object
                  resolver:
                    description: Resolver is the name of the resolver that will be
                      used to issue the DefaultCertificate.
                    type: string
                type: object
            type: object
        required:
        - metadata
        - spec
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []

---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.6.2
  creationTimestamp: null
  name: traefikservices.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: TraefikService
    listKind: TraefikServiceList
    plural: traefikservices
    singular: traefikservice
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: 'TraefikService is the CRD implementation of a Traefik Service.
          TraefikService object allows to: - Apply weight to Services on load-balancing
          - Mirror traffic on services More info: https://doc.traefik.io/traefik/v2.9/routing/providers/kubernetes-crd/#kind-traefikservice'
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: TraefikServiceSpec defines the desired state of a TraefikService.
            properties:
              mirroring:
                description: Mirroring defines the Mirroring service configuration.
                properties:
                  kind:
                    description: Kind defines the kind of the Service.
                    enum:
                    - Service
                    - TraefikService
                    type: string
                  maxBodySize:
                    description: MaxBodySize defines the maximum size allowed for
                      the body of the request. If the body is larger, the request
                      is not mirrored. Default value is -1, which means unlimited
                      size.
                    format: int64
                    type: integer
                  mirrors:
                    description: Mirrors defines the list of mirrors where Traefik
                      will duplicate the traffic.
                    items:
                      description: MirrorService holds the mirror configuration.
                      properties:
                        kind:
                          description: Kind defines the kind of the Service.
                          enum:
                          - Service
                          - TraefikService
                          type: string
                        name:
                          description: Name defines the name of the referenced Kubernetes
                            Service or TraefikService. The differentiation between
                            the two is specified in the Kind field.
                          type: string
                        namespace:
                          description: Namespace defines the namespace of the referenced
                            Kubernetes Service or TraefikService.
                          type: string
                        passHostHeader:
                          description: PassHostHeader defines whether the client Host
                            header is forwarded to the upstream Kubernetes Service.
                            By default, passHostHeader is true.
                          type: boolean
                        percent:
                          description: 'Percent defines the part of the traffic to
                            mirror. Supported values: 0 to 100.'
                          type: integer
                        port:
                          anyOf:
                          - type: integer
                          - type: string
                          description: Port defines the port of a Kubernetes Service.
                            This can be a reference to a named port.
                          x-kubernetes-int-or-string: true
                        responseForwarding:
                          description: ResponseForwarding defines how Traefik forwards
                            the response from the upstream Kubernetes Service to the
                            client.
                          properties:
                            flushInterval:
                              description: 'FlushInterval defines the interval, in
                                milliseconds, in between flushes to the client while
                                copying the response body. A negative value means
                                to flush immediately after each write to the client.
                                This configuration is ignored when ReverseProxy recognizes
                                a response as a streaming response; for such responses,
                                writes are flushed to the client immediately. Default:
                                100ms'
                              type: string
                          type: object
                        scheme:
                          description: Scheme defines the scheme to use for the request
                            to the upstream Kubernetes Service. It defaults to https
                            when Kubernetes Service port is 443, http otherwise.
                          type: string
                        serversTransport:
                          description: ServersTransport defines the name of ServersTransport
                            resource to use. It allows to configure the transport
                            between Traefik and your servers. Can only be used on
                            a Kubernetes Service.
                          type: string
                        sticky:
                          description: 'Sticky defines the sticky sessions configuration.
                            More info: https://doc.traefik.io/traefik/v2.9/routing/services/#sticky-sessions'
                          properties:
                            cookie:
                              description: Cookie defines the sticky cookie configuration.
                              properties:
                                httpOnly:
                                  description: HTTPOnly defines whether the cookie
                                    can be accessed by client-side APIs, such as JavaScript.
                                  type: boolean
                                name:
                                  description: Name defines the Cookie name.
                                  type: string
                                sameSite:
                                  description: 'SameSite defines the same site policy.
                                    More info: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite'
                                  type: string
                                secure:
                                  description: Secure defines whether the cookie can
                                    only be transmitted over an encrypted connection
                                    (i.e. HTTPS).
                                  type: boolean
                              type: object
                          type: object
                        strategy:
                          description: Strategy defines the load balancing strategy
                            between the servers. RoundRobin is the only supported
                            value at the moment.
                          type: string
                        weight:
                          description: Weight defines the weight and should only be
                            specified when Name references a TraefikService object
                            (and to be precise, one that embeds a Weighted Round Robin).
                          type: integer
                      required:
                      - name
                      type: object
                    type: array
                  name:
                    description: Name defines the name of the referenced Kubernetes
                      Service or TraefikService. The differentiation between the two
                      is specified in the Kind field.
                    type: string
                  namespace:
                    description: Namespace defines the namespace of the referenced
                      Kubernetes Service or TraefikService.
                    type: string
                  passHostHeader:
                    description: PassHostHeader defines whether the client Host header
                      is forwarded to the upstream Kubernetes Service. By default,
                      passHostHeader is true.
                    type: boolean
                  port:
                    anyOf:
                    - type: integer
                    - type: string
                    description: Port defines the port of a Kubernetes Service. This
                      can be a reference to a named port.
                    x-kubernetes-int-or-string: true
                  responseForwarding:
                    description: ResponseForwarding defines how Traefik forwards the
                      response from the upstream Kubernetes Service to the client.
                    properties:
                      flushInterval:
                        description: 'FlushInterval defines the interval, in milliseconds,
                          in between flushes to the client while copying the response
                          body. A negative value means to flush immediately after
                          each write to the client. This configuration is ignored
                          when ReverseProxy recognizes a response as a streaming response;
                          for such responses, writes are flushed to the client immediately.
                          Default: 100ms'
                        type: string
                    type: object
                  scheme:
                    description: Scheme defines the scheme to use for the request
                      to the upstream Kubernetes Service. It defaults to https when
                      Kubernetes Service port is 443, http otherwise.
                    type: string
                  serversTransport:
                    description: ServersTransport defines the name of ServersTransport
                      resource to use. It allows to configure the transport between
                      Traefik and your servers. Can only be used on a Kubernetes Service.
                    type: string
                  sticky:
                    description: 'Sticky defines the sticky sessions configuration.
                      More info: https://doc.traefik.io/traefik/v2.9/routing/services/#sticky-sessions'
                    properties:
                      cookie:
                        description: Cookie defines the sticky cookie configuration.
                        properties:
                          httpOnly:
                            description: HTTPOnly defines whether the cookie can be
                              accessed by client-side APIs, such as JavaScript.
                            type: boolean
                          name:
                            description: Name defines the Cookie name.
                            type: string
                          sameSite:
                            description: 'SameSite defines the same site policy. More
                              info: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite'
                            type: string
                          secure:
                            description: Secure defines whether the cookie can only
                              be transmitted over an encrypted connection (i.e. HTTPS).
                            type: boolean
                        type: object
                    type: object
                  strategy:
                    description: Strategy defines the load balancing strategy between
                      the servers. RoundRobin is the only supported value at the moment.
                    type: string
                  weight:
                    description: Weight defines the weight and should only be specified
                      when Name references a TraefikService object (and to be precise,
                      one that embeds a Weighted Round Robin).
                    type: integer
                required:
                - name
                type: object
              weighted:
                description: Weighted defines the Weighted Round Robin configuration.
                properties:
                  services:
                    description: Services defines the list of Kubernetes Service and/or
                      TraefikService to load-balance, with weight.
                    items:
                      description: Service defines an upstream HTTP service to proxy
                        traffic to.
                      properties:
                        kind:
                          description: Kind defines the kind of the Service.
                          enum:
                          - Service
                          - TraefikService
                          type: string
                        name:
                          description: Name defines the name of the referenced Kubernetes
                            Service or TraefikService. The differentiation between
                            the two is specified in the Kind field.
                          type: string
                        namespace:
                          description: Namespace defines the namespace of the referenced
                            Kubernetes Service or TraefikService.
                          type: string
                        passHostHeader:
                          description: PassHostHeader defines whether the client Host
                            header is forwarded to the upstream Kubernetes Service.
                            By default, passHostHeader is true.
                          type: boolean
                        port:
                          anyOf:
                          - type: integer
                          - type: string
                          description: Port defines the port of a Kubernetes Service.
                            This can be a reference to a named port.
                          x-kubernetes-int-or-string: true
                        responseForwarding:
                          description: ResponseForwarding defines how Traefik forwards
                            the response from the upstream Kubernetes Service to the
                            client.
                          properties:
                            flushInterval:
                              description: 'FlushInterval defines the interval, in
                                milliseconds, in between flushes to the client while
                                copying the response body. A negative value means
                                to flush immediately after each write to the client.
                                This configuration is ignored when ReverseProxy recognizes
                                a response as a streaming response; for such responses,
                                writes are flushed to the client immediately. Default:
                                100ms'
                              type: string
                          type: object
                        scheme:
                          description: Scheme defines the scheme to use for the request
                            to the upstream Kubernetes Service. It defaults to https
                            when Kubernetes Service port is 443, http otherwise.
                          type: string
                        serversTransport:
                          description: ServersTransport defines the name of ServersTransport
                            resource to use. It allows to configure the transport
                            between Traefik and your servers. Can only be used on
                            a Kubernetes Service.
                          type: string
                        sticky:
                          description: 'Sticky defines the sticky sessions configuration.
                            More info: https://doc.traefik.io/traefik/v2.9/routing/services/#sticky-sessions'
                          properties:
                            cookie:
                              description: Cookie defines the sticky cookie configuration.
                              properties:
                                httpOnly:
                                  description: HTTPOnly defines whether the cookie
                                    can be accessed by client-side APIs, such as JavaScript.
                                  type: boolean
                                name:
                                  description: Name defines the Cookie name.
                                  type: string
                                sameSite:
                                  description: 'SameSite defines the same site policy.
                                    More info: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite'
                                  type: string
                                secure:
                                  description: Secure defines whether the cookie can
                                    only be transmitted over an encrypted connection
                                    (i.e. HTTPS).
                                  type: boolean
                              type: object
                          type: object
                        strategy:
                          description: Strategy defines the load balancing strategy
                            between the servers. RoundRobin is the only supported
                            value at the moment.
                          type: string
                        weight:
                          description: Weight defines the weight and should only be
                            specified when Name references a TraefikService object
                            (and to be precise, one that embeds a Weighted Round Robin).
                          type: integer
                      required:
                      - name
                      type: object
                    type: array
                  sticky:
                    description: 'Sticky defines whether sticky sessions are enabled.
                      More info: https://doc.traefik.io/traefik/v2.9/routing/providers/kubernetes-crd/#stickiness-and-load-balancing'
                    properties:
                      cookie:
                        description: Cookie defines the sticky cookie configuration.
                        properties:
                          httpOnly:
                            description: HTTPOnly defines whether the cookie can be
                              accessed by client-side APIs, such as JavaScript.
                            type: boolean
                          name:
                            description: Name defines the Cookie name.
                            type: string
                          sameSite:
                            description: 'SameSite defines the same site policy. More
                              info: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite'
                            type: string
                          secure:
                            description: Secure defines whether the cookie can only
                              be transmitted over an encrypted connection (i.e. HTTPS).
                            type: boolean
                        type: object
                    type: object
                type: object
            type: object
        required:
        - metadata
        - spec
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []
```

ConfigMap配置：

```yaml
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
      http:
        address: ":80"          ## 配置 80 端口，并设置入口名称为 web
        forwardedHeaders:       ## 获取客户端真实IP
          insecure: true
          trustedIPs:
          - "10.68.0.0/16"
          - "172.16.0.0/12"
      # https:
      #   address: ":443"         ## 配置 443 端口，并设置入口名称为 https
      #   forwardedHeaders:       ## 获取客户端真实IP
      #     insecure: true
      #     trustedIPs:
      #     - "10.68.0.0/16"
      #     - "172.16.0.0/12"
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
```

应用部署文件配置：

```yaml
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
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      serviceAccountName: traefik-ingress-deploy
      terminationGracePeriodSeconds: 1
      hostNetwork: true
      containers:
      - image: traefik:2.9
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

部署完成之后，可以访问Traefik的控制台：`MasterIP：8080`，更多配置可参考Traefik官网：`https://docs.traefik.io`。

### 17 Dashboard开启TLS访问

针对单个应用开启TLS安全访问的步骤：

- 1.Traefik配置443端口监听
- 2.创建证书Secret
- 3.Ingress上配置证书及使用的`Traefik Entrypoint`

修改Traefik Configmap配置文件中的entryPoints，修改更新Configmap后需要重建Traefik容器以便快速应用新的配置。

```
    entryPoints:
      http:
        address: ":80"          ## 配置 80 端口，并设置入口名称为 http
      https:
        address: ":443"         ## 配置 443 端口，并设置入口名称为 https
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
    traefik.ingress.kubernetes.io/router.entrypoints: https
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
  - https
  routes:
  - match: Host(`console.cloudnil.com`) && PathPrefix(`/`)
    kind: Rule
    services:
    - name: earth-web-svc
      port: 80
  tls:
    certResolver: tls-cert
```

### 18 结语

`Nginx-ingress-controller`或`Traefik-ingress-controller`部署完成之后，解析相关域名如`console.cloudnil.com`到master01、master02、master03的外网IP，就可以使用`console.cloudnil.com`访问dashboard，其他应用类似。

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2023/04/18/Deploy-kubernetes(1.26.4)-HA-with-kubeadm/。