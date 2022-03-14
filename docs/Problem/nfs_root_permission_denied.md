## 背景

在一个通过 `nfs` 挂载的目录中,突然发现 `root` 用户对其没有权限, 但是某普通用户却有操作权限。

权限指的是操作权限, 如: mkdir chown chmod .....

文件目录大概如下结构

```bash
——/nasapp  (mount -t nfs ...)
｜----testA  root:root (0777)
｜----testB  weblogic:root (0777)
｜----testC  weblogic:root (0777)
```

### 排查

已下步骤均使用 `root` 用户排查

1. 首先查看 mount 权限文件,发现 rw 为正常权限
2. 首先查看问题目录顶层目录权限为 777 root:root, 权限位没有异常
3. 查看特殊权限是否配置使用 `lsattr` 查看,得到的结果是文件系统不支持.
4. 在 testA 中新建读写文件均不可,但是在切换 weblogic 用户后发现没有问题.
5. 在顶层目录下新建 testd 发现一切正常, 但是这时候发现 root 创建的目录,文件夹权限却是 `nobody:nobody`
6. 由于不能正确继承权限,继而查看 nfs 服务端的问题,发现共享配置为 `all_squash`

最终通过把 `all_squash` 修改为 `no_root_squash` 解决

### 何原因导致

这里就要明确下, all_squash 和 no_root_squash 的区别了.

1. no_root_squash: 加上这个选项后，root用户就会对共享的目录拥有至高的权限控制，就像是对本机的目录操作一样。不安全，不建议使用；
2. root_squash: 和上面的选项对应，root用户对共享目录的权限不高，只有普通用户的权限，即限制了root；

因为有些目录是其他用户创建的,所以当root用户再去做操作的时候,出现了权限问题。