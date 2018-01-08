---
layout: post
title:  kubeadm快速部署kubernetes1.7.6
categories: [docker,Cloud]
date: 2017-09-20 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>Kubernetes 1.7.6+发布，调整部署文档。本次部署基于Ubuntu16.04，并使用最新的docker版本：17.06。

### 1 环境准备

准备了三台机器作安装测试工作，机器信息如下: 

|      IP     |  Name  |       Role      |      OS     |
|-------------|--------|-----------------|-------------|
| 172.16.2.1  | Master | Controller,etcd | Ubuntu16.04 |
| 172.16.2.11 | Node01 | Compute,etcd    | Ubuntu16.04 |
| 172.16.2.12 | Node02 | Compute,etcd    | Ubuntu16.04 |

### 2 安装docker

```Bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

apt-get update && apt-upgrade

apt-get install docker-ce=17.06.0~ce-0~ubuntu
```

### 3 安装etcd集群

使用了docker-compose安装，当然，如果觉得麻烦，也可以直接docker run。

Master节点的ETCD的docker-compose.yml：

```yaml
etcd:
  image: quay.io/coreos/etcd:v3.1.5
  command: etcd --name etcd-srv1 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.1:2379,http://172.16.2.1:2380 --initial-advertise-peer-urls http://172.16.2.1:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.11:2380,etcd-srv3=http://172.16.2.12:2380" -initial-cluster-state new
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

Node01节点的ETCD的docker-compose.yml：

```yaml
etcd:
  image: quay.io/coreos/etcd:v3.1.5
  command: etcd --name etcd-srv2 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.11:2379,http://172.16.2.11:2380 --initial-advertise-peer-urls http://172.16.2.11:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.11:2380,etcd-srv3=http://172.16.2.12:2380" -initial-cluster-state new
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

Node02节点的ETCD的docker-compose.yml：

```yaml
etcd:
  image: quay.io/coreos/etcd:v3.1.5
  command: etcd --name etcd-srv3 --data-dir=/var/etcd/calico-data --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.2.12:2379,http://172.16.2.12:2380 --initial-advertise-peer-urls http://172.16.2.12:2380 --listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-srv1=http://172.16.2.1:2380,etcd-srv2=http://172.16.2.11:2380,etcd-srv3=http://172.16.2.12:2380" -initial-cluster-state new
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

三种方式：博主提供、官方源安装和release工程编译，apt-get方式因为不能直接使用google提供的源，非官方源中提供的版本比较老，如果要使用新版本，可以尝试release工程编译的方式或者用博主提供的包下载。

**博主提供**

一些比较懒得同学:-D，可以直接从博主提供的位置下载RPM工具包安装，[下载地址](https://github.com/CloudNil/kubernetes-library/tree/master/package)。

```Bash
#安装kubelet的依赖包
apt-get install -y socat ebtables
dpkg -i kubelet_1.7.6-00_amd64.deb kubeadm_1.7.6-00_amd64.deb kubernetes-cni_0.5.1-00_amd64.deb kubectl_1.7.6-00_amd64.deb
```

**官方源安装**

跨越GFW方式不细说，你懂的。

```Bash
apt-get update && apt-get install -y apt-transport-https

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update

apt-get install -y kubelet kubeadm kubernetes-cni kubectl
```

>默认安装最新的stable版本，可以根据需要指定安装版本`apt-get install -y kubeadm=1.7.6-00`，版本信息可以使用命令查看：`apt-cache madison kubeadm`。

**relese编译**

```Bash
git clone https://github.com/kubernetes/release.git

docker build --tag=debian-packager debian
docker run --volume="$(pwd)/debian:/src" debian-packager
```

编译完成后生成deb包到：`debian/bin`，进入到该目录后安装deb包。

### 4 镜像准备

#### 4.1 下载docker镜像

kubeadm方式安装kubernetes集群需要的镜像在docker官方镜像中并未提供，只能去google的官方镜像库：`gcr.io` 中下载，GFW咋办？这里针对k8s-1.7.6在DockerHub上做了跳板镜像，各位可以直接下载，dashboard的版本并未紧跟kubelet主线版本，用哪个版本都可以，本文使用kubernetes-dashboard-amd64:v1.7.0。

kubernetes-1.7.6所需要的镜像：

- etcd-amd64:3.0.17
- pause-amd64:3.0
- kube-proxy-amd64:v1.7.6 
- kube-scheduler-amd64:v1.7.6
- kube-controller-manager-amd64:v1.7.6 
- kube-apiserver-amd64:v1.7.6
- kubernetes-dashboard-amd64:v1.7.0
- k8s-dns-sidecar-amd64:1.14.4
- k8s-dns-kube-dns-amd64:1.14.4
- k8s-dns-dnsmasq-nanny-amd64:1.14.4

偷下懒吧，直接执行以下脚本，提前下载好镜像，后边的动作就快了：

```Bash
#!/bin/bash
images=(kube-proxy-amd64:v1.7.6 kube-scheduler-amd64:v1.7.6 kube-controller-manager-amd64:v1.7.6 kube-apiserver-amd64:v1.7.6 etcd-amd64:3.0.17 pause-amd64:3.0 kubernetes-dashboard-amd64:v1.6.1 k8s-dns-sidecar-amd64:1.14.4 k8s-dns-kube-dns-amd64:1.14.4 k8s-dns-dnsmasq-nanny-amd64:1.14.4)
for imageName in ${images[@]} ; do
  docker pull cloudnil/$imageName
done
```

#### 4.2 KUBE_REPO_PREFIX配置

通过KUBE_REPO_PREFIX配置官方镜像包的仓库位置，才可以直接使用从DockerHub上下载的镜像，请使用以下命令增加配置：1.KUBE_REPO_PREFIX环境变量 2.KUBELET_EXTRA_ARGS参数。

```
sed -i '/mesg n/i\export KUBE_REPO_PREFIX=cloudnil' ~/.profile
source ~/.profile

cat > /etc/systemd/system/kubelet.service.d/20-extra-args.conf <<EOF
[Service]
Environment="KUBELET_EXTRA_ARGS=--pod-infra-container-image=cloudnil/pause-amd64:3.0"
EOF

systemctl daemon-reload
systemctl restart kubelet
```

### 5 安装master节点

由于kubeadm和kubelet安装过程中会生成`/etc/kubernetes`目录，而`kubeadm init`会先检测该目录是否存在，所以我们先使用kubeadm初始化环境。

```Bash
kubeadm reset
kubeadm init --api-advertise-addresses=172.16.2.1 --use-kubernetes-version v1.7.6
```

如果使用外部etcd集群，以前的kubeadm版本的`--external-etcd-endpoints`参数已经没有了，所以要使用--config参数外挂配置文件kubeadm-config.yml：

```yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: 172.16.2.1
etcd:
  endpoints:
  - http://172.16.2.1:2379
  - http://172.16.2.11:2379
  - http://172.16.2.12:2379
kubernetesVersion: v1.7.6
```

初始化指令：

```bash
kubeadm init --config kubeadm-config.yml
```

>说明：如果打算使用flannel网络，请加上：`--pod-network-cidr=10.244.0.0/16`。如果有多网卡的，请根据实际情况配置`--api-advertise-addresses=<ip-address>`，单网卡情况可以省略。

安装过程大概2-3分钟，输出结果如下：

```Bash
[kubeadm] WARNING: kubeadm is in alpha, please do not use it for production clusters.
[preflight] Running pre-flight checks
[init] Using Kubernetes version: v1.7.6
[tokens] Generated token: "064158.548b9ddb1d3fad3e"
[certificates] Generated Certificate Authority key and certificate.
[certificates] Generated API Server key and certificate
[certificates] Generated Service Account signing keys
[certificates] Created keys and certificates in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 21.317580 seconds
[apiclient] Waiting for at least one node to register and become ready
[apiclient] First node is ready after 6.556101 seconds
[apiclient] Creating a test deployment
[apiclient] Test deployment succeeded
[addons] Created essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node:

kubeadm join --token=de3d61.504a049ec342e135 172.16.2.1
```

### 6 安装Node节点

Master节点安装好了Node节点就简单了。

```Bash
kubeadm reset
kubeadm join --token=de3d61.504a049ec342e135 172.16.2.1
```

输出结果如下：

```Bash
[kubeadm] WARNING: kubeadm is in alpha, please do not use it for production clusters.
[preflight] Running pre-flight checks
[preflight] Starting the kubelet service
[tokens] Validating provided token
[discovery] Created cluster info discovery client, requesting info from "http://172.16.2.1:9898/cluster-info/v1/?token-id=f11877"
[discovery] Cluster info object received, verifying signature using given token
[discovery] Cluster info signature and contents are valid, will use API endpoints [https://172.16.2.1:6443]
[bootstrap] Trying to connect to endpoint https://172.16.2.1:6443
[bootstrap] Detected server version: v1.7.6
[bootstrap] Successfully established connection with endpoint "https://172.16.2.1:6443"
[csr] Created API client to obtain unique certificate for this node, generating keys and certificate signing request
[csr] Received signed certificate from the API server:
Issuer: CN=kubernetes | Subject: CN=system:node:yournode | CA: false
Not before: 2017-06-28 19:44:00 +0000 UTC Not After: 2018-06-28 19:44:00 +0000 UTC
[csr] Generating kubelet configuration
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"

Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```

安装完成后可以查看下状态，未安装网络组件，所以全部都是NotReady状态：

```Bash
NAME      STATUS     AGE       VERSION
master    NotReady   1h        v1.7.6
node01    NotReady   1h        v1.7.6
node02    NotReady   1h        v1.7.6
```

### 7 安装Calico网络

网络组件选择很多，可以根据自己的需要选择calico、weave、flannel，calico性能最好，flannel的vxlan也不错，默认的UDP性能较差，weave的性能比较差，测试环境用下可以，生产环境不建议使用。[Addons](http://kubernetes.io/docs/admin/addons/)中有配置好的yaml，所以本文中尝试calico网络，。

```Bash
kubectl apply -f https://docs.projectcalico.org/v2.5/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
```

如果使用了外部etcd，去掉etcd相关配置内容，并修改`etcd_endpoints: [ETCD_ENDPOINTS]`：

```yaml
# Calico Version v2.5.1
# https://docs.projectcalico.org/v2.5/releases#v2.5.1
# This manifest includes the following component versions:
#   calico/node:v2.5.1
#   calico/cni:v1.10.0
#   calico/kube-policy-controller:v0.7.0

# This ConfigMap is used to configure a self-hosted Calico installation.
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  # The location of your etcd cluster.  This uses the Service clusterIP defined below.
  etcd_endpoints: "http://172.16.2.1:2379,http://172.16.2.11:2379,http://172.16.2.12:2379"

  # Configure the Calico backend to use.
  calico_backend: "bird"

  # The CNI network configuration to install on each node.
  cni_network_config: |-
    {
        "name": "k8s-pod-network",
        "cniVersion": "0.1.0",
        "type": "calico",
        "etcd_endpoints": "__ETCD_ENDPOINTS__",
        "log_level": "info",
        "mtu": 1500,
        "ipam": {
            "type": "calico-ipam"
        },
        "policy": {
            "type": "k8s",
             "k8s_api_root": "https://__KUBERNETES_SERVICE_HOST__:__KUBERNETES_SERVICE_PORT__",
             "k8s_auth_token": "__SERVICEACCOUNT_TOKEN__"
        },
        "kubernetes": {
            "kubeconfig": "/etc/cni/net.d/__KUBECONFIG_FILENAME__"
        }
    }

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
  template:
    metadata:
      labels:
        k8s-app: calico-node
      annotations:
        # Mark this pod as a critical add-on; when enabled, the critical add-on scheduler
        # reserves resources for critical add-on pods so that they can be rescheduled after
        # a failure.  This annotation works in tandem with the toleration below.
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      hostNetwork: true
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      # Allow this pod to be rescheduled while the node is in "critical add-ons only" mode.
      # This, along with the annotation above marks this pod as a critical add-on.
      - key: CriticalAddonsOnly
        operator: Exists
      serviceAccountName: calico-cni-plugin
      containers:
        # Runs calico/node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: quay.io/calico/node:v2.5.1
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Enable BGP.  Disable to enforce policy only.
            - name: CALICO_NETWORKING_BACKEND
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: calico_backend
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "kubeadm,bgp"
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # Configure the IP Pool from which Pod IPs will be chosen.
            - name: CALICO_IPV4POOL_CIDR
              value: "10.68.0.0/16"
            - name: CALICO_IPV4POOL_IPIP
              value: "always"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            # Set MTU for tunnel device used if ipip is enabled
            - name: FELIX_IPINIPMTU
              value: "1440"
            # Set Felix logging to "info"
            - name: FELIX_LOGSEVERITYSCREEN
              value: "info"
            # Auto-detect the BGP IP address.
            - name: IP
              value: ""
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
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            httpGet:
              path: /readiness
              port: 9099
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
        # This container installs the Calico CNI binaries
        # and CNI network config file on each node.
        - name: install-cni
          image: quay.io/calico/cni:v1.10.0
          command: ["/install-cni.sh"]
          env:
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
          volumeMounts:
            - mountPath: /host/opt/cni/bin
              name: cni-bin-dir
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
      volumes:
        # Used by calico/node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        # Used to install CNI.
        - name: cni-bin-dir
          hostPath:
            path: /opt/cni/bin
        - name: cni-net-dir
          hostPath:
            path: /etc/cni/net.d

---

# This manifest deploys the Calico policy controller on Kubernetes.
# See https://github.com/projectcalico/k8s-policy
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: calico-policy-controller
  namespace: kube-system
  labels:
    k8s-app: calico-policy
spec:
  # The policy controller can only have a single active instance.
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      name: calico-policy-controller
      namespace: kube-system
      labels:
        k8s-app: calico-policy-controller
      annotations:
        # Mark this pod as a critical add-on; when enabled, the critical add-on scheduler
        # reserves resources for critical add-on pods so that they can be rescheduled after
        # a failure.  This annotation works in tandem with the toleration below.
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      # The policy controller must run in the host network namespace so that
      # it isn't governed by policy that would prevent it from working.
      hostNetwork: true
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      # Allow this pod to be rescheduled while the node is in "critical add-ons only" mode.
      # This, along with the annotation above marks this pod as a critical add-on.
      - key: CriticalAddonsOnly
        operator: Exists
      serviceAccountName: calico-policy-controller
      containers:
        - name: calico-policy-controller
          image: quay.io/calico/kube-policy-controller:v0.7.0
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # The location of the Kubernetes API.  Use the default Kubernetes
            # service for API access.
            - name: K8S_API
              value: "https://kubernetes.default:443"
            # Since we're running in the host namespace and might not have KubeDNS
            # access, configure the container's /etc/hosts to resolve
            # kubernetes.default to the correct service clusterIP.
            - name: CONFIGURE_ETC_HOSTS
              value: "true"
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: calico-cni-plugin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-cni-plugin
subjects:
- kind: ServiceAccount
  name: calico-cni-plugin
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-cni-plugin
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources:
      - pods
      - nodes
    verbs:
      - get
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-cni-plugin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: calico-policy-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-policy-controller
subjects:
- kind: ServiceAccount
  name: calico-policy-controller
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-policy-controller
  namespace: kube-system
rules:
  - apiGroups:
    - ""
    - extensions
    resources:
      - pods
      - namespaces
      - networkpolicies
    verbs:
      - watch
      - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-policy-controller
  namespace: kube-system
```

检查各节点组件运行状态：

```Bash
NAME                                        READY     STATUS    RESTARTS   AGE
calico-node-34b1k                           2/2       Running   0          21m
calico-node-bz8cw                           2/2       Running   0          21m
calico-node-psjj1                           2/2       Running   0          21m
calico-policy-controller-1324707180-97r1c   1/1       Running   2          21m
kube-apiserver-master                       1/1       Running   0          13m
kube-controller-manager-master              1/1       Running   6          23m
kube-dns-1076809945-l59j9                   3/3       Running   0          23m
kube-proxy-4bcc9                            1/1       Running   0          22m
kube-proxy-f0sq2                            1/1       Running   0          23m
kube-proxy-p6ksj                            1/1       Running   0          22m
kube-scheduler-master                       1/1       Running   6          23m
```

>说明：kube-dns需要等calico配置完成后才是running状态。

### 8 部署Dashboard

下载kubernetes-dashboard.yaml

```Bash
curl -O https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

修改配置内容，部署到default的namespace，增加ingress配置，后边配置了nginx-ingress后就可以直接绑定域名访问了。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: default
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: default
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
        image: cloudnil/kubernetes-dashboard-amd64:v1.7.0
        ports:
        - containerPort: 9090
          protocol: TCP
        args:
        livenessProbe:
          httpGet:
            path: /
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
      serviceAccountName: kubernetes-dashboard
      # Comment the following tolerations if Dashboard must not be deployed on master
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
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 9090
  selector:
    k8s-app: kubernetes-dashboard
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: default
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
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    k8s-app: default-http-backend
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        image: cloudnil/defaultbackend:1.0
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
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  labels:
    k8s-app: nginx-ingress-controller
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: nginx-ingress-controller
    spec:
      hostNetwork: true
      nodeName: master
      terminationGracePeriodSeconds: 60
      serviceAccountName: nginx-ingress-controller
      containers:
      - image: cloudnil/nginx-ingress-controller:0.9.0-beta.13
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
---
apiVersion: v1
kind: ServiceAccount
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
```

部署完Nginx-ingress-controller后，解析域名`dashboard.cloudnil.com`到node02的外网IP，就可以使用`dashboard.cloudnil.com`访问dashboard。

### 10 注意事项

kubeadm目前还在开发测试阶段，不建议在生产环境中使用kubeadm部署kubernetes环境。此外，使用kubeadm是需要注意以下几点：

#### 10.1 单点故障

当前版本的kubeadm暂且不能部署真正高可用的kubernetes环境，只具有单点的master环境，如采用内置etcd，那etcd也是单节点，若master节点故障，可能存在数据丢失的情况，所以建议采用外部的etcd集群，这样即使master节点故障，那只要重启即可，数据不会丢失，高可用文档正在编写，很快推出。

#### 10.2 暴露主机端口

POD实例配置中的HostPort和HostIP参数无法用于使用了CNI网络插件的kubernetes集群环境，如果需要暴露容器到主机端口，可以使用NodePort或者HostNetwork。

#### 10.3 CentOS环境路由错误

RHEL/CentOS7 环境中iptables的策略关系，会导致路由通讯错误，需要手动调整iptables的桥接设置：

```Bash
# cat /etc/sysctl.d/k8s.conf
 net.bridge.bridge-nf-call-ip6tables = 1
 net.bridge.bridge-nf-call-iptables = 1
```

#### 10.4 Token丢失

Master节点部署完成之后，会输出一个token用于minion节点的配置链接，不过这个token没有很方便的查看方式，导致此日志输出关闭后，没有token无法join minion节点，可以通过下述方式查看token：

```Bash
kubectl -n kube-system get secret clusterinfo -o yaml | grep token-map | awk '{print $2}' | base64 --decode | sed "s|{||g;s|}||g;s|:|.|g;s/\"//g;" | xargs echo
```

建议提前使用`kubeadm token`命令生成token，然后在执行`kubeadm init`和`kubeadm join`的使用通过`--token`指定token。

#### 10.5 Vagrant中主机名的问题

如果使用Vagrant虚拟化环境部署kubernetes，首先得确保`hostname -i`能够获取正确的通讯IP，默认情况下，如果`/etc/hosts`中未配置主机名与IP的对应关系，kubelet会取第一个非lo网卡作为通讯入口，若这个网卡不做了NAT桥接的网卡，那安装就会出现问题。

#### 10.6 Master节点上kubeconfig未加载的问题

kubectl默认应该是会加载配置文件：`/etc/kubernetes/admin.conf`，但是本次部署后，kubectl未加载该配置文件，可以添加一条环境变量：export KUBECONFIG=/etc/kubernetes/admin.conf，问题解决。

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2017/09/20/Deploy-kubernetes1.7.6-with-kubeadm/。