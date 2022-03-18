## namespace 作用

Linux 的命名空间机制提供了一种资源隔离的解决方案,PID, IPC, Network 等系统级的全局资源不再是全局性的,而是属于特定的 Namespace, Namespace 是对全局系统资源的一种封装隔离，使得处于不同 namespace 的进程拥有独立的全局系统资源，改变一个 namespace 中的系统资源只会影响当前 namespace 里的进程，对其他 namespace 中的进程没有影响。

1. Mount: 隔离文件系统挂载点
2. UTS: 隔离主机名和域名信息
3. IPC: 隔离进程间通信
4. PID: 隔离进程的ID
5. Network: 隔离网络资源
6. User: 隔离用户和用户组的ID

### 通过进程来查看命名空间

进程的 namespace 在 `/proc/[pid]/ns/` 目录下。

查看某 2 个进程的 ns, 以下可见这俩个进程处于同一个命名空间中,说明这俩个进程共享同一个 Namespace。

```bash
$ ls -l /proc/1/ns/
total 0
lrwxrwxrwx 1 root root 0 Mar 17 14:46 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Mar 17 14:46 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Mar 17 14:46 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Mar 17 14:46 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Mar 17 05:58 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Mar 17 14:46 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Mar 17 14:46 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Mar 17 14:46 uts -> 'uts:[4026531838]'

$ ls -l /proc/2/ns/
total 0
lrwxrwxrwx 1 root root 0 Mar 17 14:49 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Mar 17 14:49 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Mar 17 14:49 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Mar 17 14:49 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Mar 17 05:58 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Mar 17 14:49 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Mar 17 14:49 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Mar 17 14:49 uts -> 'uts:[4026531838]'
```

### linux 内核通过 clone() 这个系统调用,用来创建一个独立 Namespace 的进程。

通过 `unshare` 命令使用与父进程不同的命名空间来运行进程。

通过 `man clone` 中可以看到有一个例子,是创建一个进程运行在独立的 UTS 命名空间中.

```bash
$ ./a.out test
clone() returned 1407
uts.nodename in child:  test
uts.nodename in parent: u-dev01
```

### 参考

[为什么构建容器需要 Namespace](http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E7%94%B1%E6%B5%85%E5%85%A5%E6%B7%B1%E5%90%83%E9%80%8F%20Docker-%E5%AE%8C/09%20%20%E8%B5%84%E6%BA%90%E9%9A%94%E7%A6%BB%EF%BC%9A%E4%B8%BA%E4%BB%80%E4%B9%88%E6%9E%84%E5%BB%BA%E5%AE%B9%E5%99%A8%E9%9C%80%E8%A6%81%20Namespace%20%EF%BC%9F.md)

