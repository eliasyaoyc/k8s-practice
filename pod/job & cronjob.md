Deployment、StatefulSet 以及DaemonSet 都是长作业任务，除非出错或者停止否则容器进程会一直保持在 running 状态

离线业务或者叫做 Batch Job （计算业务），计算完就退出。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
# 永远不会被重启即时失败但是会重新创建OnFailure 失败了会重启，而在 Deployment 对像里， restartPolicy至被允许设置为 Always
      restartPolicy: Never 
  backoffLimit: 4
```



Job Controller 对并行作业的控制方法

* spec.parallelism：一个job 在任意事件最多可以启动多少个pod同时运行
* spec.completions：至少要完成的pod数目，即job的最小完成数

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  parallelism: 2 # 最大并行数 2个
  completions: 4 # 最小完成数 4
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc
        command: ["sh", "-c", "echo 'scale=5000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```

```bash
$ kubectl create -f job.yaml

$ kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
pi-5mt88   1/1       Running   0          6s
pi-gmcq5   1/1       Running   0          6s

#而在 40 s 后，这两个 Pod 相继完成计算。这时我们可以看到，每当有一个 Pod 完成计算进入 Completed 状态时，就会有一个新的 Pod 被自动创建出来，并且快速地从 Pending 状态进入到 ContainerCreating 状态

$ kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
pi-gmcq5   0/1       Completed   0         40s
pi-84ww8   0/1       Pending   0         0s
pi-5mt88   0/1       Completed   0         41s
pi-62rbt   0/1       Pending   0         0s

$ kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
pi-gmcq5   0/1       Completed   0         40s
pi-84ww8   0/1       ContainerCreating   0         0s
pi-5mt88   0/1       Completed   0         41s
pi-62rbt   0/1       ContainerCreating   0         0s

# 全部完成
$ kubectl get pods 
NAME       READY     STATUS      RESTARTS   AGE
pi-5mt88   0/1       Completed   0          5m
pi-62rbt   0/1       Completed   0          4m
pi-84ww8   0/1       Completed   0          4m
pi-gmcq5   0/1       Completed   0          5m

$ kubectl get job
NAME      DESIRED   SUCCESSFUL   AGE
pi        4         4            5m
```

**Job Controller 的原理：**

* Job Controller 控制的对象是 pod
* Job Controller 在控制循环中进行的调谐（reconcile），是根据实际在 running 状态pod的数目、已经成功退出的pod 数目，以及parallelism、completions 参数的值共同计算出在这个周期里，应该创建或删除的pod数目，然后调用api 来执行操作



**外部管理器+Job 模板**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM  # $ITEM 变量
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
      restartPolicy: Never
```

```bash
# 同一模板不同 job的yaml 生成
$ mkdir ./jobs
$ for i in apple banana cherry
do
  cat job-tmpl.yaml | sed "s/\$ITEM/$i/" > ./jobs/job-$i.yaml
done

$ kubectl create -f ./jobs
$ kubectl get pods -l jobgroup=jobexample
NAME                        READY     STATUS      RESTARTS   AGE
process-item-apple-kixwv    0/1       Completed   0          4m
process-item-banana-wrsf7   0/1       Completed   0          4m
process-item-cherry-dnfu9   0/1       Completed   0          4m
```



**拥有固定任务数目的并行 Job。**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-1
spec:
  completions: 8
  parallelism: 2
  template:
    metadata:
      name: job-wq-1
    spec:
      containers:
      - name: c
        image: myrepo/job-wq-1
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job1
      restartPolicy: OnFailure
```



**定时任务**

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

由于定时任务的特殊性，很可能某个 Job 还没有执行完，另外一个新 Job 就产生了。

这时候，你可以通过 **spec.concurrencyPolicy** 字段来定义具体的处理策略。比如：

* concurrencyPolicy=Allow，这也是默认情况，这意味着这些 Job 可以同时存在；
* concurrencyPolicy=Forbid，这意味着不会创建新的 Pod，该创建周期被跳过；
* concurrencyPolicy=Replace，这意味着新产生的 Job 会替换旧的、没有执行完的 Job。