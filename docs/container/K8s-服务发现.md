## 服务发现

pod 通常需要对来自集群内部其他 pod，以及来自集群外部的客户端的 HTTP 请求做出响应。

因为 Pod 的生命周期是短暂的,并且没有一个可预知的访问地址, 服务发现的作用是将一个或一组具有相同特征的 Pod 统一提供入口端点, 来实现这组 Pod 的服务发现和负载均衡。

### Service

Service 作用就是为一组功能相同的 Pod 提供单一并且稳定的接入点资源, 当服务存在时, IP地址和端口就不会发送改变, 这些请求会被路由到 svc 资源后端的任意 Pod 来提供响应.

> svc 其实就是一个 vip 概念, 本质上是 iptables ,或者 ipvs 的规则.

```
                        +--------------------+
                        |   svc: app:nginx   |
                        +----------+---------+
                        |          |         |
                        |          |         |
                     +-----+    +-----+   +-----+
                     | pod |    | Pod |   | Pod |
                     +-----+    +-----+   +-----+
```

#### svc yaml

创建一个 nginx-svc 的服务, 它将接收集群内部来自 80 端口的的请求,并转发到具有标签为 app=nginx 的 Pod 上的 8080

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  sessionAffinity: None    # 此项是默认值,如果需要同一个客户端的请求始终分配给相同的 Pod ,此项需要设置为 ClientIP
  type: ClusterIP          # 此项是默认的,可选值还有 NodePort, LoadBalancer。
#  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80               # 该 svc 可用的端口
    targetPort: 8080       # 服务将连接转发到容器的端口
#    NodePort: 32100
```

>  如需要支持集群外部访问需要通过 NodePort/LoadBalancer 类型, LoadBalancer 需要额外基础服务支持.

**集群内部访问方式:**

1. 通过 `svc` 在集群内部暴露的 IP 地址.
2. 直接访问服务名称,依靠集群内 DNS 解析. `dig @10.96.0.10 nginx-svc.default.svc.cluster.local.`
3. 通过环境变量访问,同一个 namespace 里的 pod, K8s 会把 service 的信息, 通过环境变量的方式放到 pod 里面. 如: `echo $NGINX_SVC_SERVICE_HOST`

#### Endpoint

有时候希望通过 Kubernetes 服务的特性, 暴露一组外部服务的情况, 不让服务将连接重定向到集群中的Pod, 而是让它从定向到 **外部的 IP 和端口.**

服务并不是和 Pod 直接相连, 而是和 `endpoint` 交互, endpoint 定义了一组服务的 IP 和端口列表, 在 spec 定义的 **Pod 选择器** 就是用于构建对应的 endpoint 资源.

当一个`服务(svc)` 和 `endpoint` 解耦后,我们就可以手动配置和更新它们, 如果创建了一个不包含 Pod 选择器的服务, 将不会创建 endpoint 资源。这时候就可以手动创建对应的 endpoint 资源指定后端列表.

```yaml
# 创建没有选择器的服务
apiversion: v1
kind: Service
metadata:
  name: external-service      # 服务名称必须和 Endpoint 对象的名字匹配
spec:
  ports:
  - port: 80

# 创建对应的 Endpoint 资源
---
apiversion: v1
kind: Endpoints
metadata:
  name: external-service      # Endpoint 的名称必须和服务名称匹配
subsets:
  - addresses:                # 服务将连接重定向到 endpoint 的 ip 地址
    - ip: x.x.x.x
    - ip: x.x.x.x
    ports:                    # endpoint 的目标端口
    - port: 80
```

还可以通过域名去访问外部服务,其本质就是创建了一个 `CNAME` 的别名.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: zg1.0x8.org
  ports:
  - port: 80
```

服务创建完成后, 可以通过 `external-service.default.svc.cluster.local` 来访问服务

```bash
$ curl --dns-servers 10.96.0.10  external-service.default.svc.cluster.local -L
<H1> learn nginx test page </H1>
```

查看其别名记录

```bash
$ dig external-service.default.svc.cluster.local @10.96.0.10

; <<>> DiG 9.16.1-Ubuntu <<>> external-service.default.svc.cluster.local @10.96.0.10
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44183
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 0b8c549950602d53 (echoed)
;; QUESTION SECTION:
;external-service.default.svc.cluster.local. IN A

;; ANSWER SECTION:
external-service.default.svc.cluster.local. 30 IN CNAME zg1.0x8.org.       # CNAME 记录

;; Query time: 4 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Fri Apr 22 15:52:29 CST 2022
;; MSG SIZE  rcvd: 150
```

#### HeadLess 服务

在某些场景不想通过 svc 代理请求到随机的后端的 Pod, 而是想通过服务访问到具体的所有 Pod, **可以通过 DNS 查找发现 Pod IP**

当执行 DNS 查找服务时, DNS 服务器会返回 **单个IP(svc的集群IP)**, 但是如果明确指明不需要提供集群IP, 则 DNS 服务器将返回 Pod IP, 而不是单个服务 IP

DNS 服务器不会返回单个 DNS A 记录，而是返回多个 A 记录，每个记录指向后端的单个 pod IP。客户端因此可以做一个简单的 DNS A 记录查找并获取属于该服务一部分的所有 pod 的IP。客户端可以使用该信息连接到其中的一个、多个或全部。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  clusterIP: None          # 创建 headless 服务的 svc
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 8080
```

> headless 服务仍然提供跨pod的负载平衡，但是通过DNS轮询机制不是通过服务代理

> headless 常用与 sts 控制器配合使用,可以通过 pod_name.svc_name.default.svc.cluster.local 来访问固定的 Pod

### Ingress 服务

一个重要的原因是每个 `LoadBalancer` 服务都需要自己的负载均衡器，以及独有的公有IP地址，而 `Ingress` 只需要一个 **公网IP** 就能为许多服务提供访问。当客户端向 `Ingress` 发送 `HTTP` 请求时, Ingress 会 **根据请求的主机名和路径决定请求转发到的服务** 这都是七层负载均衡器的特性.

`ingress` 工作在七层, 因此也支持 `svc(四层)` 所不支持的功能, 如基于 cookie 的会话亲和性.

```

                                                          +-------+        +-----------------+
                        +----{test.example.com/test}----> |  svc  | <----> | Pod | Pod | Pod |
                        |                                 +-------+        +-----------------+
+--------+    +---------+                                 +-------+        +-----------------+
| 客户端 |<-->| Ingress |----{foo.example.com}----------> |  svc  | <----> | Pod | Pod | Pod |
+--------+    +---------+                                 +-------+        +-----------------+
                        |                                 +-------+        +-----------------+
                        +----{bar.example.com}----------> |  svc  | <----> | Pod | Pod | Pod |
                                                          +-------+        +-----------------+
```

#### 安装

采用 [kubernetes-ingress](https://kubernetes.github.io/ingress-nginx/deploy/) 类似实现的 ingress 控制器还有 `Nginx-ingress` 分为开源版和闭源版

参考: [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/baremetal/#over-a-nodeport-service)

部署模式采用 `ds` + `hostNetwork` 模式部署:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  minReadySeconds: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
    spec:
      dnsPolicy: ClusterFirst
      hostNetwork: true
      nodeSelector:
        isIngress: "true"
      containers:
        ...
        ...
```

Nodeport 模式:

```yaml
# 主要修改 ingress-nginx 的 service 服务为 NodePort 模式
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Cluster
  ports:
  - appProtocol: http
    name: http
    port: 80
    nodePort: 30080
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 443
    nodePort: 30443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: NodePort
```

> 把 `ingress service` 端口进行固定, 81:30080, 443:30443

> 并修改externalTrafficPolicy: Local 为 `externalTrafficPolicy: Cluster`, 否则会出现 ingress 只能访问对应节点的 Pod.

设置默认 IngressClass:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: nginx
  # 新增字段
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

使用 `kubectl get all -n ingress-nginx` 查看 ingress 命名空间所有资源.

#### 创建 ingress 资源

确保 Ingress 控制器工作正常后,即可创建 Ingress 资源,来对一组相同服务的 Pod 提供统一入口端点.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-example
spec:
  replicas: 2
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - "sunhf-web"
  template:
    metadata:
      name: ingress-example-pod
      labels:
        app: sunhf-web
    spec:
      containers:
      - image: luksa/kubia
        name: kubia-ingress-example
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: nginx-http
---
apiVersion: v1
kind: Service
metadata:
  name: svc-ingress-example
spec:
  selector:
    app: sunhf-web
  ports:
  - name: nginx-svc-http
    port: 80
    targetPort: nginx-http
    protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-rs-ingress
#  annotations:
#    nginx.ingress.kubernetes.io/affinity: "cookie"
#    nginx.ingress.kubernetes.io/session-cookie-name: "route"
#    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
#    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
spec:
  ingressClassName: nginx
  rules:
  - host: "kubia.test.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: svc-ingress-example
            port:
              number: 80
```

> 需要增加 hosts 解析: 节点IP   kubia-test.com， 即可通过 curl kubia-test.com:30080 访问.

#### Ingress TLS

在现在几乎所有场景都采用了 `https` 加密传输数据, 要配置 TLS 只需要针对对应的 Ingress 控制器配置, 并不需要针对后端的 Pod 单独配置.因为客户端通过 https 与 ingress 建立连接, 而 ingress 在集群内部通过 http 与后端 Pod 建立连接。

要使控制器能够使用 `TLS` 需要将 **证书和私钥** 附加到 `Ingress`, 证书和私钥必须存储在 `Secret` 中

**1. 创建自签名证书**

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -out test-ingress-tls.crt \
    -keyout test-ingress-tls.key \
    -subj "/CN=kubia.test.com/O=ops-ingress"

$ ls -l test-ingress-tls.*
-rw-r--r-- 1 root root 1188 Apr 24 15:42 test-ingress-tls.crt
-rw------- 1 root root 1704 Apr 24 15:42 test-ingress-tls.key
```

**2. 创建secret**

```bash
$ kubectl create secret tls test-ingress-tls \
    --key test-ingress-tls.key \
    --cert test-ingress-tls.crt

$ kubectl get  secrets test-ingress-tls
NAME               TYPE                DATA   AGE
test-ingress-tls   kubernetes.io/tls   2      50s
```

**3. 创建Ingress**

```yaml
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - kubia.test.com                 # 域名
    secretName: test-ingress-tls     # 证书
  rules:
  - host: "kubia.test.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: svc-ingress-example
            port:
              number: 80
```

**4. 测试访问**

```bash
$ curl https://kubia.test.com:30443

curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

# 使用 -k 忽略证书认证
$ curl https://kubia.test.com:30443 -k
You've hit ingress-example-5b86c8f65c-5hw98
```

