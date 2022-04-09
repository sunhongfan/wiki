## 背景

通过 `Kubeadm` 安装的高可用集群, 默认 Etcd 是内置在集群内部, 由于 etcd 保存了整个集群的状态元数据信息因此, 为了可方便扩展副本和管理数据特此想把它迁移在独立集群外部来运行.

etcd 集群官方推荐 3.5.7 个节点, 因为`leader` 的选举需要集群成员半数以上得票. 也就是说 3 个节点最多支持一个节点宕机.

### 默认集群内的 etcd

这里拿 master 节点的 etcd 配置来说明一下.

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/etcd.advertise-client-urls: https://32.16.5.22:2379
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://32.16.5.22:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://32.16.5.22:2380
    - --initial-cluster=kubernetes-master01=https://32.16.5.21:2380,kubernetes-master02=https://32.16.5.22:2380
    - --initial-cluster-state=existing
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://32.16.5.22:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://32.16.5.22:2380
    - --name=kubernetes-master02
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    image: registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.1-0
    imagePullPolicy: IfNotPresent
```

由于 k8s 为了安全性启用了 `https` 认证, 可以看到用于服务端证书, 以及集群之间通信(peer)证书, 此外集群内有3个成员,分别是每个 master 节点各一个

由于 Kubernetes 内所有和 etcd 交互的只有 api-server, 再来大概看一下它的定义:

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 32.16.5.22:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=32.16.5.22
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

这里列出几个相关的参数说明:

1. `--advertise-address`: 指定 api-server 的监听地址,端口为 6443
2. `--etcd-cafile`: 指定 etcd ca证书,用于验证服务端证书.
3. `--etcd-certfile`: 指定连接 etcd 使用的证书
4. `--etcd-keyfile`: 指定连接 etcd 使用的key
5. `--etcd-servers`: **这里比较重要，指明连接 etcd 的地址, api-server 默认连接本地的 etcd, 如果这个节点的 etcd 故障,则对应节点 api-server 也就会有问题**

### 外部 etcd 构建

**前提准备**

- 采用 `docker` 的方式运行 3 节点的集群.
- 使用 `cfssl` 生成自签名证书
- 使用 `docker-compose` 管理 etcd 容器
- 使用 `etcdctl` etcd 客户端工具

#### 初始化 etcd 所需证书.

1. 生成 ca 配置信息.

```bash
# 可以定义多个profiles,分别指定不同的过期时间,使用场景等参数,后续签名证书时使用某个profile;
# signing: 表示该证书可用于签名其它证书,生成的ca.pem证书中的CA=TRUE;
# server auth: 表示client 可以用该CA 对server 提供的证书进行校验;
# client auth: 表示server 可以用该CA 对client 提供的证书进行验证。

$ cat ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
```

2. 生成私有 ca 证书签名请求

```bash
# CN: Common Name, kube-apiserver 从证书中提取该字段作为请求的用户名(User Name);浏览器使用该字段验证网站是否合法;
# O: Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组(Group)；

$ cat ca-csr.json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
           "C": "CN",
           "L": "Beijing",
           "ST": "Beijing",
           "O": "k8s",
           "OU": "System"
        }
    ]
}
```

3. 生成 ca 证书, 私钥

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

4. 私用私有 ca 为 etcd 签发证书和私钥

```bash
# 注意 hosts 字段自行根据情况修改.
$ cat etcd-csr.json
{
    "CN": "etcd",
    "hosts": [
    "kubernetes-*",
    "127.0.0.1",
    "32.16.5.31",
    "32.16.5.32",
    "32.16.5.33"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
           "O": "k8s",
           "OU": "System"
        }
    ]
}

# 签发
$ cfssl gencert -ca ca.pem -ca-key ca-key.pem -config ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

5. 相关证书如下:

```bash
pki/
├── ca-config.json
├── ca.csr
├── ca-csr.json
├── ca-key.pem
├── ca.pem
├── etcd.csr
├── etcd-csr.json
├── etcd-key.pem
└── etcd.pem
```

### 将集群内 etcd 数据恢复到集群节点的各 etcd 数据目录中.

1. 在 master 节点上 **备份** etcd 数据

```bash
$ etdctl snapshot save /tmp/etcd_snap.db --endpoints http://POD_IP:2379
```

2. copy 到新的集群各上

3. 恢复数据到新的集群节点上

```bash
# 这里每个节点都要操作, 注意替换 name 和 --initial-advertise-peer-urls 参数
$ etcdctl snapshot restore etcd_snap.db \
--name etcd-01 \
--initial-cluster etcd-01=https://32.16.5.31:2380,etcd-02=https://32.16.5.32:2380,etcd-03=https://32.16.5.33:2380 \
--initial-cluster-token my-etcd-token \
--initial-advertise-peer-urls https://32.16.5.31:2380 \
--data-dir=/data/docker/etcd
```

### 构建集群

这里为了精简篇幅, 只列出一个, 各节点修改字段如下:

1. `container_name`: 容器名称
2. `--name`: etcd 成员名称
3. `--initial-advertise-peer-urls`: 集群间通信地址
4. `--listen-client-urls`: 客户端通信地址

```yaml
$ cat etcd/docker-compose.yaml
version: "3.8"
networks:
  etcd:
services:
  node1:
    image: quay.io/coreos/etcd:v3.5.2
    container_name: 1.kubernetes-node01-etc
    ports:
      - "2379:2379"
      - "2380:2380"
    volumes:
      - /etc/localtime:/etc/localtime
      - /data/docker/etcd:/etcd-data
      - /root/etcd/pki:/etcd-ssl
    command: >
         /usr/local/bin/etcd
         --data-dir=/etcd-data
         --name etcd-01
         --cert-file=/etcd-ssl/etcd.pem
         --key-file=/etcd-ssl/etcd-key.pem
         --peer-cert-file=/etcd-ssl/etcd.pem
         --peer-key-file=/etcd-ssl/etcd-key.pem
         --trusted-ca-file=/etcd-ssl/ca.pem
         --peer-trusted-ca-file=/etcd-ssl/ca.pem
         --initial-advertise-peer-urls https://32.16.5.31:2380
         --listen-peer-urls https://0.0.0.0:2380
         --advertise-client-urls https://32.16.5.31:2379
         --listen-client-urls https://0.0.0.0:2379
         --initial-cluster etcd-01=https://32.16.5.31:2380,etcd-02=https://32.16.5.32:2380,etcd-03=https://32.16.5.33:2380
         --initial-cluster-state new
         --initial-cluster-token my-etcd-token
    networks:
      - etcd
```

#### 验证集群状态:

1. 查看集群成员列表

```bash
$ export ETCDCTL_API=3

$ alias etcdctl='etcdctl --endpoints=https://32.16.5.31:2379 --cacert=/root/etcd/pki/ca.pem --cert=/root/etcd/pki/etcd.pem --key=/root/etcd/pki/etcd-key.pem

$ etcdctl member list
e9bccbe5a8bcf85, started, etcd-03, https://32.16.5.33:2380, https://32.16.5.33:2379
a652a18ebab003ac, started, etcd-01, https://32.16.5.31:2380, https://32.16.5.31:2379
f978567486e9e245, started, etcd-02, https://32.16.5.32:2380, https://32.16.5.32:2379
```

2. 查看 etcd endpoint 状态:

```bash
$ alias etcdctl='etcdctl --endpoints=https://32.16.5.31:2379,https://32.16.5.32:2379,https://32.16.5.33:2379 --cacert=/root/etcd/pki/ca.pem --cert=/root/etcd/pki/etcd.pem --key=/root/etcd/pki/etcd-key.pem'

$  etcdctl endpoint status --write-out=table
+-------------------------+------------------+---------+---------+-----------+-----------+------------+
|        ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+-------------------------+------------------+---------+---------+-----------+-----------+------------+
| https://32.16.5.31:2379 | a652a18ebab003ac |   3.5.2 |   21 MB |      true |         2 |      58210 |
| https://32.16.5.32:2379 | f978567486e9e245 |   3.5.2 |   21 MB |     false |         2 |      58210 |
| https://32.16.5.33:2379 |  e9bccbe5a8bcf85 |   3.5.2 |   21 MB |     false |         2 |      58210 |
+-------------------------+------------------+---------+---------+-----------+-----------+------------+
```

3. 查看 etcd endpoint 健康状态

```bash
$ alias etcdctl='etcdctl --endpoints=https://32.16.5.31:2379,https://32.16.5.32:2379,https://32.16.5.33:2379 --cacert=/root/etcd/pki/ca.pem --cert=/root/etcd/pki/etcd.pem --key=/root/etcd/pki/etcd-key.pem'

$ etcdctl  endpoint health
https://32.16.5.31:2379 is healthy: successfully committed proposal: took = 1.776694ms
https://32.16.5.33:2379 is healthy: successfully committed proposal: took = 2.0682ms
https://32.16.5.32:2379 is healthy: successfully committed proposal: took = 2.179021ms
```

### 修改 master 节点使用新的 etcd 集群.

1. 备份原本清单文件

```bash
$ mv /etc/kubernetes/manifests /etc/kubernetes/manifests_bak
$ mkdir /etc/kubernetes/manifests
```

2. 修改 `kube-apiserver` 使用新的清单文件启动 `static pod`

```yaml
# 这里主要替换了连接etcd证书和后端地址.
# 相关改动在 366-169 行
$ vim apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 32.16.5.21:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=32.16.5.21
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/pki/ca.pem
    - --etcd-certfile=/etc/kubernetes/pki/etcd/pki/etcd.pem
    - --etcd-keyfile=/etc/kubernetes/pki/etcd/pki/etcd-key.pem
    - --etcd-servers=https://32.16.5.31:2379,https://32.16.5.32:2379,https://32.16.5.33:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
....
```

3. 恢复 **kube-controller-manager** **kube-scheduler** 配置

```bash
$ cp manifests_bak/{kube-controller-manager.yaml,kube-scheduler.yaml} manifests/
```

> 如过有多个 master 的话, 每个 master 节点均需要修改.

4. 验证集群是否正常. 这里需要注意的是, 重启kube-system下的相关组件后, 确认每个 master 节点上的组件工作正常.

#### TODO

恢复集群后,遇到一个问题:

1. kubectl apply 执行速度正常.
2. kubectl delete 执行速度比较慢, 删除一个 Pod 需要 30 秒, 其他资源则不会。
3. 重启 master 各组件,这里一定要是移除 manifests 文件夹, 不然 delete 静态 kube-system pod 并不会产生重启效果
4. 在重启后恢复正常, 暂时没有找到有力证明是具体什么原因导致, 初步推测是因为没有同时重新加载 master 组件导致.
