## 背景

之前在了解了容器的 `cgroupfs` 后,又转而使用了 `K8s` 而在 K8s 中 需要配置 Docker 的 `Cgroup Driver = systemd`, 而 Docker 默认的是 `cgroupfs`. 这里很不解为什么要使用 systemd.

为什么 K8s 要依赖于 docker 使用 systemd 的 cgroup, 原因是 K8s 使用的是默认的 Systemd 自带的 cgroup 管理器. 如果 docker 使用 cgroupfs 就会造成冲突。

### cgroup 的默认层级

默认情况下, systemd 会自动创建 slice, scope 和 service 单位的层级,来为cgroup树提供统一结构。使用systemctl指令,您可以通过创建自定义slice进一步修改此结构, systemd也自动为/sys/fs/cgroup/目录中重要的kernel资源管控器挂载层级。

cgroupfs 默认挂载到 `/sys/fs/cgroup/` 下, 除了一些系统支持的 `subsystem` 外, 还有一个目录就是 `systemd`, 而这个目录是由 systemd 自己使用的.

**systemd 的单位类型**

系统中运行的所有进程，都是 systemd init 进程的子进程。在资源管控方面，systemd提供了三种单位类型:

service : 一个或一组进程，由 systemd 依据单位配置文件启动。service 对指定进程进行封装，这样进程可以作为一个整体被启动或终止。service 参照以下方式命名：`name.service`

scope : 一组外部创建的进程。由强制进程通过 fork() 函数启动和终止、之后被 systemd 在运行时注册的进程，scope 会将其封装。例如：用户会话、 容器和虚拟机被认为是 scope。scope 的命名方式如下: `name.scope`

slice： 一组按层级排列的 unit。slice 并不包含进程，但会组建一个层级，并将 scope 和 service 都放置其中。真正的进程包含在 scope 或 service 中。在这一被划分层级的树中，每一个 slice 单位的名字对应通向层级中一个位置的路径。`A.slce` `A-xxx.slce` 是 `A.slce` 的子 slce

> 可以通过 systemd-cgls 命令来查看 systemd 层级

> service、scope 和 slice unit 被直接映射到 cgroup 树中的对象

### 举个栗子

创建一个 docker 限制 200m 内存

```bash
$ docker run -itd --name=test -m 200M  ubuntu

$ docker ps -a |grep test
7325065fce10   ubuntu          "bash"           39 minutes ago   Up 39 minutes   test
```

其实 systemd 管理的资源绑定在了各个 subsystem 中

```
$ find /sys/fs/cgroup/ -name "*7325065fce10*"
./pids/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./perf_event/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./devices/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./freezer/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./hugetlb/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./blkio/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./net_cls,net_prio/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./cpu,cpuacct/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./memory/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./cpuset/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./systemd/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
./unified/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope
```

这里主要查看我们限制的内存资源

```bash
# 可以看到 确实是 200M, 具体换算单位是 1024
$ cat ./memory/system.slice/docker-7325065fce10d1950546261f1ff3e19c1ccd8c5b44d174731a81f74a2cae6f73.scope/memory.limit_in_bytes
209715200
```