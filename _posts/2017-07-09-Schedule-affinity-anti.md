---
layout: post
title:  深入kubernetes调度之Affinity
categories: [docker,Cloud]
date: 2017-07-09 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>前边讲了Taints和Tolerations的调度策略，可以满足一些需求场景，但是基于Taints和Tolerations的调度还是毕竟“生硬”，并且也不够灵活，例如：POD的多实例尽量分布到不同的Node节点、POD_A尽量调度到POD_B所在的Node节点等，此时我们就需要Affinity（亲和性）调度策略。

本文主要介绍kubernetes的中调度算法中的Node affinity和Pod affinity用法，实际上是对前文提到的优选策略中的`NodeAffinityPriority`策略和`InterPodAffinityPriority`策略的具体应用。

### 1 Affinity

亲和性策略（Affinity）能够提供比`NodeSelector`或者Taints更灵活丰富的调度方式，例如：

- 丰富的匹配表达式（In, NotIn, Exists, DoesNotExist. Gt, and Lt）
- 软约束和硬约束（Required/Preferred）
- 以节点上的其他Pod作为参照物进行调度计算

亲和性策略分为`NodeAffinityPriority`策略和`InterPodAffinityPriority`策略。

### 2 Node affinity

据官方说法未来`NodeSeletor`策略会被废弃，由`NodeAffinityPriority`策略中`requiredDuringSchedulingIgnoredDuringExecution`替代。

`NodeAffinityPriority`策略和`NodeSelector`一样，通过Node节点的Label标签进行匹配，匹配的表达式有：`In, NotIn, Exists, DoesNotExist. Gt, and Lt`。

说这么多，似乎也没说清楚，不如举个栗子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-example
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - k8s-node01
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: kubernetes.io/hostname
            operator: NotIn
            values:
            - k8s-master01
            - k8s-master02
  containers:
  - name: example
    image: gcr.io/google_containers/pause:2.0
```

定义描述：

1. `requiredDuringSchedulingIgnoredDuringExecution`，硬约束，一定要满足，效果同`NodeSelector`，Pod只能调度到具有`kubernetes.io/hostname=k8s-node01`标签的Node节点。
2. `preferredDuringSchedulingIgnoredDuringExecution`，软约束，不一定满足，k8s调度会尽量不调度Pod到具有`kubernetes.io/hostname=k8s-node01`或`kubernetes.io/hostname=k8s-node02`标签的Node节点。
3. `nodeSelectorTerms`可以定义多条约束，只需满足其中一条。
4. `matchExpressions`可以定义多条约束，必须满足全部约束。

### 3 Pod affinity

`InterPodAffinityPriority`策略有`podAffinity`和`podAntiAffinity`两种配置方式。

`InterPodAffinityPriority`是干嘛的呢？简单来说，就说根据Node上运行的Pod的Label来进行调度匹配的规则，匹配的表达式有：`In, NotIn, Exists, DoesNotExist`，通过该策略，可以更灵活地对Pod进行调度，例如：将多实例的Pod分散到不通的Node、尽量调度A-Pod到有B-Pod运行的Node节点上等等。另外与Node-affinity不同的是：该策略是依据Pod的Label进行调度，所以会受到namespace约束。

Pod亲和性调度请使用：`podAffinity`，非亲和性调度请使用：`podAntiAffinity`。

说这么多，似乎也没说清楚，不如举个栗子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-example
  labels:
    app: test-pod
spec:
  affinity:
    podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - test-pod
            topologyKey: kubernetes.io/hostname
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: 
                - test-pod
          topologyKey: "kubernetes.io/hostname"
  containers:
  - name: example
    image: gcr.io/google_containers/pause:2.0
```

定义描述：

1. `requiredDuringSchedulingIgnoredDuringExecution`，硬约束，一定要满足，Pod的亲和性调度必须要满足后续定义的约束条件。
2. `preferredDuringSchedulingIgnoredDuringExecution`，软约束，不一定满足，Pod的亲和性调度会尽量满足后续定义的约束条件。
3. `topologyKey`用于定于调度时作用于特指域，这是通过Node节点的标签来实现的，例如指定为`kubernetes.io/hostname`，那就是以Node节点为区分范围，如果指定为`beta.kubernetes.io/os`,则以Node节点的操作系统类型来区分。

`topologyKey`的使用约束：

1. 除了`podAntiAffinity`的`preferredDuringSchedulingIgnoredDuringExecution`，其他模式下的topologyKey不能为空。
2. 如果`Admission Controller`中添加了`LimitPodHardAntiAffinityTopology`，那么`podAntiAffinity`的`requiredDuringSchedulingIgnoredDuringExecution`被强制约束为`kubernetes.io/hostname`。
3. 如果`podAntiAffinity`的`preferredDuringSchedulingIgnoredDuringExecution`中的topologyKey为空，则默认为适配`kubernetes.io/hostname`,`failure-domain.beta.kubernetes.io/zone`,`failure-domain.beta.kubernetes.io/region`。

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2017/07/09/Schedule-affinity-anti/。
