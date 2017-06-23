---
layout: post
title:  深入kubernetes调度之Taints和Tolerations
categories: [docker,Cloud]
date: 2017-06-08 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>Kubernetes中的调度策略比较多，每个策略都有其比较适用的需求场景。

本文主要介绍kubernetes的中调度算法中的Taints和Tolerations用法，实际上是对PodToleratesNodeTaints策略和TaintTolerationPriority策略的具体应用。先从中文字面意思上理解下这两个词语：Taints（污点），Tolerations（容忍）。

### 1 Dashboard与Master那些事

部署kube-dashboard的时候，在yaml文件中有这样一段定义：

```yaml
...
# Comment the following annotation if Dashboard must not be deployed on master
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
...
```

注释`Comment the following annotation if Dashboard must not be deployed on master`说的很清楚：如果Dashboard不想部署在master节点上，那就注释掉下下边的这段annotations定义。

有的同学在部署Dashboard的时候就疑惑了，说注释写的怎么跟annotations定义是相反的，annotations的定义中说的是：如果Node节点上定义有key为dedicated，并且value为master的annotations，那就不调度Pod，如果注释掉这段代码，那岂不是把这个约束去掉了？

单从annotations定义的字面意思来理解，似乎的确是这种说法，但是事实上，这是忽略一件事情，那就是`Taints`和`Tolerations`。

来看下Master节点上的annotations定义：

```yaml
annotations:
  scheduler.alpha.kubernetes.io/taints: '[{"key":"dedicated","value":"master","effect":"NoSchedule"}]'
  volumes.kubernetes.io/controller-managed-attach-detach: "true"
```

可见，Master节点上定义了`Taints`，`Taints`是用来干什么的？它表达的是一个含义：此节点已被key=value污染，Pod调度不允许（PodToleratesNodeTaints策略）或尽量不（TaintTolerationPriority策略）调度到此节点，除非是能够容忍（Tolerations）key=value污点的Pod。

Master节点上定义了`Taints`，声明：如果不是带有`Tolerations`定义为`[{"key":"dedicated","value":"master","effect":"NoSchedule"}]`的Pod，不允许调度到Master节点，PS：`operator`的默认值为`Equal`，所以可以不必显示声明。

这下明白了，Master上定义一个污点A（Taints）禁止Pod调度，Dashboard的yaml里定义一个容忍（Tolerations）允许A污点，所以可以调度到Master节点上。

### 2 Taints和Tolerations

如上所述，Taints和Tolerations和搭配使用的，Taints定义在Node节点上，声明污点及标准行为，Tolerations定义在Pod，声明可接受得污点。

可以在命令行为Node节点添加Taints：

```bash
kubectl taint nodes node1 key=value:NoSchedule
```

也可以直接在node的定义中修改annotations：

```yaml
annotations:
  scheduler.alpha.kubernetes.io/taints: '[{"key":"xxx","operator":"Equal","value":"yyy","effect":"NoSchedule"}]'
```

`operator`可以定义为：

- Equal     表示key是否等于value，默认
- Exists    表示key是否存在，此时无需定义value

`effect`可以定义为：

- NoSchedule            表示不允许调度，已调度的不影响
- PreferNoSchedule      表示尽量不调度
- NoExecute             表示不允许调度，已调度的在`tolerationSeconds`（定义在Tolerations上）后删除

Node和Pod上都可以定义多个Taints和Tolerations，Scheduler会根据具体定义进行筛选，Node筛选Pod列表的时候，会保留Tolerations定义匹配的，过滤掉没有Tolerations定义的，过滤的过程是这样的：

- 如果Node中存在一个或多个影响策略为`NoSchedule`的Taint，该Pod不会被调度到该Node
- 如果Node中不存在影响策略为`NoSchedule`的Taint，但是存在一个或多个影响策略为`PreferNoSchedule`的Taint，该Pod会尽量不调度到该Node
- 如果Node中存在一个或多个影响策略为`NoExecute`的Taint，该Pod不会被调度到该Node，并且会驱逐已经调度到该Node的Pod实例

