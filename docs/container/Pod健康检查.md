## 如何保持 Pod 健康

一个通过控制器管理的 Pod 如(rs), 控制器会监控 `Pod` 的状态,并在容器中的主进程失败的时候,重新启动它们. 往往有时候即使容器中的 `init process` 没有崩溃,应用程序也无法正常提供工作, 比如 nginx 容器的网页没有准备好, java 程序发生内存泄漏但是却不会停止 JVM。

### 探针机制

**K8s 中 2 种探针类型**

1. `liveness probe` 用来检查容器主进程是否还在正常运行, 可以为 Pod 中的每个容器单独指定此探针. 如何探测失败则会定期执行探针并重新启动容器.
2. `readiness probe` 如果不定义的话将会马上成为 `svc` 的端点, 如果应用程序需要一点时间做初始化操作后才能正常响应用户请求,当探测成功后才会被加入到后端端点中来提供服务.

**每个类型的探针对应 3 种探测容器的机制**

1. `HTTP GET`: 对容器的 `IP:PORT/URI` 执行 http get 请求, 如果返回响应正常(2xx,3xx) 则探测成功.
2. `TCP`: 对容器指定的端口建立 TCP 连接, 如果建立成功则探测成功.
3. `Exec`: 在容器内执行命令, 检查命令的退出状态码是否为 0

#### liveness probe demo

在 Kubernetes 中一定要定义一个存活性探针, 没有探针的话, Kubernetes 无法知道应用是否还存活, 只要进程存在 Kubernetes 就会认为容器是健康的.

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: kubernetes-node01
  containers:
  - name: kubia
    image: luksa/kubia
    ports:
    - containerPort: 8080
      protocol: TCP
    livenessProbe:
      initialDelaySeconds: 5   # 在第一次探测前等待 5 秒钟
      httpGet:
        port: 8080
        path: /
```

> 一定要检查应用程序内部,而没有任何外部的影响,比如服务器无法连接到后端的数据库时, 前端的存活性探针不应该返回失败. 重启 web 并不会解决问题,而且会导致反复重启前端 Pod 直到数据库恢复

> 当探测失败后会重新启动容器,如果想通过 `logs` 命令查看 Pod 的日志,只能查看当前的,如果想要查看前一个容器为什么终止时可以通过 `logs mypod --previous`
