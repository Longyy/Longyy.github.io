# 容器镜像之RootFS解析


我们知道Docker使用Namespace对容器进程进行隔离，让容器进程只能看到该Namespce内的“世界”；同时使用Cgroups来限制容器资源，那容器内的文件系统呢，难到要继承宿主机的文件系统吗？显然不是，不然宿主机上多个容器之间岂不乱套了。实际上，Docker使用了Mount Namespace和RootFS技术，对文件系统进行了隔离, 使得容器可以拥有完全独立的文件系统。

## Mount Namespace

为了理解其原理，我使用某大V的一段C程序：

```c

#define _GNU_SOURCE
#include <sys/mount.h> 
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];
char* const container_args[] = {
  "/bin/bash",
  NULL
};

int container_main(void* arg)
{  
  printf("Container - inside the container!\n");
  execv(container_args[0], container_args);
  printf("Something's wrong!\n");
  return 1;
}

int main()
{
  printf("Parent - start a container!\n");
  int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWNS | SIGCHLD , NULL);
  waitpid(container_pid, NULL, 0);
  printf("Parent - container stopped!\n");
  return 0;
}
```

该程序通过clone创建一个子进程并声明启用Mount Namespace (CLONE_NEWNS), 子进程创建后执行"/bin/bash"命令，这个shell运行在Mount Namespace的隔离环境中。我们来编译并执行这个程序：

```shell
$ gcc -o ns ns.c
$ ./ns
Parent - start a container!
Container - inside the container!
```

于是，我们进入了“容器”当中，我们接着看一下/tmp目录下的内容：

```shell
$ ls /tmp
# 会看到内容与宿主机上/tmp目录的一模一样
```

新创建的容器会直接继承宿主机上的各个挂载点。并没有起到我们想要的“隔离”的作用。

事实上，要让Mount Namespace起作用，除了在创建容器时进行声明，我们还要告诉容器哪些目录要进行挂载。这里我们修改代码重新挂载/tmp目录：

```c
// ...
int container_main(void* arg)
{  
  printf("Container - inside the container!\n");
  mount("none", "/tmp", "tmpfs", 0, ""); //重新以tmpfs(内存盘)的格式挂载/tmp目录
  execv(container_args[0], container_args);
  printf("Something's wrong!\n");
  return 1;
}
// ...
```

修改后再次编译执行，

```shell
$ gcc -o ns ns.c
$ ./ns
Parent - start a container!
Container - inside the container!
$ ls /tmp
```

发现此时tmp变成了空目录。可以用“df -a”命令查看一下。

```shell
$ df -a | grep tmp
# none              512920        0    512920   0% /tmp
```

我们在宿主机上再次查看：

```shell
$ df -a | grep tmp
# none              512920        0    512920   0% /tmp
```

这是因为很多云服务器上根目录的挂载是shared类型的，可以如下进行查看：

```shell
$ cat /proc/self/mountinfo
22 1 253:1 / / rw,relatime shared:1 - ext4 /dev/vda1 rw,errors=remount-ro,data=ordered
```

为避免这种情况，我们需要在容器中对根目录进行重新挂载：

```c
// ...
int container_main(void* arg)
{  
  printf("Container - inside the container!\n");
  //根目录的挂载类型是shared，需要重新挂载根目录
  //mount("", "/", NULL, MS_PRIVATE, "");
  mount("none", "/tmp", "tmpfs", 0, ""); //重新以tmpfs(内存盘)的格式挂载/tmp目录
  execv(container_args[0], container_args);
  printf("Something's wrong!\n");
  return 1;
}
// ...
```

再次编译执行，发现宿主机中没有/tmp的挂载信息了。

Mount Namespce与其他Namespace不同的地方，它对容器进程视图的改变，一定是有挂载操作（mount）之后才能生效。

## chroot

当创建容器后我们希望在容器里边看到的文件系统是一个独立的隔离环境。

有了Mount Namespace, 我们可以在容器进程开始前挂载根目录/, 这个挂载对宿主机是不可见的。

Linux系统中有命令chroot: change root file system, 可以方便实现这一目的。

我们一起来操作一下：

```shell
# 创建目录
$ mkdir -p $HOME/test
$ mkdir -p $HOME/test/{lib,lib64,bin}
$ mkdir -p $HOME/test/lib/i386-linux-gnu
# 拷贝命令
$ cp -v /bin/{bash,ls} $HOME/test/bin
# 拷贝bash,ls命令所需要的所有.so文件
$ T=$HOME/test
$ list="$(ldd /bin/ls | egrep -o '/lib.*\.[0-9]')"
$ for i in list; do cp -v "$i" "${T}${i}"; done
$ list2="$(ldd /bin/bash | egrep -o '/lib.*\.[0-9]')"
$ for i in list2; do co -v "$i" "${T}${i}"; done
```

最后执行chroot命令，我们将使用$HOME/test目录作为/bin/bash的根目录：

```shell
$ chroot $HOME/test /bin/bash
bash-4.3# ls /
bin  lib  lib64
```

对于被chroot的进程它们并不知道自己的根目录已经被“修改”了。

为了让根目录更真实，我们通常会在根目录下挂载一个完整的操作系统的文件系统，比如Ubuntu16.04的ISO。

这个挂载在容器根目录上，用来给容器提供隔离后执行环境的文件系统，就是“容器镜像”，也叫：rootfs（根文件系统）.


