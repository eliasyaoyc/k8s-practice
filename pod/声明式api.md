```bash
$ kubectl create -f xxx.yaml # 命令式创建pod
$ kubectl replace -f xxx.yaml # 命令式的修改 pod

$ kubectl apply -f xxx.yaml # 声明式创建pod，更新也是此命令
```

综上所述可以得知，kubectl replace 的执行过程，就是使用新的 YAML 文件中的api对象，替换原有的api对象；而 kubectl apply 则是执行了一个对原有api对象的 patch 操作

> 相似的 kubectl set image 和 kubectl edit 也是对已有api对象的修改

更进一步地，这意味着 kube-apiserver 在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），一次能处理多个写操作，并且具备 Merge 能力。

**Envoy 原理详解**：

> Envoy 作为sidecar ，运行在每一个被治理的应用pod 中，pod 里的所有容器共享一个 network namespace。所以，envoy 容器就能通过配置 pod 里面的iptables规则，把整个pod的进出流量接管下来

问题是：Istio 需要在每一个pod 中安装一个 Envoy 容器，怎么做到无感的？

> Dynamic Admission Control 也叫做 Initializer，这是 k8s 提供的一个热插拔式的 Admission 机制，可以让我们提交一个pod 或者任何一个api对象的时候，可以根据我们自定义的方式进行对pod 或 api 对象的初始化。
>
> 举个例子：
>
> ```yaml
> # 我们编写的 pod
> apiVersion: v1
> kind: Pod
> metadata:
>   name: myapp-pod
>   labels:
>     app: myapp
> spec:
>   containers:
>   - name: myapp-container
>     image: busybox
>     command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
>     
> # 提交后，istio 会在这个api对象自动加上 envoy 容器的配置
> apiVersion: v1
> kind: Pod
> metadata:
>   name: myapp-pod
>   labels:
>     app: myapp
> spec:
>   containers:
>   - name: myapp-container
>     image: busybox
>     command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
>   - name: envoy
>     image: lyft/envoy:845747b88f102c0fd262ab234308e9e22f693a1
>     command: ["/usr/local/bin/envoy"]
>     ...    
> ```

问题：Istio 是怎么编写这个 Initializer 的呢？

* 首先 istio 会将这个Envoy 容器本身的定义，以configmap 的方式保存在 k8s中 叫做 envoy-initializer

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: envoy-initializer
  data:
    config: |
      containers:
        - name: envoy
          image: lyft/envoy:845747db88f102c0fd262ab234308e9e22f693a1
          command: ["/usr/local/bin/envoy"]
          args:
            - "--concurrency 4"
            - "--config-path /etc/envoy/envoy.json"
            - "--mode serve"
          ports:
            - containerPort: 80
              protocol: TCP
          resources:
            limits:
              cpu: "1000m"
              memory: "512Mi"
            requests:
              cpu: "100m"
              memory: "64Mi"
          volumeMounts:
            - name: envoy-conf
              mountPath: /etc/envoy
      volumes:
        - name: envoy-conf
          configMap:
            name: envoy
  ```

  > Initializer 要做的工作，就是把这部分 Envoy 相关的字段，自动添加到用户提交的 Pod 的 API 对象里。可是，用户提交的 Pod 里本来就有 containers 字段和 volumes 字段，所以 Kubernetes 在处理这样的更新请求时，就必须使用类似于 git merge 这样的操作，才能将这两部分内容合并在一起。
  >
  > 所以说，在 Initializer 更新用户的 Pod 对象的时候，必须使用 PATCH API 来完成。而这种 PATCH API，正是声明式 API 最主要的能力。

* 将一个编写好的 Initializer，作为一个pod 部署在 k8s中

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      app: envoy-initializer
    name: envoy-initializer
  spec:
    containers:
      - name: envoy-initializer
        image: envoy-initializer:0.0.1 # 实现编写好的 自定义控制器（custom controller）
        imagePullPolicy: Always
  ```

  > 这个控制器原理就是，死循环检查pod 是否添加过 envoy 容器，没有的话就添加进行initializer 操作。
  >
  > initializer 操作就是往这个pod 中合并保存在 envoy-initializer 这个configmap 里的数据
  >
  > k8s api 库里 提供了 TwoWayMergePatch 方法，可以用来把新旧两个pod 对象生成一个 TwoWayMergePatch ，然后发送给 k8s api server

