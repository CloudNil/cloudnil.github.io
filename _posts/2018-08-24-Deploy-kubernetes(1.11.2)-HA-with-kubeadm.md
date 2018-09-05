---
layout: post
title:  kubeadm快速部署kubernetes(1.11.2,HA)
categories: [docker,Cloud]
date: 2018-08-24 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>当前版本的kubeadm已经原生支持部署HA模式集群，非常方便即可实现HA模式的kubernetes集群。本次部署基于Ubuntu16.04，并使用最新的docker版本：18.03.1，kubernetes适用1.11.x版本，本文采用1.11.2。

### 1 环境准备

准备了六台机器作安装测试工作，机器信息如下: 

|      IP      |   Name   |       Role      |      OS     |
|--------------|----------|-----------------|-------------|
| 172.16.2.1   | Master01 | Controller,etcd | Ubuntu16.04 |
| 172.16.2.2   | Master02 | Controller,etcd | Ubuntu16.04 |
| 172.16.2.3   | Master03 | Controller,etcd | Ubuntu16.04 |
| 172.16.2.11  | Node01   | Compute,DNS     | Ubuntu16.04 |
| 172.16.2.12  | Node02   | Compute,DNS     | Ubuntu16.04 |
| 172.16.2.13  | Node03   | Compute,DNS     | Ubuntu16.04 |

### 2 安装docker

```Bash
apt-get update && apt-get install -y apt-transport-https software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

apt-get update

apt-get install docker-ce=18.03.1~ce-0~ubuntu
```

### 3 安装etcd集群

使用了docker-compose安装，当然，如果觉得麻烦，也可以直接docker run。

Master01节点的ETCD的docker-compose.yml：

```yaml
etcd:
  image: quay.io/coreos/etcd:v3.2.17
  command: etcd --name etcd-srv1 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.1:2379,http://172.16.2.1:2380 --initial-advertise-peer-urls http://172.16.2.1:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.2:2380,etcd-srv3=http://172.16.2.3:2380" -initial-cluster-state new
  net: "bridge"
  ports:
  - "2379:2379"
  - "2380:2380"
  restart: always
  stdin_open: true
  tty: true
  volumes:
  - /store/etcd:/var/etcd
```

Master02节点的ETCD的docker-compose.yml：

```yaml
etcd:
  image: quay.io/coreos/etcd:v3.2.17
  command: etcd --name etcd-srv2 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.2:2379,http://172.16.2.2:2380 --initial-advertise-peer-urls http://172.16.2.2:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.2:2380,etcd-srv3=http://172.16.2.3:2380" -initial-cluster-state new
  net: "bridge"
  ports:
  - "2379:2379"
  - "2380:2380"
  restart: always
  stdin_open: true
  tty: true
  volumes:
  - /store/etcd:/var/etcd
```

Master03节点的ETCD的docker-compose.yml：

```yaml
etcd:
  image: quay.io/coreos/etcd:v3.2.17
  command: etcd --name etcd-srv3 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.3:2379,http://172.16.2.3:2380 --initial-advertise-peer-urls http://172.16.2.3:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.2:2380,etcd-srv3=http://172.16.2.3:2380" -initial-cluster-state new
  net: "bridge"
  ports:
  - "2379:2379"
  - "2380:2380"
  restart: always
  stdin_open: true
  tty: true
  volumes:
  - /store/etcd:/var/etcd
```

创建好docker-compose.yml文件后，使用命令`docker-compose up -d`部署。

>关于docker-compose的使用，可以参考：[docker-compose安装文档](https://docs.docker.com/compose/install/#alternative-install-options)。

### 3 安装k8s工具包

**阿里源安装**

```Bash
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl ipvsadm
```

### 4 启用ipvs模块

本方案中采用ipvs作为kube-proxy的转发机制，效率比iptables高很多，开启ipvs模块支持。

```Bash
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

### 5 镜像准备

#### 5.1 下载docker镜像

kubeadm方式安装kubernetes集群需要的镜像在docker官方镜像中并未提供，只能去google的官方镜像库：`gcr.io` 中下载，GFW咋办？本文中所使用镜像在DockerHub上做了跳板镜像，各位可以直接下载。

kubernetes-1.11.2所需要的镜像：

- etcd-amd64:3.2.17
- pause-amd64:3.1
- kube-proxy-amd64:v1.11.2
- kube-scheduler-amd64:v1.11.2
- kube-controller-manager-amd64:v1.11.2
- kube-apiserver-amd64:v1.11.2
- coredns:1.1.3

偷下懒吧，直接执行以下脚本，提前下载好镜像，后边的动作就快了：

```Bash
#!/bin/bash
images=(kube-proxy-amd64:v1.11.2 kube-scheduler-amd64:v1.11.2 kube-controller-manager-amd64:v1.11.2 kube-apiserver-amd64:v1.11.2 etcd-amd64:3.2.17 pause-amd64:3.1 coredns:1.1.3)
for imageName in ${images[@]} ; do
  docker pull cloudnil/$imageName
done
```

#### 5.2 KUBE_REPO_PREFIX配置

通过KUBE_REPO_PREFIX配置官方镜像包的仓库位置，才可以直接使用从DockerHub上下载的镜像，请使用以下命令增加配置：1.KUBE_REPO_PREFIX环境变量 2.KUBELET_EXTRA_ARGS参数。

```
sed -i '/mesg n/i\export KUBECONFIG=/etc/kubernetes/admin.conf' ~/.profile
source ~/.profile

sed -i 's/KUBELET_EXTRA_ARGS=/&--pod-infra-container-image=cloudnil\/pause-amd64:3.1/' /etc/default/kubelet

systemctl daemon-reload
systemctl restart kubelet
```

### 6 Api-Server负载均衡

配置负载均衡器对kube-apiserver进行负载均衡，可采用DNS轮询解析或者Haproxy（Nginx）反向代理实现负载均衡。

本文采用DNS轮询解析实现简单的负载均衡，在Node01,Node02,Node03节点上部署DNS。

1、修改`/etc/hosts`文件，添加域名解析

```
172.16.2.1 api.me
172.16.2.2 api.me
172.16.2.3 api.me
```

2、docker-compose部署dnsmasq服务：

```yaml
version: "3"
services:
  dnsmasq:
    image: cloudnil/dnsmasq:2.76
    command: -q --log-facility=- --all-servers
    network_mode: "host"
    cap_add:
    - NET_ADMIN
    restart: always
    stdin_open: true
    tty: true
```

3、除了部署dnsmasq服务的其他所有节点上，配置DNS

```
cat <<EOF >/etc/resolvconf/resolv.conf.d/base
nameserver 172.16.2.11
nameserver 172.16.2.12
nameserver 172.16.2.13
EOF
```

### 7 安装master节点

kubeadm配置文件kubeadm-config.yml：

```yaml
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.2
imageRepository: cloudnil
apiServerCertSANs:
- api.me
api:
  controlPlaneEndpoint: api.me:6443
etcd:
  external:
    endpoints:
    - http://172.16.2.1:2379
    - http://172.16.2.2:2379
    - http://172.16.2.3:2379
networking:
  podSubnet: "10.68.0.0/16"
kubeletConfiguration:
  baseConfig:
    systemReserved: 
      cpu: "0.25"
      memory: "128Mi"
kubeProxy:
  config:
    ipvs:
      minSyncPeriod: 1s
      scheduler: rr
      syncPeriod: 10s
    mode: ipvs
```

master01初始化指令：

```bash
kubeadm init --config kubeadm-config.yml
```

安装过程大概30秒，输出结果如下：

```Bash
[endpoint] WARNING: port specified in api.controlPlaneEndpoint overrides api.bindPort in the controlplane address
[init] using Kubernetes version: v1.11.2
[preflight] running pre-flight checks
I0828 08:51:30.883141    1526 kernel_validator.go:81] Validating kernel version
I0828 08:51:30.883269    1526 kernel_validator.go:96] Validating kernel config
  [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.03.1-ce. Max validated version: 17.03
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [master01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local api.me api.me] and IPs [10.96.0.1 172.16.2.1]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki"
[endpoint] WARNING: port specified in api.controlPlaneEndpoint overrides api.bindPort in the controlplane address
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[init] waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests" 
[init] this might take a minute or longer if the control plane images have to be pulled
[apiclient] All control plane components are healthy after 41.441297 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.11" in namespace kube-system with the configuration for the kubelets in the cluster
[markmaster] Marking the node master01 as master by adding the label "node-role.kubernetes.io/master=''"
[markmaster] Marking the node master01 as master by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "master01" as an annotation
[bootstraptoken] using token: duvzzs.nby9ixavfkcumuuq
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[endpoint] WARNING: port specified in api.controlPlaneEndpoint overrides api.bindPort in the controlplane address
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join api.me:6443 --token duvzzs.nby9ixavfkcumuuq --discovery-token-ca-cert-hash sha256:2fb5382e9821c3aacd706c7fc9a85975aea360f6f344244330a5bb59121ac0d0
```

>PS：`token`是使用指令`kubeadm token generate`生成的，执行过程如有异常，用命令`kubeadm reset`初始化后重试，生成的`token`有效时间为24小时，超过24小时后需要重新使用命令`kubeadm token create`创建新的`token`。

复制`/etc/kubernetes/pki`下的以下文件到Master02和Master03对应目录，`.ssh/cloudnil.pem`是方便节点之间访问的证书，可以使用`ssh-keygen`生成，具体使用不详细阐述，网络上文章很多，也可以直接使用账号密码。

```
USER=root
CONTROL_PLANE_IPS="172.16.2.2 172.16.2.3"
for host in ${CONTROL_PLANE_IPS}; do
    scp -i .ssh/cloudnil.pem /etc/kubernetes/pki/ca.crt "${USER}"@$host:/etc/kubernetes/pki
    scp -i .ssh/cloudnil.pem /etc/kubernetes/pki/ca.key "${USER}"@$host:/etc/kubernetes/pki
    scp -i .ssh/cloudnil.pem /etc/kubernetes/pki/sa.key "${USER}"@$host:/etc/kubernetes/pki
    scp -i .ssh/cloudnil.pem /etc/kubernetes/pki/sa.pub "${USER}"@$host:/etc/kubernetes/pki
    scp -i .ssh/cloudnil.pem /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:/etc/kubernetes/pki
    scp -i .ssh/cloudnil.pem /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:/etc/kubernetes/pki
done
```

在master02，master03上分别用同样配置文件`kubeadm-config.yml`的执行：

```bash
kubeadm init --config kubeadm-config.yml
```

### 8 安装Node节点

Master节点安装好了Node节点就简单了,在各个Node节点上执行。

```Bash
kubeadm join api.me:6443 --token duvzzs.nby9ixavfkcumuuq --discovery-token-ca-cert-hash sha256:2fb5382e9821c3aacd706c7fc9a85975aea360f6f344244330a5bb59121ac0d0
```

输出结果如下：

```Bash
[preflight] running pre-flight checks
I0828 18:30:09.878584   30585 kernel_validator.go:81] Validating kernel version
I0828 18:30:09.878688   30585 kernel_validator.go:96] Validating kernel config
  [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.03.1-ce. Max validated version: 17.03
[discovery] Trying to connect to API Server "api.me:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://api.me:6443"
[discovery] Requesting info from "https://api.me:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "api.me:6443"
[discovery] Successfully established connection with API Server "api.me:6443"
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.11" ConfigMap in the kube-system namespace
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[preflight] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "node01" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

安装完成后可以查看下状态，未安装网络组件，所以全部都是NotReady状态：

```Bash
NAME       STATUS     ROLES     AGE       VERSION
master01   NotReady   master    1h        v1.11.2
master02   NotReady   master    1h        v1.11.2
master03   NotReady   master    1h        v1.11.2
node01     NotReady   <none>    1m        v1.11.2
node02     NotReady   <none>    1m        v1.11.2
node03     NotReady   <none>    1m        v1.11.2
```

### 8 安装Calico网络

网络组件选择很多，可以根据自己的需要选择calico、weave、flannel，calico性能最好，flannel的vxlan也不错，默认的UDP性能较差，weave的性能比较差，测试环境用下可以，生产环境不建议使用。calico的安装配置可以参考官方部署：[https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/calico](https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/calico)

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
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-kube-controllers
rules:
  - apiGroups:
    - ""
    - extensions
    resources:
      - pods
      - namespaces
      - networkpolicies
      - nodes
      - serviceaccounts
    verbs:
      - watch
      - list
  - apiGroups:
    - networking.k8s.io
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
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
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-node
rules:
  - apiGroups: [""]
    resources:
      - pods
      - nodes
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
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
# Calico Version v3.2.1
# https://docs.projectcalico.org/v3.2/releases#v3.2.1
# This manifest includes the following component versions:
#   calico/node:v3.2.1
#   calico/cni:v3.2.1
#   calico/kube-controllers:v3.2.1

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
  # Configure the Calico backend to use.
  calico_backend: "bird"

  # Configure the MTU to use
  veth_mtu: "1440"

  # The CNI network configuration to install on each node.  The special
  # values in this config will be automatically populated.
  cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.0",
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
# The following contains k8s Secrets for use with a TLS enabled etcd cluster.
# For information on populating Secrets, see http://kubernetes.io/docs/user-guide/secrets/
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # Populate the following files with etcd TLS configuration if desired, but leave blank if
  # not using TLS for etcd.
  # This self-hosted install expects three files with the following names.  The values
  # should be base64 encoded strings of the entire contents of each file.
  # etcd-key: null
  # etcd-cert: null
  # etcd-ca: null
---
# This manifest installs the calico/node container, as well
# as the Calico CNI plugins and network config on
# each master and worker node in a Kubernetes cluster.
kind: DaemonSet
apiVersion: extensions/v1beta1
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
      containers:
        # Runs calico/node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: quay.io/calico/node:v3.2.1
          env:
            # The location of the Calico etcd cluster.
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
            httpGet:
              path: /liveness
              port: 9099
              host: localhost
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              - -bird-ready
              - -felix-ready
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
            - mountPath: /var/lib/calico
              name: var-lib-calico
              readOnly: false
            - mountPath: /calico-secrets
              name: etcd-certs
        # This container installs the Calico CNI binaries
        # and CNI network config file on each node.
        - name: install-cni
          image: quay.io/calico/cni:v3.2.1
          command: ["/install-cni.sh"]
          env:
            # Name of the CNI config file to create.
            - name: CNI_CONF_NAME
              value: "10-calico.conflist"
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # The CNI network config to install on each node.
            - name: CNI_NETWORK_CONFIG
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: cni_network_config
            # CNI MTU Config variable
            - name: CNI_MTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
          volumeMounts:
            - mountPath: /host/opt/cni/bin
              name: cni-bin-dir
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
            - mountPath: /calico-secrets
              name: etcd-certs
      volumes:
        # Used by calico/node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        - name: var-lib-calico
          hostPath:
            path: /var/lib/calico
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
---
# This manifest deploys the Calico Kubernetes controllers.
# See https://github.com/projectcalico/kube-controllers
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: calico-kube-controllers
  namespace: kube-system
  labels:
    k8s-app: calico-kube-controllers
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ''
spec:
  # The controllers can only have a single active instance.
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      name: calico-kube-controllers
      namespace: kube-system
      labels:
        k8s-app: calico-kube-controllers
    spec:
      # The controllers must run in the host network namespace so that
      # it isn't governed by policy that would prevent it from working.
      hostNetwork: true
      tolerations:
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      serviceAccountName: calico-kube-controllers
      containers:
        - name: calico-kube-controllers
          image: quay.io/calico/kube-controllers:v3.2.1
          env:
            # The location of the Calico etcd cluster.
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
              value: policy,profile,workloadendpoint,node
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

```Bash
NAME                                       READY     STATUS    RESTARTS   AGE       IP              NODE       NOMINATED NODE
calico-kube-controllers-5945598866-2f89g   1/1       Running   0          10m       172.16.2.1     master01   <none>
calico-node-6zgpf                          2/2       Running   0          10m       172.16.2.13    node03     <none>
calico-node-7cldg                          2/2       Running   0          10m       172.16.2.2     master02   <none>
calico-node-98lq8                          2/2       Running   0          10m       172.16.2.3     master03   <none>
calico-node-f2rwg                          2/2       Running   0          10m       172.16.2.12    node02     <none>
calico-node-ttnqs                          2/2       Running   0          10m       172.16.2.1     master01   <none>
calico-node-xlv9t                          2/2       Running   0          10m       172.16.2.11    node01     <none>
coredns-76949ddffb-7xxt5                   1/1       Running   0          14s       10.68.186.194  node03     <none>
coredns-76949ddffb-pc4n9                   1/1       Running   0          14s       10.68.140.66   node02     <none>
kube-apiserver-master01                    1/1       Running   0          9h        172.16.2.1     master01   <none>
kube-apiserver-master02                    1/1       Running   0          9h        172.16.2.2     master02   <none>
kube-apiserver-master03                    1/1       Running   0          9h        172.16.2.3     master03   <none>
kube-controller-manager-master01           1/1       Running   0          9h        172.16.2.1     master01   <none>
kube-controller-manager-master02           1/1       Running   0          9h        172.16.2.2     master02   <none>
kube-controller-manager-master03           1/1       Running   0          9h        172.16.2.3     master03   <none>
kube-proxy-55n9q                           1/1       Running   0          4m        172.16.2.12    node02     <none>
kube-proxy-878jx                           1/1       Running   0          9h        172.16.2.1     master01   <none>
kube-proxy-cn7qx                           1/1       Running   0          4m        172.16.2.11    node01     <none>
kube-proxy-k82xq                           1/1       Running   0          9h        172.16.2.2     master02   <none>
kube-proxy-lqb7s                           1/1       Running   0          4m        172.16.2.13    node03     <none>
kube-proxy-sqgm7                           1/1       Running   0          9h        172.16.2.3     master03   <none>
kube-scheduler-master01                    1/1       Running   0          9h        172.16.2.1     master01   <none>
kube-scheduler-master02                    1/1       Running   0          9h        172.16.2.2     master02   <none>
kube-scheduler-master03                    1/1       Running   0          9h        172.16.2.3     master03   <none>
```

>说明：kube-dns需要等calico配置完成后才是running状态。

### 9 DNS集群部署

删除原单点coredns

```bash
kubectl delete deploy coredns -n kube-system
```

部署多实例的coredns集群，参考配置coredns.yml：

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system
spec:
  replicas: 3
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
        image: hub.lonhcloud.com/coredns:1.1.3
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

### 10 部署Dashboard

下载kubernetes-dashboard.yaml

```Bash
curl -O https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

修改配置内容，部署到default的namespace，增加ingress配置，后边配置了nginx-ingress后就可以直接绑定域名访问了。

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
---
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kube-system
spec:
  rules:
  - host: dashboard.cloudnil.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```

### 11 服务暴露到公网

kubernetes中的Service暴露到外部有三种方式，分别是：

- LoadBlancer Service
- NodePort Service
- Ingress

LoadBlancer Service是kubernetes深度结合云平台的一个组件；当使用LoadBlancer Service暴露服务时，实际上是通过向底层云平台申请创建一个负载均衡器来向外暴露服务；目前LoadBlancer Service支持的云平台已经相对完善，比如国外的GCE、DigitalOcean，国内的 阿里云，私有云 Openstack 等等，由于LoadBlancer Service深度结合了云平台，所以只能在一些云平台上来使用。

NodePort Service顾名思义，实质上就是通过在集群的每个node上暴露一个端口，然后将这个端口映射到某个具体的service来实现的，虽然每个node的端口有很多(0~65535)，但是由于安全性和易用性(服务多了就乱了，还有端口冲突问题)实际使用可能并不多。

Ingress可以实现使用nginx等开源的反向代理负载均衡器实现对外暴露服务，可以理解Ingress就是用于配置域名转发的一个东西，在nginx中就类似upstream，它与ingress-controller结合使用，通过ingress-controller监控到pod及service的变化，动态地将ingress中的转发信息写到诸如nginx、apache、haproxy等组件中实现方向代理和负载均衡。

### 12 部署Nginx-ingress-controller

`Nginx-ingress-controller`是kubernetes官方提供的集成了Ingress-controller和Nginx的一个docker镜像。

```yaml
apiVersion: v1
kind: ServiceAccount
imagePullSecrets:
metadata:
  name: nginx-ingress-controller
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: view-services-cluster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: nginx-ingress-controller
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    k8s-app: default-http-backend
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: default-http-backend
  template:
    metadata:
      labels:
        k8s-app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        image: hub.lonhcloud.com/defaultbackend:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  labels:
    k8s-app: default-http-backend
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    k8s-app: default-http-backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  labels:
    k8s-app: nginx-ingress-controller
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      k8s-app: nginx-ingress-controller
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: nginx-ingress-controller
    spec:
      hostNetwork: true
      terminationGracePeriodSeconds: 60
      serviceAccountName: nginx-ingress-controller
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
                  - key: k8s-app
                    operator: In
                    values: 
                    - nginx-ingress-controller
              topologyKey: "kubernetes.io/hostname"
      containers:
      - image: hub.lonhcloud.com/nginx-ingress-controller:0.17.1
        name: nginx-ingress-controller
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
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
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
        - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
        resources:
          limits:
            cpu: 1
            memory: 1024Mi
          requests:
            cpu: 0.25
            memory: 512Mi
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
```

部署完Nginx-ingress-controller后，解析域名`dashboard.cloudnil.com`到node02的外网IP，就可以使用`dashboard.cloudnil.com`访问dashboard。

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2018/08/24/Deploy-kubernetes(1.11.2)-HA-with-kubeadm/。