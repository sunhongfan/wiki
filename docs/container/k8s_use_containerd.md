## 背景

我们知道 `kubernetes` 在 `v1.20` 版本宣布了放弃对 `docker` 的支持, 那就需要找替代方案, 和简单了解一些原因.

### Docker-CE

我们现在使用的 Docker 版本都叫 `Docker-ce` 了, 从`Docker 1.11` 版本开始 集成了 `containerd`、`runc` 等多个组件来完成的, 守护进程负责和 `Docker Client` 端交互, 并管理 Docker 镜像和容器。现在的架构中组件 `containerd` 就会负责集群节点上容器的生命周期管理，并向上为 `Docker Daemon` 提供 `gRPC` 接口

那现在的 docker 组件如下:

- dockerd: docker 的守护进程,用于接受客户端请求.
- containerd: 也是守护进程,他的作用在于 **接收 Dockerd 请求**
- containerd-shim-runc-v2: 这个组件是作为`containerd` 的附加组件, 当`containerd` 接收到请求时,并不是直接创建容器的, 因为容器需要一个父进程来收集状态,维持 `stdin` 等 `fd` 打开等工作的，如果`containerd` 挂掉了那全部的容器都会出现问题, shim 就是来规避这个问题的.
- runc: 创建容器是需要做一些 `namespaces` 和 `cgroups` 的配置，以及`挂载 root 文件系统`等操作, 这些操作都遵循一个规范`OCI`, runc 就是它的一个参考实现方案. 然后 shim 调用 runc 来创建容器, runc 操作完成后会退出, 容器的父进程就会变成 shim 它负责管理容器,负责收集容器进程的状态, 上报给 containerd, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理.


docker 创建容器调用链如下:

```
+------------+           +---------+           +-------------+
| docker-cli |---请求--> | dockerd |---请求--> | containerd  |
+------------+           +---------+           +-------------+
                                                      |
                                                      |
                                               +-------------+
                                               | c-shim-runc |
                                               +-------------+
                                                      |
                                                      |
                                               +-------------+
                                               |    runc     |
                                               +-------------+
```

kubernetes 调用 docker 调用链如下:

```
# 当我们在 Kubernetes 中创建一个 Pod 的时候，首先就是 kubelet 通过 CRI 接口调用 dockershim，请求创建一个容器，kubelet 可以视作一个简单的 CRI Client, 而 dockershim 就是接收请求的 Server，不过他们都是在 kubelet 内置的(将在后续版本移除)。dockershim 收到请求后, 转化成 Docker Daemon 能识别的请求, 发到 Docker Daemon 上请求创建一个容器，请求到了 Docker Daemon 后续就是 Docker 创建容器的流程了.

+------------+       +---------+           +-------------+
| dockershim |-----> | dockerd |---请求--> | containerd  |
+------------+--+    +---------+           +-------------+
                |
               gRPC
+------------+  |
|  kubelet   |--+
+------------+
```

相关名词解释:

- **OCI(开放容器标准)**: 其实就是一个标准化的文档,其中定义了容器镜像要长啥样, 即 `ImageSpec`, 容器要需要能接收哪些指令, 这些指令的行为是什么, 即 `RuntimeSpec`. 这里面的大致内容就是”容器”要能够执行 `create`, `start`, `stop`, `delete` 这些命令, 并且行为要规范
- **CRI(容器运行时)**:  单纯是一组 `gRPC` 接口, 一套针对容器操作的接口, 包括创建,启停容器等等; 一套针对镜像操作的接口, 包括拉取镜像删除镜像等; 等等.

### Containerd

可以看到 dockerd 由于不支持 `CRI` 所以需要中间加入一个转发请求的玩意来翻译,多了东西就会增开销,真正容器相关的操作其实 `containerd` 就完全足够了,

而在 `containerd v1.1+` 直接把适配 `CRI` 逻辑作为插件的方式集成到了 `containerd` 主进程中, 所以调用链比较简洁 如下:

```
+----------+           +-----------------------+           +------------+
| kubelet  |---请求--> | [CRI Plug] containerd |---创建--> | conrainers |
+----------+           +-----------------------+           +------------+
```

调用关系显而易见,肯定是 containerd 更胜一筹, 将 `Kubernetes` 的 `CONTAINER-RUNTIME`, 应该更为妥当。

> Kubernetes 社区也做了一个专门用于 Kubernetes 的 CRI 运行时 CRI-O，直接兼容 CRI 和 OCI 规范,不过好像没什么人用, 暂时不做了解

#### Containerd 官方介绍

containerd 是一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性，containerd 可以负责干下面这些事情：

- 管理容器的生命周期（从创建容器到销毁容器）
- 拉取/推送容器镜像
- 存储管理（管理镜像及容器数据的存储）
- 调用 runc 运行容器（与 runc 等容器运行时交互）
- 管理容器网络接口及网络

containerd 采用的也是 C/S 架构，服务端通过 unix domain socket 暴露低层的 gRPC API 接口出去，客户端通过这些 API 管理节点上的容器

### Install Containerd

containerd 需要调用 runc，所以我们也需要先安装 runc，不过 containerd 提供了一个包含相关依赖的压缩包 `cri-containerd-cni-${VERSION}.${OS}-${ARCH}.tar.gz`，可以直接使用这个包来进行安装。首先从 [release 页面](https://github.com/containerd/containerd/releases)下载最新版本的压缩包

```
$ wget https://github.com/containerd/containerd/releases/download/v1.6.2/cri-containerd-1.6.2-linux-amd64.tar.gz
```

可以通过 tar 的 `-t` 选项直接看到压缩包中包含哪些文件：

```
$ tar tf cri-containerd-1.6.2-linux-amd64.tar.gz
etc/crictl.yaml
etc/systemd/
etc/systemd/system/
etc/systemd/system/containerd.service
usr/
usr/local/
usr/local/bin/
usr/local/bin/containerd-shim-runc-v2
usr/local/bin/containerd-shim
usr/local/bin/crictl
usr/local/bin/ctr
usr/local/bin/containerd-shim-runc-v1
usr/local/bin/containerd
usr/local/bin/ctd-decoder
usr/local/bin/critest
usr/local/bin/containerd-stress
usr/local/sbin/
usr/local/sbin/runc
opt/containerd/
opt/containerd/cluster/
opt/containerd/cluster/version
opt/containerd/cluster/gce/
opt/containerd/cluster/gce/cni.template
opt/containerd/cluster/gce/configure.sh
opt/containerd/cluster/gce/env
opt/containerd/cluster/gce/cloud-init/
opt/containerd/cluster/gce/cloud-init/master.yaml
opt/containerd/cluster/gce/cloud-init/node.yaml
```

直接将压缩包解压到系统的各个目录中：

```
$ tar -C / -xzf cri-containerd-1.6.2-linux-amd64.tar.gz
```

containerd 的默认配置文件为 `/etc/containerd/config.toml`，我们可以通过如下所示的命令生成一个默认的配置：

```
$ mkdir -p /etc/containerd
$ containerd config default > /etc/containerd/config.toml
```

containerd 压缩包中包含一个 `etc/systemd/system/containerd.service` 的文件，这样我们就可以通过 systemd 来配置 containerd 作为守护进程运行了，内容如下所示：

```
$ cat /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

这里有两个重要的参数：

-   `Delegate`: 这个选项允许 containerd 以及运行时自己管理自己创建容器的 cgroups。如果不设置这个选项，systemd 就会将进程移到自己的 cgroups 中，从而导致 containerd 无法正确获取容器的资源使用情况。
-   `KillMode`: 这个选项用来处理 containerd 进程被杀死的方式。默认情况下，systemd 会在进程的 cgroup 中查找并杀死 containerd 的所有子进程。KillMode 字段可以设置的值如下。
    -   `control-group`（默认值）：当前控制组里面的所有子进程，都会被杀掉
    -   `process`：只杀主进程
    -   `mixed`：主进程将收到 SIGTERM 信号，子进程收到 SIGKILL 信号
    -   `none`：没有进程会被杀掉，只是执行服务的 stop 命令

我们需要将 KillMode 的值设置为 process，这样可以确保升级或重启 containerd 时不杀死现有的容器。

启动 containerd

```
$ systemctl enable containerd --now
```

启动完成后就可以使用 containerd 的本地 CLI 工具 `ctr` 了，比如查看版本：

```
$ ctr version
Client:
  Version:  v1.6.2
  Revision: de8046a5501db9e0e478e1c10cbcfb21af4c6b2d
  Go version: go1.17.2

Server:
  Version:  v1.6.2
  Revision: de8046a5501db9e0e478e1c10cbcfb21af4c6b2d
  UUID: 332cc05b-35dd-412d-ac67-2e29e153627d
```

##### 配置

默认生成的配置文件 `/etc/containerd/config.toml`：

这个配置文件比较复杂，我们可以将重点放在其中的 `plugins` 配置上面，仔细观察我们可以发现每一个顶级配置块的命名都是 `plugins."io.containerd.xxx.vx.xxx"` 这种形式，每一个顶级配置块都表示一个插件，其中 `io.containerd.xxx.vx` 表示插件的类型，`vx` 后面的 `xxx` 表示插件的 ID，我们可以通过 `ctr` 查看插件列表：

```
$ ctr plugin ls
ctr plugin ls
TYPE                            ID                       PLATFORMS      STATUS
io.containerd.content.v1        content                  -              ok
io.containerd.snapshotter.v1    aufs                     linux/amd64    ok
io.containerd.snapshotter.v1    btrfs                    linux/amd64    skip
io.containerd.snapshotter.v1    devmapper                linux/amd64    error
io.containerd.snapshotter.v1    native                   linux/amd64    ok
io.containerd.snapshotter.v1    overlayfs                linux/amd64    ok
io.containerd.snapshotter.v1    zfs                      linux/amd64    skip
io.containerd.metadata.v1       bolt                     -              ok
io.containerd.differ.v1         walking                  linux/amd64    ok
io.containerd.gc.v1             scheduler                -              ok
io.containerd.service.v1        introspection-service    -              ok
io.containerd.service.v1        containers-service       -              ok
io.containerd.service.v1        content-service          -              ok
io.containerd.service.v1        diff-service             -              ok
io.containerd.service.v1        images-service           -              ok
io.containerd.service.v1        leases-service           -              ok
io.containerd.service.v1        namespaces-service       -              ok
io.containerd.service.v1        snapshots-service        -              ok
io.containerd.runtime.v1        linux                    linux/amd64    ok
io.containerd.runtime.v2        task                     linux/amd64    ok
io.containerd.monitor.v1        cgroups                  linux/amd64    ok
io.containerd.service.v1        tasks-service            -              ok
io.containerd.internal.v1       restart                  -              ok
io.containerd.grpc.v1           containers               -              ok
io.containerd.grpc.v1           content                  -              ok
io.containerd.grpc.v1           diff                     -              ok
io.containerd.grpc.v1           events                   -              ok
io.containerd.grpc.v1           healthcheck              -              ok
io.containerd.grpc.v1           images                   -              ok
io.containerd.grpc.v1           leases                   -              ok
io.containerd.grpc.v1           namespaces               -              ok
io.containerd.internal.v1       opt                      -              ok
io.containerd.grpc.v1           snapshots                -              ok
io.containerd.grpc.v1           tasks                    -              ok
io.containerd.grpc.v1           version                  -              ok
io.containerd.grpc.v1           cri                      linux/amd64    ok
```

顶级配置块下面的子配置块表示该插件的各种配置，比如 cri 插件下面就分为 containerd、cni 和 registry 的配置，而 containerd 下面又可以配置各种 runtime，还可以配置默认的 runtime。比如现在我们要为镜像配置一个加速器，那么就需要在 cri 配置块下面的 `registry` 配置块下面进行配置 `registry.mirrors`：

```
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://bqr1dr1n.mirror.aliyuncs.com"]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
      endpoint = ["https://registry.aliyuncs.com/k8sxio"]
```

-   `registry.mirrors."xxx"`: 表示需要配置 mirror 的镜像仓库，例如 `registry.mirrors."docker.io"` 表示配置 docker.io 的 mirror。
-   `endpoint`: 表示提供 mirror 的镜像加速服务，比如我们可以注册一个阿里云的镜像服务来作为 docker.io 的 mirror。

另外在默认配置中还有两个关于存储的配置路径：

```
root = "/var/lib/containerd"
state = "/run/containerd"
```

其中 `root` 是用来保存持久化数据，包括 Snapshots, Content, Metadata 以及各种插件的数据，每一个插件都有自己单独的目录，Containerd 本身不存储任何数据，它的所有功能都来自于已加载的插件。

而另外的 `state` 是用来保存运行时的临时数据的，包括 sockets、pid、挂载点、运行时状态以及不需要持久化的插件数据。

#### 使用

我们知道 Docker CLI 工具提供了需要增强用户体验的功能，containerd 同样也提供一个对应的 CLI 工具：`ctr`，不过 ctr 的功能没有 docker 完善，但是关于镜像和容器的基本功能都是有的。接下来我们就先简单介绍下 `ctr` 的使用。

#### 镜像操作

**拉取镜像**

拉取镜像可以使用 `ctr image pull` 来完成，比如拉取 Docker Hub 官方镜像 `nginx:alpine`，需要注意的是镜像地址需要加上 `docker.io` Host 地址：

```
$ ctr image pull docker.io/library/nginx:alpine
docker.io/library/nginx:alpine:                                                   resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:bead42240255ae1485653a956ef41c9e458eb077fcb6dc664cbc3aa9701a05ce:    exists         |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:ce6ca11a3fa7e0e6b44813901e3289212fc2f327ee8b1366176666e8fb470f24: done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 11.9s                                                                    total:  8.7 Mi (748.1 KiB/s)
unpacking linux/amd64 sha256:bead42240255ae1485653a956ef41c9e458eb077fcb6dc664cbc3aa9701a05ce...
done: 410.86624ms
```

**列出本地镜像**

```
$ ctr image ls
REF                            TYPE                                                      DIGEST                                                                  SIZE    PLATFORMS                                                                                LABELS
docker.io/library/nginx:alpine application/vnd.docker.distribution.manifest.list.v2+json sha256:bead42240255ae1485653a956ef41c9e458eb077fcb6dc664cbc3aa9701a05ce 9.5 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -

$ ctr image ls -q
docker.io/library/nginx:alpine
```

**重新打标签**

同样的我们也可以重新给指定的镜像打一个 Tag：

```
$ ctr image tag docker.io/library/nginx:alpine harbor.k8s.local/course/nginx:alpine
harbor.k8s.local/course/nginx:alpine

$ ctr image ls -q
docker.io/library/nginx:alpine
harbor.k8s.local/course/nginx:alpine
```

**删除镜像**

不需要使用的镜像也可以使用 `ctr image rm` 进行删除：

```
$ ctr image rm harbor.k8s.local/course/nginx:alpine
harbor.k8s.local/course/nginx:alpine
```

加上 `--sync` 选项可以同步删除镜像和所有相关的资源。


#### 容器操作

容器相关操作可以通过 `ctr container` 获取。

**创建容器**

```
$ ctr container create docker.io/library/nginx:alpine nginx
```

**列出容器**

```
$ ctr container ls
CONTAINER    IMAGE                             RUNTIME
nginx        docker.io/library/nginx:alpine    io.containerd.runc.v2
```

**查看容器详细配置**

类似于 `docker inspect` 功能。

```
$ ctr container info nginx
{
    "ID": "nginx",
    "Labels": {
        "io.containerd.image.config.stop-signal": "SIGQUIT"
    },
    "Image": "docker.io/library/nginx:alpine",
    "Runtime": {
        "Name": "io.containerd.runc.v2",
        "Options": {
            "type_url": "containerd.runc.v1.Options"
        }
    },
    "SnapshotKey": "nginx",
    "Snapshotter": "overlayfs",
    "CreatedAt": "2021-08-12T08:23:13.792871558Z",
    "UpdatedAt": "2021-08-12T08:23:13.792871558Z",
    "Extensions": null,
    "Spec": {
......
```

**删除容器**

```
$ ctr container rm nginx
$ ctr container ls
CONTAINER    IMAGE    RUNTIME
```

除了使用 `rm` 子命令之外也可以使用 `delete` 或者 `del` 删除容器。

#### 任务

上面我们通过 `container create` 命令创建的容器，并没有处于运行状态，只是一个静态的容器。一个 container 对象只是包含了运行一个容器所需的资源及相关配置数据，表示 namespaces、rootfs 和容器的配置都已经初始化成功了，只是用户进程还没有启动。

一个容器真正运行起来是由 Task 任务实现的，Task 可以为容器设置网卡，还可以配置工具来对容器进行监控等。

Task 相关操作可以通过 `ctr task` 获取，如下我们通过 Task 来启动容器：

```
$ ctr task start -d nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
```

启动容器后可以通过 `task ls` 查看正在运行的容器：

```
$ ctr task ls
TASK     PID     STATUS
nginx    3630    RUNNING
```

同样也可以使用 `exec` 命令进入容器进行操作：

```
$ ctr task exec --exec-id 0 -t nginx sh
/ #
```

不过这里需要注意必须要指定 `--exec-id` 参数，这个 id 可以随便写，只要唯一就行。

暂停容器，和 `docker pause` 类似的功能：

```
$ ctr task pause nginx
```

暂停后容器状态变成了 `PAUSED`：


使用 `resume` 命令来恢复容器：

```
$ ctr task resume nginx
$ ctr task ls
TASK     PID     STATUS
nginx    3630    RUNNING
```

不过需要注意 ctr 没有 stop 容器的功能，只能暂停或者杀死容器。杀死容器可以使用 `task kill` 命令:

杀掉容器后可以看到容器的状态变成了 `STOPPED`。同样也可以通过 `task rm` 命令删除 Task：

```
$ ctr task rm nginx
$ ctr task ls
TASK    PID    STATUS
```

除此之外我们还可以获取容器的 cgroup 相关信息，可以使用 `task metrics` 命令用来获取容器的内存、CPU 和 PID 的限额与使用量。

```
$ ctr task metrics nginx
ID       TIMESTAMP
nginx    2021-08-12 08:50:46.952769941 +0000 UTC

METRIC                   VALUE
memory.usage_in_bytes    8855552
memory.limit_in_bytes    9223372036854771712
memory.stat.cache        0
cpuacct.usage            22467106
cpuacct.usage_percpu     [2962708 860891 1163413 1915748 1058868 2888139 6159277 5458062]
pids.current             9
pids.limit               0
```

还可以使用 `task ps` 命令查看容器中所有进程在宿主机中的 PID：

```
$ ctr task ps nginx
PID     INFO
3984    -
4029    -
4030    -
4031    -
4032    -
4033    -
4034    -
4035    -
4036    -

$ ctr task ls
TASK     PID     STATUS
nginx    3984    RUNNING
```

其中第一个 PID `3984` 就是我们容器中的1号进程。

#### 命名空间

另外 Containerd 中也支持命名空间的概念，比如查看命名空间：

```
$ ctr ns ls
NAME    LABELS
default
```

如果不指定，ctr 默认使用的是 `default` 空间。同样也可以使用 `ns create` 命令创建一个命名空间：

```
$ ctr ns create test
$ ctr ns ls
NAME    LABELS
default
test
```

使用 `remove` 或者 `rm` 可以删除 namespace：

```
$ ctr ns rm test
test
$ ctr ns ls
NAME    LABELS
default
```

有了命名空间后就可以在操作资源的时候指定 namespace，比如查看 test 命名空间的镜像，可以在操作命令后面加上 `-n test` 选项：

```
$ ctr -n test image ls
REF TYPE DIGEST SIZE PLATFORMS LABELS
```

我们知道 Docker 其实也是默认调用的 containerd，事实上 Docker 使用的 containerd 下面的命名空间默认是 `moby`，而不是 `default`，所以假如我们有用 docker 启动容器，那么我们也可以通过 `ctr -n moby` 来定位下面的容器：

```
$ ctr -n moby container ls
```

同样 Kubernetes 下使用的 containerd 默认命名空间是 `k8s.io`，所以我们可以使用 `ctr -n k8s.io` 来查看 Kubernetes 下面创建的容器。

### nerdctl

ctr 使用起来体验比较差,因为我们习惯了 Docker 的cli 的语法.

[nerdctl](https://github.com/containerd/nerdctl) 是一个与 docker cli 风格兼容的 containerd 客户端工具，而且直接兼容 docker compose 的语法的

```bash
$ wget https://github.com/containerd/nerdctl/releases/download/v0.18.0/nerdctl-0.18.0-linux-amd64.tar.gz
$ mkdir -p /usr/local/containerd/bin/
$ tar -zxvf nerdctl-0.18.0-linux-amd64.tar.gz
$ mv nerdctl /usr/local/containerd/bin/
$ ln -s /usr/local/containerd/bin/nerdctl /usr/local/bin/nerdctl

# nerdctl 用法和 docker 类似,就不过多叙述
$ nerdctl version
Client:
 Version:       v0.18.0
 Git commit:    77276ff0fffad3f855ab9f2f5a4ad5527ef76485

Server:
 containerd:
  Version:      v1.6.2
  GitCommit:    de8046a5501db9e0e478e1c10cbcfb21af4c6b2d
```

配置命令补全

```bash
$ echo "source <(nerdctl completion bash)" >> ~/.bashrc
$ source ~/.bashrc
```

配置 build 镜像时候使用的依赖.

```bash
# 构建镜像时候遇到如下错误, 原因是 nerdctl build 依赖于 buildkitd 工具
$ nerdctl build -t nginx:nerdctl -f Dockerfile .
FATA[0000] `buildctl` needs to be installed and `buildkitd` needs to be running, see https://github.com/moby/buildkit: exec: "buildctl": executable file not found in $PATH
```

需要我们安装 `buildctl` 并运行 `buildkitd`，这是因为 `nerdctl build` 需要依赖 `buildkit` 工具。

[buildkit](https://github.com/moby/buildkit) 项目也是 Docker 公司开源的一个构建工具包，支持 OCI 标准的镜像构建。它主要包含以下部分:

-   服务端 `buildkitd`：当前支持 runc 和 containerd 作为 worker，默认是 runc，我们这里使用 containerd
-   客户端 `buildctl`：负责解析 Dockerfile，并向服务端 buildkitd 发出构建请求

```bash
$ wget https://github.com/moby/buildkit/releases/download/v0.10.1/buildkit-v0.10.1.linux-amd64.tar.gz

$ tar -zxvf buildkit-v0.9.1.linux-amd64.tar.gz -C /usr/local/containerd/
$ ln -s /usr/local/containerd/bin/buildkitd /usr/local/bin/buildkitd
$ ln -s /usr/local/containerd/bin/buildctl /usr/local/bin/buildctl
```

这里我们使用 Systemd 来管理 buildkitd，创建如下所示的 systemd unit 文件：

```bash
$ cat /etc/systemd/system/buildkit.service
[Unit]
Description=BuildKit
Documentation=https://github.com/moby/buildkit

[Service]
ExecStart=/usr/local/bin/buildkitd --oci-worker=false --containerd-worker=true

[Install]
WantedBy=multi-user.target

$ systemctl daemon-reload
$ systemctl enable buildkit --now
```

> 此时使用 nerdctl 构建镜像即可

### 配置 Kubernetes 使用 Containerd

1. 首先标记需要切换的节点为维护模式，强制驱逐节点上正在运行的 Pod

```bash
$ kubectl cordon kubernetes-node09
$ kubectl drain kubernetes-node09 --ignore-daemonsets
```

2. 停止containerd 和 kubelet

3. 修改 containerd 使用 cgroup 的驱动为 systemd

对于使用 systemd 作为 init system 的 Linux 的发行版，使用 systemd 作为容器的 cgroup driver 可以确保节点在资源紧张的情况更加稳定，所以推荐将 containerd 的 cgroup driver 配置为 systemd。

修改前面生成的配置文件 /etc/containerd/config.toml，在 plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options 配置块下面将 SystemdCgroup 设置为 true

```bash
$ cat /etc/containerd/config.toml | grep  SystemdCgroup
            SystemdCgroup = true
```

4. 修改沙箱镜像地址, 这个就是 pause 容器的地址

```bash
$ cat /etc/containerd/config.toml | grep sandbox_image
    sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6"
```

5. 在 `"io.containerd.grpc.v1.cri".registry` 插件下配置镜像加速地址

```bash
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = ""

  [plugins."io.containerd.grpc.v1.cri".registry.auths]

  [plugins."io.containerd.grpc.v1.cri".registry.configs]

  [plugins."io.containerd.grpc.v1.cri".registry.headers]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://bqr1dr1n.mirror.aliyuncs.com"]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
      endpoint = ["https://registry.aliyuncs.com/k8sxio"]
```

6. 启动 containerd

7. 配置 kubelet 使用 containerd

将默认使用的docker改为containerd [参考 flag设置](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)。我们可以通过运行

```bash
$ cat /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --network-plugin=cni"
```

8. 重新启动 kubelet

9. 恢复节点

```bash
$ kubectl uncordon kubernetes-node09

# 查看 node09 使用的 contianer-runtime
kubectl get nodes -o wide
NAME                  STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kubernetes-master01   Ready    control-plane,master   22d   v1.23.5   32.16.5.21    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-master02   Ready    control-plane,master   22d   v1.23.5   32.16.5.22    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-master03   Ready    control-plane,master   22d   v1.23.5   32.16.5.23    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-node01     Ready    <none>                 22d   v1.23.5   32.16.5.31    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-node02     Ready    <none>                 22d   v1.23.5   32.16.5.32    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-node03     Ready    <none>                 22d   v1.23.5   32.16.5.33    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-node04     Ready    <none>                 22d   v1.23.5   32.16.5.34    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-node05     Ready    <none>                 22d   v1.23.5   32.16.5.35    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-node06     Ready    <none>                 22d   v1.23.5   32.16.5.36    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-node07     Ready    <none>                 22d   v1.23.5   32.16.5.37    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-node08     Ready    <none>                 22d   v1.23.5   32.16.5.38    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   docker://20.10.7
kubernetes-node09     Ready    <none>                 22d   v1.23.5   32.16.5.39    <none>        Ubuntu 20.04.4 LTS   5.4.0-104-generic   containerd://1.6.2
```


