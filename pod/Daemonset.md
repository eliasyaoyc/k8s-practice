DaemonSet 是让 k8s 运行一个  daemon pod，有如下特征：

* 这个pod 运行在 k8s 集群里的每一个node 上
* 每个节点上只有一个这样的pod实例
* 当有新的节点加入k8s集群后，该pod 会自动的在新节点上创建出来；而当旧节点被删除的时候，它上面的pod 也相应的被回收掉

> **例子**
>
> * 各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络
> * 各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录
> * 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。

```yaml
// 这是个 daemonset 管理一个 fluentd-elasticsearch 镜像的pod，通过 fluentd 将 docker 容器里的日志转发到 es 上
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

Daemonset 保证每个node上有且只有一个被管理的pod

* 没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；
* 有这种 Pod，但是数量大于 1，那就说明要把多余的 Pod 从这个 Node 上删除掉；
* 正好只有一个这种 Pod，那说明这个节点是正常的。



在指定node上创建新的pod：nodeAffinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: # 表示这个nodeAffinity 必须在每次调度的时候考虑这个pod
        nodeSelectorTerms:
        - matchExpressions:  # 这个pod 只允许跑在 metadata.name 是 node-geektime 的节点上
          - key: metadata.name
            operator: In
            values:
            - node-geektime
```

> Daemonset 会自动的给pod加上 nodeAffinity 的定义，不会修改yaml 文件，而是在向k8s发起请求之前吗，直接修改根据模板生成的pod对象

Daemonset 还会自动的给pod 加上 toleration 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations: # 表示 “容忍” 所有被标记为 unschedulable “污点” 的 Node； “容忍” 的效果是允许调度。
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```

正常情况下，被标记了 unschedulable“污点”的 Node，是不会有任何 Pod 被调度上去的（effect: NoSchedule）。可是，DaemonSet 自动地给被管理的 Pod 加上了这个特殊的 Toleration，就使得这些 Pod 可以忽略这个限制，继而保证每个节点上都会被调度一个 Pod。当然，如果这个节点有故障的话，这个 Pod 可能会启动失败，而 DaemonSet 则会始终尝试下去，直到 Pod 启动成功。

> 例子：
>
> 假如当前 DaemonSet 管理的，是一个网络插件的 Agent Pod，那么你就必须在这个 DaemonSet 的 YAML 文件里，给它的 Pod 模板加上一个能够“容忍”node.kubernetes.io/network-unavailable“污点”的 Toleration。正如下面这个例子所示：
>
> ```yaml
> ...
> template:
>     metadata:
>       labels:
>         name: network-plugin-agent
>     spec:
>       tolerations:
>       - key: node.kubernetes.io/network-unavailable
>         operator: Exists
>         effect: NoSchedule
> ```
>
> 在 Kubernetes 项目中，当一个节点的网络插件尚未安装时，这个节点就会被自动加上名为node.kubernetes.io/network-unavailable的“污点”。
>
> **而通过这样一个 Toleration，调度器在调度这个 Pod 的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的 Agent 组件调度到这台机器上启动起来。**

