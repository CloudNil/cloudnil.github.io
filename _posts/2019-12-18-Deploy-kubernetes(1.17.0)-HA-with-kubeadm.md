---
layout: post
title:  kubeadm快速部署kubernetes(1.17.0,HA)
categories: [docker,Cloud]
date: 2019-12-18 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>当前版本的kubeadm已经原生支持部署HA模式集群，非常方便即可实现HA模式的kubernetes集群。本次部署基于Ubuntu18.04，并使用最新的docker版本：18.06.3~ce~3-0~ubuntu，kubernetes适用1.17.x版本，本文采用1.17.0。

### 1 环境准备

准备了六台机器作安装测试工作，机器信息如下: 

|      IP      |   Name   |       Role      |      OS     |
|--------------|----------|-----------------|-------------|
| 172.16.2.1   | Master01 | Controller,etcd | Ubuntu18.04 |
| 172.16.2.2   | Master02 | Controller,etcd | Ubuntu18.04 |
| 172.16.2.3   | Master03 | Controller,etcd | Ubuntu18.04 |
| 172.16.2.11  | Node01   | Compute         | Ubuntu18.04 |
| 172.16.2.12  | Node02   | Compute         | Ubuntu18.04 |
| 172.16.2.13  | Node03   | Compute         | Ubuntu18.04 |
| 172.16.2.251 | Dns01    | DNS             | Ubuntu18.04 |
| 172.16.2.252 | Dns01    | DNS             | Ubuntu18.04 |

>注意：需要在/etc/hosts中配置本机的主机名解析，如Master01，添加：172.16.2.1 master01 。

### 2 安装docker

```
apt update && apt install -y apt-transport-https software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

apt update

apt install docker-ce=18.06.3~ce~3-0~ubuntu
```

配置docker使用systemd驱动，相比默认的cgrouops更稳定。

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
systemctl daemon-reload
systemctl restart docker
```

### 3 安装etcd集群

使用了docker-compose安装，当然，如果觉得麻烦，也可以直接docker run。

Master01节点的ETCD的docker-compose.yml：

```yaml
version: "3.7"
services:
  etcd:
    image: quay.io/coreos/etcd:v3.3.13
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
    image: quay.io/coreos/etcd:v3.3.13
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
    image: quay.io/coreos/etcd:v3.3.13
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

创建好docker-compose.yml文件后，使用命令`docker-compose up -d`部署。

>关于docker-compose的使用，可以参考：[docker-compose安装文档](https://docs.docker.com/compose/install/#alternative-install-options)。

### 3 安装k8s工具包

**阿里源安装**

```
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl ipvsadm ipset
apt-mark hold kubelet kubeadm kubectl ipvsadm docker-ce
```

### 4 启用ipvs模块

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

### 5 Api-Server负载均衡

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

### 6 安装master节点

kubeadm配置文件kubeadm-config.yaml：

```yaml
apiVersion: kubeadm.k8s.io/v1beta1
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
kubernetesVersion: v1.17.0
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

>说明：因为gcr.io在墙外，导致镜像无法获取，感谢阿里云提供了镜像仓库：`registry.cn-hangzhou.aliyuncs.com/google_containers`，镜像下载需要点时间，也可以提前下载镜像：`kubeadm config images pull --config kubeadm-config.yaml`。

相关镜像：
```
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.0
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5
```

master01初始化指令：

```
kubeadm init --config=kubeadm-config.yaml --upload-certs
```
如果镜像已经提前下载，安装过程大概30秒，输出结果如下：

```
[init] Using Kubernetes version: v1.17.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [master01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local api.k8s.com api.k8s.com] and IPs [10.96.0.1 172.16.2.1]
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
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W1220 08:00:34.269246    5095 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W1220 08:00:34.270648    5095 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[apiclient] All control plane components are healthy after 42.054178 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.17" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
fbf4fd508065cb4478a6269de8ca7402ceec7640209df73dbdef254e3d02efbd
[mark-control-plane] Marking the node master01 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: obrxsw.2wzhv0z6nfnc4nyr
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
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

### 7 安装calico网络

网络组件选择很多，可以根据自己的需要选择calico、weave、flannel，calico性能最好，flannel的vxlan也不错，默认的UDP性能较差，weave的性能比较差，测试环境用下可以，生产环境不建议使用。calico的安装配置可以参考官方部署：[点击查看](https://docs.projectcalico.org/v3.7/getting-started/kubernetes/installation/calico)

calico-rbac.yml：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-kube-controllers
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-node
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

  # Configure the MTU to use
  veth_mtu: "1440"

  # The CNI network configuration to install on each node.  The special
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
      annotations:
        # This, along with the CriticalAddonsOnly toleration below,
        # marks the pod as a critical add-on, ensuring it gets
        # priority scheduling and that its resources are reserved
        # if it ever gets evicted.
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      nodeSelector:
        beta.kubernetes.io/os: linux
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
          image: calico/cni:v3.10.2
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
        # Adds a Flex Volume Driver that creates a per-pod Unix Domain Socket to allow Dikastes
        # to communicate with Felix over the Policy Sync API.
        - name: flexvol-driver
          image: calico/pod2daemon-flexvol:v3.10.2
          volumeMounts:
          - name: flexvol-driver-host
            mountPath: /host/driver
      containers:
        # Runs calico-node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: calico/node:v3.10.2
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
            # Set MTU for tunnel device used if ipip is enabled
            - name: FELIX_IPINIPMTU
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
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      nodeSelector:
        beta.kubernetes.io/os: linux
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
          image: calico/kube-controllers:v3.10.2
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
master01   Ready    master   13m   v1.17.0
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

### 8 安装Master02和Master03节点

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
master01   Ready    master   25m     v1.17.0
master02   Ready    master   8m6s    v1.17.0
master03   Ready    master   7m33s   v1.17.0
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

### 9 安装Node节点

Master节点安装好了Node节点就简单了,在各个Node节点上执行。

```
kubeadm join api.k8s.com:6443 --token obrxsw.2wzhv0z6nfnc4nyr \
    --discovery-token-ca-cert-hash sha256:312a5bab2f699a4e701fb08ce65218b01aff7521fc962f35ad56e5d3214d3e6f
```

### 10 DNS集群部署

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
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5
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

### 10 部署Metrics-Server

kubernetesv1.11以后不再支持通过`heaspter`采集监控数据，支持新的监控数据采集组件`metrics-server`，比`heaspter`轻量很多，也不做数据的持久化存储，提供实时的监控数据查询还是很好用的。

获取部署文档，[点击这里](https://github.com/kubernetes-incubator/metrics-server/tree/master/deploy/1.8%2B)。

下载所有yaml到目录`metrics-server`，修改`metrics-server-deployment.yaml`为以下内容：

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.3
        imagePullPolicy: Always
        command:
        - /metrics-server
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
```

执行部署命令：

```
kubectl apply -f metrics-server/
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

### 11 部署Dashboard

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
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: k8dash-ing
  namespace: kube-system
  labels:
    k8s-app: k8dash
spec:
  rules:
  - host: console.cloudnil.com
    http:
      paths:
      - path: /
        backend:
          serviceName: k8dash-svc
          servicePort: 80
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

### 12 服务暴露到公网

kubernetes中的Service暴露到外部有三种方式，分别是：

- LoadBlancer Service
- NodePort Service
- Ingress

LoadBlancer Service是kubernetes深度结合云平台的一个组件；当使用LoadBlancer Service暴露服务时，实际上是通过向底层云平台申请创建一个负载均衡器来向外暴露服务；目前LoadBlancer Service支持的云平台已经相对完善，比如国外的GCE、DigitalOcean，国内的 阿里云，私有云 Openstack 等等，由于LoadBlancer Service深度结合了云平台，所以只能在一些云平台上来使用。

NodePort Service顾名思义，实质上就是通过在集群的每个node上暴露一个端口，然后将这个端口映射到某个具体的service来实现的，虽然每个node的端口有很多(0~65535)，但是由于安全性和易用性(服务多了就乱了，还有端口冲突问题)实际使用可能并不多。

Ingress可以实现使用nginx等开源的反向代理负载均衡器实现对外暴露服务，可以理解Ingress就是用于配置域名转发的一个东西，在nginx中就类似upstream，它与ingress-controller结合使用，通过ingress-controller监控到pod及service的变化，动态地将ingress中的转发信息写到诸如nginx、apache、haproxy等组件中实现方向代理和负载均衡。

### 13 部署Nginx-ingress-controller

>说明：Nginx-controller和Traefik-controller二选一。

`Nginx-ingress-controller`是kubernetes官方提供的集成了Ingress-controller和Nginx的一个docker镜像。

本次部署中，将Nginx-ingress部署到`master01、master02、master03`上，监听宿主机的`80`端口:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      hostNetwork: true
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - master01
                - master02
                - master03
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/name
                    operator: In
                    values: 
                    - ingress-nginx
              topologyKey: "kubernetes.io/hostname"
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            # - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources:
            limits:
              cpu: 1
              memory: 1024Mi
            requests:
              cpu: 0.25
              memory: 512Mi
```

更多配置可参考Nginx-ingress-controller官网：`https://kubernetes.github.io/ingress-nginx`。

### 14 部署Traefik-ingress-controller

>说明：Nginx-ingress-controller和Traefik-ingress-controller二选一。

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
  name: traefik-ingress-controller
  namespace: traefik
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
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
    resources:
      - ingresses
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
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: traefik
```

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: traefik-ingress-controller
  namespace: traefik
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 3
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      volumes:
      - name: certs
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-config
      hostNetwork: true
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - master01
                - master02
                - master03
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/name
                    operator: In
                    values: 
                    - traefik-ingress-controller
              topologyKey: "kubernetes.io/hostname"
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik:1.7.12-alpine
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
        - name: admin
          containerPort: 8080
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        resources:
          limits:
            cpu: 1
            memory: 1024Mi
          requests:
            cpu: 0.25
            memory: 512Mi
```

部署完成之后，可以访问Traefik的控制台（只能看Ingress规则）：`MasterIP：8080`，更多配置可参考Traefik官网：`https://docs.traefik.io`。

### 15 结语

`Nginx-ingress-controller`或`Traefik-ingress-controller`部署完成之后，解析相关域名如`console.cloudnil.com`到master01、master02、master03的外网IP，就可以使用`console.cloudnil.com`访问dashboard，其他应用类似。

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2019/06/23/Deploy-kubernetes(1.17.0)-HA-with-kubeadm/。