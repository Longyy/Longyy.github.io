# 理解Docker中镜像与容器的存储引擎


Docker提供pull/run命令来拉取镜像和运行容器，可你知道镜像被拉取下来后是如何存放的吗，一个镜像就是一个文件？如果你知道镜像的存储格式或者对RootFS有所了解，那么你显然会说：No. 下面我们来一步一步看看究竟是怎么一回事。

演示环境：ubuntu 16.04 / docker 20.10.0

## 一、获取docker存储关键信息

执行 

```shell
$ docker info
...
Storage Driver: overlay2
  Backing Filesystem: extfs
Docker Root Dir: /var/lib/docker
...
```

在输出信息中找到**Storage Driver**、**Docker Root Dir**、**Backing Filesystem**三个配置项，合起来的意思是说，docker的存储路径是`/var/lib/docker`，该路径下的文件格式为`extfs`，使用的存储引擎是`overlay2`.

不同操作系统中Docker存储路径有所不同：

- Ubuntu: `/var/lib/docker/`
- Fedora: `/var/lib/docker/`
- Debian: `/var/lib/docker/`
- Windows: `C:\ProgramData\DockerDesktop`
- MacOS: `~/Library/Containers/com.docker.docker/Data/vms/0/`

Docker Engine - Community (社区版)， 在Linux的不同发行版上推荐的存储引擎如下表：

| **Linux distribution**              | **Recommended storage drivers**                              | **Alternative drivers**                   |
| ----------------------------------- | ------------------------------------------------------------ | ----------------------------------------- |
| Docker Engine - Community on Ubuntu | `overlay2` or `aufs` (for Ubuntu 14.04 running on kernel 3.13) | `overlay`¹, `devicemapper`², `zfs`, `vfs` |
| Docker Engine - Community on Debian | `overlay2` (Debian Stretch), `aufs` or `devicemapper` (older versions) | overlay`¹, `vfs                           |
| Docker Engine - Community on CentOS | overlay2                                                     | overlay`¹, `devicemapper`², `zfs`, `vfs   |
| Docker Engine - Community on Fedora | overlay2                                                     | overlay`¹, `devicemapper`², `zfs`, `vfs   |

¹) The `overlay` 引擎已被废弃，推荐使用 `overlay2`.

²) The `devicemapper` 引擎已被废弃，推荐使用 `overlay2`.

OverlayFS是一种堆叠文件系统（联合文件系统），它依赖并建立在其它的文件系统之上（例如ext4fs和xfs等等），并不直接参与磁盘空间结构的划分，仅仅将原来底层文件系统中不同的目录进行“合并”，然后向用户呈现。因此对于用户来说，它所见到的overlay文件系统根目录下的内容就来自挂载时所指定的不同目录的“合集”。

Docker为OverlayFS提供了两个存储引擎，原始的Overlay和更稳定的Overlay2.

当前安装Docker时，**`overlay2`已经是默认的存储引擎了**, 在此之前`aufs`是默认引擎，[点此获取关于`aufs`的更多信息](https://docs.docker.com/storage/storagedriver/aufs-driver/)

Backing Filesystem与存储引擎的对应关系：

| Storage driver        | Supported backing filesystems |
| :-------------------- | :---------------------------- |
| `overlay2`, `overlay` | `xfs` with ftype=1, `ext4`    |
| `fuse-overlayfs`      | any filesystem                |
| `aufs`                | `xfs`, `ext4`                 |
| `devicemapper`        | `direct-lvm`                  |
| `btrfs`               | `btrfs`                       |
| `zfs`                 | `zfs`                         |
| `vfs`                 | any filesystem                |

## 二、理解overlay2存储引擎

OverlayFS在单个Linux主机上将两个目录合并成一个目录，这些目录叫做**Layers**（层），这个过程叫做**Union mount**（联合挂载）,将较低的目录叫做**lowerdir**, 较高的目录叫做**upperdir**,合并后的目录通过**merged**目录暴露。

OverlayFS原生支持128层的`lowerdir`,将提升`docker build` `docker commit`的性能，并在主机文件系统上消耗更少的inode.

我们首先拉取示例镜像`docker pull ubuntu`,该镜像包含三个层，在`/var/lib/docker/overlay2`目录下可见4个文件。

```shell
$ ls -l /var/lib/docker/overlay2
total 16
drwx------ 4 root root 4096 Dec 14 18:48 6bcbe401bf9563cef765537d56d6b41614916fea9c9a4a04da7740df22be2b0a
drwx------ 3 root root 4096 Dec 14 18:48 ec043c375f562d2450b6ec600df751d3c175d45928d65bbec06f29de2021c022
drwx------ 4 root root 4096 Dec 14 18:48 fdb1a5544aa14447881a4168a34715387a4e1eb2d4bead7111987cc8eb779498
drwx------ 2 root root 4096 Dec 14 18:48 l
```

`l`目录下以符号链接的方式重新组织了层信息，避免`mount`参数超限。

```shell
$ ls -l /var/lib/docker/overlay2/l
total 12
lrwxrwxrwx 1 root root 72 Dec 14 18:48 4BXWJGCYLNYIKGVSNAC2FTDV7C -> ../fdb1a5544aa14447881a4168a34715387a4e1eb2d4bead7111987cc8eb779498/diff
lrwxrwxrwx 1 root root 72 Dec 14 18:48 HS3PTDGILSMO7PRIHEHCPO2FGU -> ../ec043c375f562d2450b6ec600df751d3c175d45928d65bbec06f29de2021c022/diff
lrwxrwxrwx 1 root root 72 Dec 14 18:48 U6CEQQLZFHHVL62ROBOALZJUSR -> ../6bcbe401bf9563cef765537d56d6b41614916fea9c9a4a04da7740df22be2b0a/diff
```

最底层包含一个文件`link`和一个目录`diff`, `link`文件里存放了符号链接的标识符，`diff`目录下存放层内容。

```shell
$ cat /var/lib/docker/overlay2/ec043c375f562d2450b6ec600df751d3c175d45928d65bbec06f29de2021c022/link
HS3PTDGILSMO7PRIHEHCPO2FGU
$ ls /var/lib/docker/overlay2/ec043c375f562d2450b6ec600df751d3c175d45928d65bbec06f29de2021c022/diff
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

倒数第二层及上面各层均含有`lower`文件，存放着联合挂载信息；`diff`目录包含有该层文件内容；`merge`目录包含该层与父级目录合并后的文件内容。`work`目录是OverlayFS内置的。

```shell
$ tree -L 2 /var/lib/docker/overlay2/
/var/lib/docker/overlay2/
├── 6bcbe401bf9563cef765537d56d6b41614916fea9c9a4a04da7740df22be2b0a
│   ├── diff
│   ├── link
│   ├── lower
│   └── work
├── ec043c375f562d2450b6ec600df751d3c175d45928d65bbec06f29de2021c022
│   ├── committed
│   ├── diff
│   └── link
├── fdb1a5544aa14447881a4168a34715387a4e1eb2d4bead7111987cc8eb779498
│   ├── committed
│   ├── diff
│   ├── link
│   ├── lower
│   └── work
└── l
    ├── 4BXWJGCYLNYIKGVSNAC2FTDV7C -> ../fdb1a5544aa14447881a4168a34715387a4e1eb2d4bead7111987cc8eb779498/diff
    ├── HS3PTDGILSMO7PRIHEHCPO2FGU -> ../ec043c375f562d2450b6ec600df751d3c175d45928d65bbec06f29de2021c022/diff
    └── U6CEQQLZFHHVL62ROBOALZJUSR -> ../6bcbe401bf9563cef765537d56d6b41614916fea9c9a4a04da7740df22be2b0a/diff
    
$ cat /var/lib/docker/overlay2/fdb1a5544aa14447881a4168a34715387a4e1eb2d4bead7111987cc8eb779498/lower
l/HS3PTDGILSMO7PRIHEHCPO2FGU

$ ls /var/lib/docker/overlay2/fdb1a5544aa14447881a4168a34715387a4e1eb2d4bead7111987cc8eb779498/diff
etc  usr  var
```

当容器启动后，Docker会使用Overlay2引擎挂载容器目录。我们运行`docker run`命令看一下实际效果

```SHELL
$ docker run -d ubuntu:latest sleep 3600
566450ecf2684dc2a033beb49b34cce92d003674ec5a8d606810d2b68fa489de
```

```shell
$ mount | grep overlay2
overlay on /var/lib/docker/overlay2/fea4e43477579fa2e2f5a4b287592e9319a8c17c5e3d659199ba56b02f580ef9/merged type overlay (rw,relatime,
lowerdir=/var/lib/docker/overlay2/l/H3E7H2MFUWPVV3NLOJBVDFJED3:/var/lib/docker/overlay2/l/J3EFNT6RSZDLTHG524LM42JLJT:/var/lib/docker/overlay2/l/TDFRV2Z66UAACTA3KP5KLQLADK:/var/lib/docker/overlay2/l/ALV2R4IDYWACLQRARWKANDDPW7,
upperdir=/var/lib/docker/overlay2/fea4e43477579fa2e2f5a4b287592e9319a8c17c5e3d659199ba56b02f580ef9/diff,
workdir=/var/lib/docker/overlay2/fea4e43477579fa2e2f5a4b287592e9319a8c17c5e3d659199ba56b02f580ef9/work)
```

第二行中`rw`表示该挂载目录可读可写。

`/var/lib/docker/overlay2/fea4e43477579fa2e2f5a4b287592e9319a8c17c5e3d659199ba56b02f580ef9/merged`目录即是联合挂载后的`视图`，也就是在容器中看到的文件结构。容器有多个`lowerdir`（只读层），只有一个`upperdir`（可读写层），当在容器中真正`更新` `创建` `删除` 文件时，内容都保存在`upperdir` 目录中。

## 三、Container对Overlay2上文件的读写

下图是docker存储结构与OverlayFS存储结构的映射关系

![overlayfs lowerdir, upperdir, merged](https://docs.docker.com/storage/storagedriver/images/overlay_constructs.jpg)

### 1.Read

* 文件在Container layer不存在，则从Image layer读取；（file1）
* 文件在Container layer存在，则直接从Container layer读取（file2, file4）

### 2.Write

* **第一次写入一个文件**。如果文件已存在，但并不存在于Container layer中，则overlay会进行一次copy_up操作，先将文件内容从Image layer(lowerdir)拷贝到Container layer(upperdir)中，然后对Container layer上的文件进行写入操作。

> OverlayFS是工作在文件级别的，而不是块级别的，因此copy_up操作会将整个文件进行一次拷贝，即使这个文件非常大而我们只修改一个小地方。因此container的第一次写性能还是有一定损耗的。

* **其他时间写入文件**。则直接操作Container layer中对应文件。

* **删除文件和目录**。当删除文件时，Container layer中建立一个`whiteout`文件，而Image layer中该文件不会被删除，因为是Read-Only的。`Whiteout`文件会阻止对此文件的访问。
* 当我们删除目录时，会建立一个`opaque`目录。

## 四、总结

我们可以把层看作是有文件的目录，多个目录间有至上而下的依赖关系，然后通过联合挂载的文式`合并`成一个视图（目录），再将这个合并后的目录以RootFS方式挂载到docker容器进程中去。

我们的写操作都在Container layer层上进行，不会影响镜像层。另外，我们在容器里边删除Image layer中文件，并不是真正的删除，而是建立一个`whiteout`文件（套用大V的话来说就叫：白障）来阻止对相应文件的访问。


