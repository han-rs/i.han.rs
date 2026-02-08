+++
title = "了解和使用 Rootless Podman"
date = "2026-02-04"
updated = "2026-02-09"
description = "本文是笔者近日在个人存储服务器通过 rootless Podman 隔离部署应用程序, 尤其是闭源应用程序的过程中总结的经验分享 (踩坑体验), 供读者参考."

[taxonomies]
tags = ["Podman"]

[extra]
toc = true
tldr = """
"""
+++

## 导语

我们需要意识到:

1. 日常使用 root 用户本来就存在极大安全隐患;
1. Docker 的守护程序设计, 导致 Docker 并不能真正实现 rootless, rootless Docker 的配置也是极其麻烦;
1. 在容器中运行的进程与在宿主上直接运行的其他进程没有什么不同. 部分镜像遵循最优实践, 以自定义用户启动程序, 安全性尚可; 但部分镜像直接以容器内的 root 用户启动进程, 如果在使用该镜像时没有指定用户或配置用户映射, 则近似等价于以宿主机的 root 用户启动了程序; 如果还指定了 `--privileged`, 安全性直接归零...

Podman 由于其无守护程序设计, 天然支持 rootless 模式, 逐渐走进了用户的视野.

## 安装 Podman

在 Debian 13 中, 直接通过 apt 安装即可:

```sh
sudo apt update
sudo apt install podman
```

完成安装后, 执行 `podman info` 可以查看 Podman 的详细配置信息.

需要注意, 截止至本文写作时, 虽然 Podman 的最新 stable 版本是 5.7.1, 但 Debian 13 官方软件包提供的 Podman 的版本是 5.4.2, 本文所有例子均基于 5.4.2 测试.

## rootless Podman 基本概念

### 命名空间 (Namespace)

命名空间是 Linux 提供的一种内核级别环境隔离的机制, 容器化技术重度使用了命名空间机制来实现容器的隔离. 关于命名空间的详细介绍此处不再赘述, 读者可以阅读文末的参考文献, 此处仅介绍我们最常打交道的, 也是给普通用户带来最多疑问的用户命名空间和网络命名空间.

#### 用户命名空间

不严谨地说, 用户命名空间提供了一种使容器内用户能够映射到宿主机上的(通常是非特权的)用户的机制, 而容器中的 "root" 仅在容器的用户命名空间内具有 "完全" 的权限, 对容器的用户命名空间外的资源的操作权限则被限制为 "映射" 到的 "普通用户" 的操作权限, 正如官方文档所言: "Rootless Podman is not, and will never be, root".

默认情况下, 通过 rootless Podman 运行容器时, 容器内的 "root" (UID=0, GID=0) 会被映射为运行容器的用户的 UID (GID), 其他 UID (GID) 则映射至宿主机上的高位 UID (GID), 这些高位 UID (GID) 往往并没有对应的宿主机用户(组), 权限则按照 "Others" 处理, 例如, 无法访问权限配置为 `drwxrwx---` (770) 的宿主机文件. 具体的映射规则参见后文. 当然, 也可以选择不映射, 此时容器用户命名空间的 UID 拥有的任何文件对象则将被视为由 "nobody" (65534, `kernel.overflowuid`) 拥有.

可以通过 [`podman-unshare`] 工具创建并进入一新用户命名空间, 以便捷地验证上述映射关系 (值得一提, 在调试 rootless Podman 的权限问题时我们会经常和这个工具打交道). 如:

```shellsession
> id
uid=1000(hantong) gid=1000(hantong) groups=1000(hantong),4(adm),20(dialout),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev)
> podman unshare
> id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
> cat /proc/self/uid_map
         0       1000          1
         1     100000      65536
> cat /proc/self/gid_map
         0       1000          1
         1     100000      65536
> exit
```

可以看到, 容器内的 UID (GID) 0 被映射为宿主机的 UID (GID) 1000; 从 UID (GID) 1 开始的 65536 个 UID (GID) 逐个被映射为宿主机的 UID (GID) 100000、100001, 以此类推.

这里的映射规则是由 `/etc/subuid` 和 `/etc/subgid` 文件定义的, 通常在使用 `adduser` (`addgroup`) 添加用户(组)时会自动设置:

```shellsession
> cat /etc/subuid
hantong:100000:65536
> cat /etc/subgid
hantong:100000:65536
```

前面提到, 默认情况下容器中的 "root" 被映射为运行容器的用户, 对容器外资源的访问权限与当前用户 "基本" 一致. 参考下面的例子, 相信读者就知道 "映射" 是什么含义了:

```shellsession
> ls -ildn /home/hantong
143631 drwx------ 17 1000 1000 4096 Feb  8 21:53 /home/hantong
> podman unshare ls -ildn /home/hantong
143631 drwx------ 17 0 0 4096 Feb  8 21:53 /home/hantong
```

```shellsession
> podman unshare touch /tmp/test
> podman unshare ls -iln /tmp/test
3432 -rw-r--r-- 1 0 0 0 Feb  8 21:54 /tmp/test
> ls -iln /tmp/test
3432 -rw-r--r-- 1 1000 1000 0 Feb  8 21:54 /tmp/test
```

如果不幸地, 你的容器进程通过某些安全漏洞逃逸了容器, 那么它也只能以映射到的高位 UID (GID) 的权限 (或者 "nobody" 的权限) 在宿主机上运行, 这大大降低了容器逃逸后的危害.

#### 网络命名空间

(TBD.)

网络命名空间负责提供容器内网络与宿主机网络乃至其他容器网络的隔离.

Host 网络模式 (`--network=host`) 在 rootless Podman 中也可用, 此时容器完全复用主机网络命名空间, 牺牲一定安全性换取网络性能; 否则, 各个容器有其独立的网络命名空间, 不在同一网络命名空间中的进程无法直接通信.

Docker 和旧版本 Podman 使用 `slirp4netns` 用户态网络驱动来实现 rootless 网络, 性能较差; Podman v5 以后, 默认使用 "性能更优" 的 `pasta` 代替 `slirp4netns`.

相对于 `slirp4netns`, `pasta` 的特点:

1. `pasta` 支持 IPv6;
1. `pasta` 使用主机中的接口名称, 从主机复制 IP 地址并使用主机中的网关地址.

下面是一个例子:

```shellscript
> podman unshare --rootless-netns ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host proto kernel_lo 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65520 qdisc fq state UNKNOWN group default qlen 1000
    link/ether 52:3a:d5:14:43:4c brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.200/24 brd 192.168.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 192.168.1.25/24 metric 100 brd 192.168.1.255 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 [REDACTED]/64 scope global nodad mngtmpaddr noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 [REDACTED]/64 scope global nodad 
       valid_lft forever preferred_lft forever
    inet6 fe80::503a:d5ff:fe14:434c/64 scope link nodad tentative proto kernel_ll 
       valid_lft forever preferred_lft forever
> podman unshare ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq state UP group default qlen 1000
    link/ether bc:24:11:46:30:a6 brd ff:ff:ff:ff:ff:ff
    altname enp6s18
    altname enxbc24114630a6
    inet 192.168.1.200/24 brd 192.168.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 192.168.1.25/24 metric 100 brd 192.168.1.255 scope global secondary dynamic eth0
       valid_lft 512686sec preferred_lft 512686sec
    inet6 [REDACTED]/64 scope global temporary dynamic 
       valid_lft 534478sec preferred_lft 3195sec
    inet6 [REDACTED]/64 scope global temporary deprecated dynamic 
       valid_lft 448111sec preferred_lft 0sec
  # ...
       valid_lft 1209195sec preferred_lft 3195sec
    inet6 fe80::be24:11ff:fe46:30a6/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
3: lan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq state UP group default qlen 1000
    link/ether bc:24:11:40:61:e6 brd ff:ff:ff:ff:ff:ff
    altname enp6s19
    altname enxbc24114061e6
    inet 172.16.0.2/24 brd 172.16.0.255 scope global lan0
       valid_lft forever preferred_lft forever
    inet6 fe80::be24:11ff:fe40:61e6/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
```

### 存储驱动

(TBD.)

1. OverlayFS 和 VFS 的权限表现似乎有差异.

### 网络

(TBD.)

Podman 提供多种网络模式, 参考文档 <https://docs.podman.io/en/latest/markdown/podman-run.1.html#network-mode-net>:

1. `bridge` (桥接).

   rootful Podman 默认, 直接桥接到主机网络接口.

1. `none`

   创建网络命名空间, 但是不配置网络, 容器内没有任何网络接口.

1. `container:id`

   加入另一个容器的网络命名空间.

   (注: 笔者更推荐使用 Pod 管理需要通过网络相互访问的容器.)

1. `host`

   复用主机的网络命名空间. 虽然性能很好, 但这也使容器能够完全访问宿主机的 abstract unix socket 以及绑定到 localhost 的 TCP / UDP socket, 存在安全隐患.

1. `pasta[:OPTIONS,…]`

   rootless Podman 默认, 创建一个新的网络命名空间, 使用 `pasta` 创建用户态网络堆栈.

其他模式此处暂不展开, 如有需要参考官方文档.

## 配置 Podman

1. 配置镜像仓库

   Podman 默认设置下无法像 Docker 那样不提供镜像源时默认从 `docker.io` 拉取镜像.

   建议配置 `registries.conf` 文件:

   ```shellsession
   > sudo nano /etc/containers/registries.conf 
   ```

   修改 `unqualified-search-registries` 字段, 添加 `docker.io`:

   ```conf
   unqualified-search-registries = ["docker.io"]
   ```

## 运行 rootless Podman 容器

完成上述配置后, 即可以非 root 用户运行 Podman 容器. 测试如下:

```shellsession
> podman run hello
!... Hello Podman World ...!

         .--"--.
       / -     - \
      / (O)   (O) \
   ~~~| -=(,Y,)=- |
    .---. /`  \   |~~
 ~/  o  o \~~~~.----. ~~
  | =(X)= |~  / (O (O) \
   ~~~~~~~  ~| =(Y_)=-  |
  ~~~~    ~~~|   U      |~~

Project:   https://github.com/containers/podman
Website:   https://podman.io
Desktop:   https://podman-desktop.io
Documents: https://docs.podman.io
YouTube:   https://youtube.com/@Podman
X/Twitter: @Podman_io
Mastodon:  @Podman_io@fosstodon.org
```

当然, rootless Podman 的配置和使用有许多和 rootful Docker 不同的地方, 下面不完整地列举一些.

1. 需要配置 `/etc/subuid` 和 `/etc/subgid` 文件.

   一般来说, 通过 `useradd` 命令创建用户时, 系统会自动为新用户在 `/etc/subuid` 和 `/etc/subgid` 中分配 UID 和 GID 范围; 如果是手动创建用户, 则需要手动添加相应条目. 例如, 为用户 `hantong` 分配 UID 和 GID 范围:

   ```sh
   sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 hantong
   ```

   请注意, 每个用户的子 UID (GID) 范围不应重叠, 否则可能导致权限冲突和安全问题.

1. 需要解除 Linux 对非 root 用户绑定 1024 以下端口的限制.

   可以通过以下命令解除该限制:

   ```sh
   sudo sysctl -w net.ipv4.ip_unprivileged_port_start=0
   ```

1. 用户配置文件位置和生效顺序问题.

   以下三个目录保存了 Podman 的配置文件:

   1. `/usr/share/containers`
   1. `/etc/containers`
   1. `${XDG_CONFIG_HOME}/containers` (默认 `~/.config/containers`)

   包含以下配置文件:

   1. `containers.conf`

      按前述顺序, 后者配置项覆盖前者.

   1. `storage.conf`

      在 rootless Podman 中, `/etc/containers/storage.conf` 中的某些字段将被忽略:

      ```ini
      graphroot=""
       container storage graph dir (default: "/var/lib/containers/storage")
       Default directory to store all writable content created by container storage programs.

      runroot=""
       container storage run dir (default: "/run/containers/storage")
       Default directory to store all temporary writable content created by container storage programs.
      ```

      在 rootless Podman 中, 这些字段默认为:

      ```ini
      graphroot="${XDG_DATA_HOME}/containers/storage"
      runroot="${XDG_RUNTIME_DIR}/containers"
      ```

      `XDG_DATA_HOME` 默认为 `~/.local/share`, `XDG_RUNTIME_DIR` 默认为 `/run/user/<uid>` (systemd).

   1. `registries.conf`

      按前述顺序读取.

   1. 授权文件

      `podman login` 和 `podman logout` 命令使用的默认授权文件是 `${XDG_RUNTIME_DIR}/containers/auth.json`.

1. 用户命名空间相关的大量问题.

   这个应该是 rootless Podman 最让人头疼的地方了.

   Podman 提供了以下用户命名空间模式, 需要我们了解:

   1. `host` (默认): 创建用户命名空间, 容器中的根用户(组)映射为运行容器时的用户(组), 其余 UID (GID) 按照前述映射规则依次映射为宿主机高位 UID (GID).

      ```shellsession
      > podman run -it --rm --userns=host --name="test" --replace debian:trixie-slim bash
      root@0df58777f431:/# id
      uid=0(root) gid=0(root) groups=0(root)
      root@0df58777f431:/# sleep 3000
      ^C
      root@0df58777f431:/# useradd test
      root@0df58777f431:/# su test
      $ id
      uid=1000(test) gid=1000(test) groups=1000(test)
      $ sleep 3000
      ^C
      $ exit
      root@0df58777f431:/# exit
      exit
      ```

      ```shellsession
      # 以容器内 root 用户 (UID=0) 运行 sleep 时
      > ps -ax -o pid,user,group,uid,gid,args | grep "sleep 3000"
      20953 hantong  hantong   1000  1000 sleep 3000
      # 以容器内 test 用户 (UID=1000) 运行 sleep 时
      > ps -ax -o pid,user,group,uid,gid,args | grep "sleep 3000"
      21384 100999   100999   100999 100999 sleep 3000
      ```

   1. `keep-id`: 类似 `host`, 但是映射当前用户的 UID (GID) 到容器内的相同 UID (GID). 同时, 忽视镜像的 `USER` 配置, 在当前用户的 UID 下运行 init 进程, 除非手动指定用户.

      ```shellsession
      > podman run -it --rm --userns=keep-id --name="test" --replace debian:trixie-slim bash
      hantong@cea54e0fd3fb:/$ id
      uid=1000(hantong) gid=1000(hantong) groups=1000(hantong)
      hantong@cea54e0fd3fb:/$ exit
      exit
      > id
      uid=1000(hantong) gid=1000(hantong) groups=1000(hantong),27(sudo)
      ```

   1. "nomap": 创建用户命名空间, 但不映射当前用户, 容器内的所有 UID (GID) 按照前述映射规则依次映射为宿主机高位 UID (GID).

      ```shellsession
      > podman run -it --rm --userns=nomap --name="test" --replace debian:trixie-slim bash
      root@f3794e591a2c:/# id
      uid=0(root) gid=0(root) groups=0(root)
      root@f3794e591a2c:/# sleep 3000
      ^C
      root@f3794e591a2c:/# useradd test
      root@f3794e591a2c:/# su test
      $ sleep 3000
      ^C
      $ exit
      root@f3794e591a2c:/# exit
      exit
      ```

      ```shellsession
      # 以容器内 root 用户 (UID=0) 运行 sleep 时
      > ps -ax -o pid,user,group,uid,gid,args | grep "sleep 3000"
      28326 100000   100000   100000 100000 sleep 3000
      # 以容器内 test 用户 (UID=1000) 运行 sleep 时
      > ps -ax -o pid,user,group,uid,gid,args | grep "sleep 3000"
      28563 101000   101000   101000 101000 sleep 3000
      ```

   很多时候, 我们需要将宿主机的某个目录挂载到容器内, 使用 rootful Docker 外加关掉 (或者说根本没开启) SELinux 自然可以忽视 (~~暴力解决~~) 权限问题, 但 rootless Podman 就不行了.

   1. 当镜像没有配置 `USER`:

      此时, 默认以容器内的 "root" (UID=0, GID=0) 运行. 容器内的 "root" 在 rootless Podman 默认的用户命名空间模式 ("--userns=host") 下如前所述被映射到宿主机下运行容器的当前用户, 文件系统权限表现和当前用户**基本**一致.

   1. 当镜像遵循最佳实践配置了 (rootless) `USER=UID[:GID]`:

      此时, 容器内进程以容器内的该 UID (GID) 运行, 而容器内 UID (GID) 在 rootless Podman 默认的用户命名空间模式 ("--userns=host") 下如前所述被映射到宿主机下的高位 UID (GID), 此时文件系统权限表现和宿主机上的 "其他用户" 一样了.

   这里有一个实例:

   ```shellsession
   > podman run -d --restart=unless-stopped -v /home/hantong/.config/openlist:/opt/openlist/data -p 5244:5244 --name="openlist" --replace openlistteam/openlist:latest
   > podman logs openlist
   Error: Current user does not have write and/or execute permissions for the ./data directory: /opt/openlist/data
   # ...
   ```

   ```dockerfile
   # ...
   ARG USER=openlist
   ARG UID=1001
   ARG GID=1001
   # ...
   RUN addgroup -g ${GID} ${USER} && \
       adduser -D -u ${UID} -G ${USER} ${USER} && \
       mkdir -p /opt/openlist/data
   # ...
   USER ${USER}
   # ...
   CMD [ "/entrypoint.sh" ]
   ```

   在这个案例中, "/entrypoint.sh" 是以容器内的 UID=1001 GID=1001 运行的, 被映射到宿主机下的 UID=101000 GID=101000. 显然其无法访问权限为 600, 归属 UID=1000 GID=1000 的的宿主机目录 `/home/hantong`.

   此时, 可以配置 Podman 的用户命名空间模式为 "keep-id", 以确保 init 进程 "实质" 以当前用户的 UID (GID) 运行, 从而对 `/home/hantong` 目录具有访问权限, 如:

   ```shellsession
   > podman run -d --userns=keep-id --restart=unless-stopped -v /home/hantong/.config/openlist:/opt/openlist/data -p 5244:5244 --name="openlist" --replace openlistteam/openlist:latest
   > podman logs openlist
   INFO[2026-02-08 07:14:43] reading config file: /opt/openlist/data/config.json 
   INFO[2026-02-08 07:14:43] load config from env with prefix:            
   INFO[2026-02-08 07:14:43] max buffer limit: 1502MB                     
   INFO[2026-02-08 07:14:43] mmap threshold: 4MB                          
   INFO[2026-02-08 07:14:43] init logrus...                               
   start HTTP server @ 0.0.0.0:5244
   > ps -ax -o pid,user,group,uid,gid,comm | grep openlist
   31334 101000   101000   101000 101000 openlist
   > podman exec -it openlist bash
   695025dade25:/opt/openlist$ id
   uid=1001(openlist) gid=1001(openlist) groups=1001(openlist)
   695025dade25:/opt/openlist$ ps -o pid,user,group,comm | grep openlist
      1 openlist openlist openlist
     26 openlist openlist bash
     28 openlist openlist ps
     29 openlist openlist grep
   695025dade25:/opt/openlist$ exit
   exit
   ```

   当然, 此时非 init 进程还是以 USER 配置的 UID (GID) 运行的:

   ```shellsession
   > ls -aln /home/hantong/.config/openlist
   total 276
   drwxrwxrwx  4   1000   1000   4096 Jan 31 16:36 .
   drwxrwxr-x 14   1000   1000   4096 Jan 31 16:23 ..
   -rw-r--r--  1 101000 101000   2899 Feb  8 15:14 config.json
   -rw-r--r--  1 101000 101000   4096 Jan 31 16:28 data.db
   -rw-r--r--  1 101000 101000  32768 Feb  8 15:14 data.db-shm
   -rw-r--r--  1 101000 101000 222512 Feb  8 15:14 data.db-wal
   drwxr--r--  2 101000 101000   4096 Jan 31 16:28 log
   drwxr-xr-x  2 101000 101000   4096 Jan 31 16:28 temp
   ```

   为了避免编辑配置文件还得 `sudo` 的麻烦, 建议配置为 `--userns=keep-id:uid=${UID},gid=${GID}`, 其中 UID, GID 为构建镜像时所配置的用户的数字 ID, 或者直接强制指定用户, 如:

   ```shellsession
   > podman run -d --userns=keep-id:uid=1001,gid=1001 --restart=unless-stopped -v /home/hantong/.config/openlist:/opt/openlist/data -p 5244:5244 --name="openlist" --replace openlistteam/openlist:latest
   4fd47db9916432a8d4631b99c6a6fedd9a9c94c8ca3580e40f0f6bde41b83102
   > ls -aln /home/hantong/.config/openlist
   total 260
   drwxrwxr-x  4 1000 1000   4096 Feb  8 16:13 .
   drwxrwxr-x 14 1000 1000   4096 Feb  8 16:13 ..
   -rw-r--r--  1 1000 1000   2842 Feb  8 16:13 config.json
   -rw-r--r--  1 1000 1000   4096 Feb  8 16:13 data.db
   -rw-r--r--  1 1000 1000  32768 Feb  8 16:13 data.db-shm
   -rw-r--r--  1 1000 1000 206032 Feb  8 16:13 data.db-wal
   drwxr--r--  2 1000 1000   4096 Feb  8 16:13 log
   drwxr-xr-x  2 1000 1000   4096 Feb  8 16:13 temp
   ```

   这个时候, 容器内的 init 进程和非 init 进程均 "本质" 以当前用户的 UID (GID) "降权" 运行, 文件系统权限表现和当前用户基本一致.

1. Capabilities

   关于 Capabilities, 参考 <https://man7.org/linux/man-pages/man7/capabilities.7.html>, 暂且理解为对权限控制的细化.

   [Podman 的默认 Capabilities](https://github.com/rhatdan/common/blob/ed2840c80e47bdadaff2c7d827111d401dff282d/pkg/config/containers.conf#L48-L60) 和 [Docker 的默认 Capabilities](https://github.com/moby/moby/blob/docker-29.x/daemon/pkg/oci/caps/defaults.go) 是不同的:

   | Capability | Podman | Docker |
   | :---: | :---: | :---: |
   | **CHOWN** | ✔️ | ✔️ |
   | **DAC_OVERRIDE** | ✔️ | ✔️ |
   | **FSETID** | ✔️ | ✔️ |
   | **FOWNER** | ✔️ | ✔️ |
   | **MKNOD** | ❌ | ✔️ |
   | **NET_RAW** | ❌ | ✔️ |
   | **SETGID** | ✔️ | ✔️ |
   | **SETUID** | ✔️ | ✔️ |
   | **SETFCAP** | ✔️ | ✔️ |
   | **SETPCAP** | ✔️ | ✔️ |
   | **NET_BIND_SERVICE** | ✔️ | ✔️ |
   | **SYS_CHROOT** | ✔️ | ✔️ |
   | **KILL** | ✔️ | ✔️ |
   | **AUDIT_WRITE** | ❌ | ✔️ |

   在容器化现有应用程序时时, 需要注意两者的差异; 对终端用户来说没有什么直接影响, 但建议启动容器时配置 `--security-opt=no-new-privileges` 以禁止容器进程获得额外权限, 或者直接禁用所有 Capabilities (`--cap-drop=all`) 再按需添加 (`--cap-add=CAP_NAME`), 防止类似 `CAP_BPF` 这些后面才引入的高危 Capabilities 被意外添加.

## 编写 rootless 友好的 Dockerfile

总体和编写普通 Dockerfile 没什么区别, 但需要注意以下几点:

1. 指定 rootless `USER`, 需要是数字 ID.

   推荐: `USER=65532:65532`.

1. 尽量使用 Distroless 镜像作为基础镜像.

   1. 如果应用程序是完全静态编译的, 直接使用 `scratch` (适合现代应用程序, 没有奇奇怪怪的依赖) 或 `gcr.io/distroless/static` (适合传统应用程序, 仍然依赖系统 CA 之类的奇怪玩意).

      不同语言编写的项目的静态编译方法参考:

      1. 对于 C / C++, 具体项目具体分析.
      1. 对于 Go, 可以配置 `CGO_ENABLED=0`, 如果项目以及项目的依赖没有依赖 CGO 的话.
      1. 对于 Rust, 可以配置 target `x86_64-unknown-linux-musl` (amd64) 或 `aarch64-unknown-linux-musl` (arm64) 目标平台进行静态编译.

         ```shellsession
         > rustup target add x86_64-unknown-linux-musl
         # ...
         > cd /tmp
         > cargo new test-static-build
             Creating binary (application) `test-static-build` package
         note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
         > cd test-static-build
         > cargo build --release --target x86_64-unknown-linux-musl
           Compiling test-static-build v0.1.0 (/tmp/test-static-build)
             Finished `release` profile [optimized] target(s) in 1.22s
         > ldd /data/Compile/cargo-target-dir/x86_64-unknown-linux-musl/release/test-static-build
                 statically linked
         ```

         一般来说, 能编译为 `x86_64-unknown-linux-gnu` 就能编译为 `x86_64-unknown-linux-musl`, 除非依赖到某些奇奇怪怪的 C 库.

       需要注意的是, 如果应用程序依赖于 glibc, 则无法直接使用 `scratch` 镜像, 因为 `scratch` 是一个完全空白的镜像, 不包含任何库文件. 此时, 可以使用 `gcr.io/distroless/base` 镜像, 它包含了 glibc 和一些基本的系统工具, 适合运行依赖 glibc 的应用程序.

      1. ...

   1. 如果应用程序依赖于 glibc, 可以使用 `gcr.io/distroless/base-nossl`; 如果应用程序还依赖于 OpenSSL, 可以使用 `gcr.io/distroless/base`.

   一些问题:

   1. 尝试运行容器时, 报告 "error while loading shared libraries: XXX: cannot open shared object file: No such file or directory" 错误.

      此即缺失依赖库问题.

      这种情况下, 使用 ldd 查看依赖库, 再直接参考相关库的安装方法从完整镜像复制文件即可, 如[笔者在打包 Resilio Sync 时的做法](https://github.com/han-rs/container-ci-resilio-sync/blob/main/Dockerfile):

      ```dockerfile
      # ...
      ARG DEBIAN_BASE_VERSION=trixie-slim
      ARG DEBIAN_BASE_HASH=f6e2cfac5cf956ea044b4bd75e6397b4372ad88fe00908045e9a0d21712ae3ba
      # ...
      FROM --platform=linux/${TARGETARCH} debian:${DEBIAN_BASE_VERSION}@sha256:${DEBIAN_BASE_HASH} AS downloader
      # ...
      RUN set -e && \
      # ...
      && \
      ARCH=$(dpkg --print-architecture) \
      && \
      case "${ARCH}" in \
      amd64) LIBARCH="x86_64-linux-gnu" ;; \
      arm64) LIBARCH="aarch64-linux-gnu" ;; \
      *) echo "Unsupported architecture: ${ARCH}" && exit 1 ;; \
      esac \
      && \
      cp /lib/${LIBARCH}/libcrypt.so.1 /tmp/resilio-sync/lib/libcrypt.so.1
      # ...
      FROM scratch
      # ...
      ENV LD_LIBRARY_PATH=/opt/resilio-sync/lib
      # ...
      ```

      (笔者评: Resilio Sync 怎么还在用 OpenSSL 1.1 啊, 硬要用就不能静态编译进程序里面么... C / C++ 的依赖灾难在这就体现了...)

   1. OpenSSL CA 问题.

      但凡最终应用程序报告 SSL 相关错误, 大抵是这里所述的问题.

      这种情况下, 可以在构建阶段安装 ca-certificates 包, 将 CA 文件复制到最终镜像中, 并正确配置环境变量, 如[笔者在打包 qBittorrent 时的做法](https://github.com/han-rs/container-ci-qbittorrent/blob/main/Dockerfile):

      ```dockerfile
      # ...
      ARG ALPINE_BASE_VERSION=3.23.3
      ARG ALPINE_BASE_HASH=25109184c71bdad752c8312a8623239686a9a2071e8825f20acb8f2198c3f659
      # ...
      FROM alpine:${ALPINE_BASE_VERSION}@sha256:${ALPINE_BASE_HASH} AS downloader
      # ...
      RUN set -e && \
          apk -U upgrade && apk add --no-cache \
          ca-certificates=20251003-r0 \
      # ...
      FROM scratch
      # ...
      COPY --from=downloader /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
      # ...
      ENV SSL_CERT_DIR=/etc/ssl/certs \
          SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
      # ...
      ```

      (笔者评: 还是喜欢 Rustls + WebPKI, OpenSSL 这种岁月痕迹过于重的库, 唉...)

## 使用 systemd 管理 rootless Podman 容器

Podman 不像 Docker 那样有守护程序, 需要以其他方式管理容器的自动启动等. Podman 提供了 [Quadlet], 以通过 systemd 来管理 Podman 容器.

下面是一个示例 Quadlet 文件:

```systemd
[Unit]
Description=FreeNGINX Distroless Container Service
RequiresMountsFor=%t/containers

[Container]
# * Specify the name of the container.
ContainerName=freenginx
# * Specify the container image to use.
Image=ghcr.io/han-rs/container-ci-freenginx:latest
# * Automatically pull the latest image.
AutoUpdate=registry
# * Run the container in user namespace mode with the current user's ID.
# *
# * Don't modify this field unless you know what you're doing.
UserNS=keep-id:uid=65532,gid=65532
# * Use the host's network stack for better performance.
Network=host
# * Publish ports from the container to the host.
# *
# * If Network=host is used, these options will be ignored.
# * The format is [host_ip:]host_port:container_port[/protocol].
# * Please set these according to what you configured.
# PublishPort=80:80
# PublishPort=443:443
# PublishPort=443:443/udp
# * Persists configuration files.
Volume=%h/.local/share/freenginx/conf:/opt/freenginx/conf
# * Persists logs.
Volume=%h/.local/share/freenginx/logs:/opt/freenginx/logs
# * Persists SSL stuffs.
Volume=%h/.local/share/freenginx/ssl:/opt/freenginx/ssl
# * Example of mounting a named volume.
# Volume=cloudflared.volume:/opt/cloudflared/shared-volume
# * Example of mounting a host directory.
# Volume=/host/dir:/container/dir

[Service]
# * Always restart the container if it stops.
Restart=always
# * Give the container more time to start.
TimeoutStartSec=900
# * Generate configuration extracting real IP from HTTP headers added by Cloudflare.
#
# * If you gate your servers behind Cloudflare, uncomment the following line.
# ExecStartPre=%h/.local/share/freenginx/scripts/cloudflare-real-ip-helper.sh
# * For Podman v5.4 or lower, we have to manually specify reload command.
ExecReload=/usr/bin/podman exec freenginx /opt/freenginx/sbin/nginx -s reload

[Install]
# * Enable the service to start at boot.
WantedBy=default.target
```

基本上, Podman Quadlet systemd 文件可以由 `podman run` 命令逐参数转换而来, 官方文档也给出了[对照表](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html#container-units-container), 编写并不困难.

上面的例子基本上相当于:

```sh
podman run -d --restart=unless-stopped \
    --restart=always \
    --userns=keep-id:uid=65532,gid=65532 \
    --network=host \
    -v ~/.local/share/freenginx/conf:/opt/freenginx/conf \
    -v ~/.local/share/freenginx/logs:/opt/freenginx/logs \
    -v ~/.local/share/freenginx/ssl:/opt/freenginx/ssl \
    --name="freenginx" \
    --replace \
    ghcr.io/han-rs/container-ci-freenginx:latest
```

配置完成后, 可以通过 `systemctl --user ${CMD} ${CONTAINER_NAME}.service` 来管理该容器, 如:

```shellsession
> systemctl --user status freenginx.service
● freenginx.service - FreeNGINX Distroless Container Service
     Loaded: loaded (/home/hantong/.config/containers/systemd/freenginx.container; generated)
     Active: active (running) since Sat 2026-02-07 19:29:12 CST; 21h ago
 Invocation: eda008ef44dc4e79b198a0c69a7ca32f
    Process: 25667 ExecStartPre=/home/hantong/.local/share/freenginx/scripts/cloudflare-real-ip-helper.sh (code=exited, status=0/SUCCESS)
   Main PID: 25696 (conmon)
      Tasks: 134 (limit: 4915)
     Memory: 467M (peak: 3.6G)
        CPU: 16min 28.154s
# ...
```

值得指出, 依托于 systemd, 像什么 `ExecStartPre` 之类的钩子也可以使用了, 不再依赖容器本身去实现这些功能, 这允许容器打包者大幅度简化镜像:

```shellsession
> podman images
REPOSITORY                                TAG         IMAGE ID      CREATED       SIZE
ghcr.io/han-rs/container-ci-freenginx     latest      f583dc3c0362  28 hours ago  3.95 MB
ghcr.io/han-rs/container-ci-resilio-sync  latest      af6cae8d5f91  47 hours ago  38.3 MB
ghcr.io/han-rs/container-ci-qbittorrent   latest      2bf49e5c41e5  2 days ago    45.5 MB
ghcr.io/han-rs/container-ci-cloudflared   latest      a4bbf0388c6b  2 days ago    28 MB
# ...
```

当然, 还有些注意事项和疑难杂症:

1. 自动启动

   1. 需要执行 `sudo loginctl enable-linger $USER`, 使得 rootless 用户的 systemd 服务能够在用户未登录时也能自动启动并驻留后台.
   1. 不需要执行 `systemctl --user enable ***`, Podman 创建的服务被 systemd 视为 "瞬态" (transient) 的.
   1. 需要在 Quadlet 配置文件 `[Install]` 项添加 `WantedBy=default.target` 使容器自动启动. (**存疑**: 是 `default.target` 还是 `multi-user.target`, 还是两者均使用? 笔者个人部署时仅使用前者.)
1. 笔者本人在部署过程中遇到 `network-online.target` 始终没有 active (`systemctl is-active network-online.target` 提示 inactive) 的问题, 导致:

   1. 容器自动启动会延滞于服务器启动完成很久之后;
   1. 启动或重启服务等待很久 (默认 90s 超时), 但成功;
   1. ...

   根本原因暂未清楚, 或许是 `netplan` + `systemd-networkd` 的问题?

需要注意的是, 从 Podman v5 开始, `podman generate systemd` 已经被弃用, 不再推荐使用.

## 结语

总的来说, 如果不想折腾, 直接使用 rootful Docker 确实方便, root 解决一切权限问题, 但面对[飞牛变肥牛](https://linux.do/t/topic/1548846)之类的教训, 还是建议使用 rootless Podman, 也就繁琐一点, 问题并不大.

本文是在笔者个人存储服务器通过 rootless Podman 隔离部署应用程序, 尤其是闭源应用程序的过程中总结的经验, 相关成果也开源在以下仓库中, 欢迎参考、批评指正:

1. [ghcr.io/han-rs/container-ci-freenginx](https://github.com/han-rs/container-ci-freenginx);
1. [ghcr.io/han-rs/container-ci-qbittorrent](https://github.com/han-rs/container-ci-qbittorrent);
1. [ghcr.io/han-rs/container-ci-cloudflared](https://github.com/han-rs/container-ci-cloudflared);
1. [ghcr.io/han-rs/container-ci-resilio-sync](https://github.com/han-rs/container-ci-resilio-sync).

## 参考文献

1. 官方文档, 包括但不限于:

   1. 官方 rootless Podman 指引: <https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md>
   1. 官方 rootless Podman 常见问题总结: <https://github.com/containers/podman/blob/main/rootless.md>
   1. 用户命名空间模式: <https://docs.podman.io/en/latest/markdown/podman-run.1.html#userns-mode>
1. [How does rootless Podman work?](https://opensource.com/article/19/2/how-does-rootless-podman-work)
1. [探索 Linux Namespace: Docker 隔离的神奇背后](https://www.lixueduan.com/posts/docker/05-namespace/)

[`podman-unshare`]: https://docs.podman.io/en/latest/markdown/podman-unshare.1.html
[Quadlet]: https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html
