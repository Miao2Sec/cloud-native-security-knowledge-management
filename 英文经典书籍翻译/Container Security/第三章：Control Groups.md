# 第三章：Control Groups

## 0x00 前言

本章我们来说说，用于制作容器的基本构建块之一：控制组，也就是我们常说的 cgroup。cgroup可以限制进程可以使用的资源，例如内存、CPU和网络输入（输出）。

从安全的角度来看，经过良好调优的cgroup可以确保一个进程不会通过占用所有资源来影响其他进程的行为，例如使用所有的CPU或内存来饿死其它的应用程序。

此外，还有一个名为 `pid` 的控制组，可以用来限制控制组内允许的总进程数，这样可以有效地防止fork炸弹的有效性。

> 🚦**提示**
>
> fork炸弹会快速创建进程，进而创建更多进程，导致资源使用呈指数级增长，最终导致机器瘫痪。详情请看视频：What Have Namespaces Done for You Lately?[^1]

我们在第4章了解到，容器作为常规的 Linux 进程运行，应该使用 cgroup 来限制它可以使用的资源。接下来，让我们看看cgroup 是如何组织的。

本文所使用cgroup 均为v1，实验环境如下所示：

```bash
cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.4 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

```bash
uname -r
5.15.0-43-generic
```

```bash
stat -fc %T /sys/fs/cgroup/
cgroup2fs
```

也有提到 cgroup v2，具体实验环境如下所示：

```bash
cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"

```

```bash
uname -r
3.10.0-957.el7.x86_64
```

```bash
stat -fc %T /sys/fs/cgroup/
tmpfs
```

## 0x01 Cgroup结构

每种被管理的资源都有一个控制组结构，每个结构都由**cgroup 控制器**（cgroup  controller） 管理。任何 Linux 进程都是各种cgroup类型的成员，并且进程在首次创建时，会继承其父进程的 cgroup。

> 🚦**提示**
>
> cgroup可以分为不同的类型，每种类型用于控制特定资源。例如：用于控制CPU资源的cgroup、用于控制内存资源的cgroup等。进程可以成为这些不同类型的cgroup的成员，使系统可以对它们进行资源管理。

Linux内核通过一组伪文件系统来传递有关cgroup的信息，这些伪文件系统通常位于`/sys/fs/cgroup`。我们可以通过列出该目录的内容来查看系统上不同类型的cgroup：

> 🚦**提示**
>
> 伪文件系统（Pseudo File System）通常是指一种虚拟的文件系统，它并不直接映射到物理存储设备上，而是通过操作系统或应用程序的抽象层来模拟文件系统的行为。这种文件系统的目的可能是提供对特定信息或资源的访问，或者为应用程序提供一种统一的接口，使它们能够以文件系统的方式与不同类型的数据进行交互。

```bash
ls /sys/fs/cgroup/
```

![查看 Linux 主机上的 cgroup](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725421.png)

管理 cgroup 涉及到读取和写入cgroup结构中的文件和目录。让我们以cgroup v1 中的 memory cgroup 为例：

```bash
ls /sys/fs/cgroup/memory/
```

![查看 Linux 主机上的 memory cgroup](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725459.png)

我们可以通过改写其中的一些文件来操作cgroup，而有些文件是内核写入的用来提供关于cgroup状态数据。

在不查阅文档[^2]的情况下，我们很难立即确定哪些是参数，哪些是信息，但我们可以根据文件名称猜测其中一些文件的功能。例如：

- `memory.limit_in_bytes` 包含一个可写的值，用于设置组中进程可用的内存量；
- `memory.max_usage_in_bytes` 提供了组内内存使用的高水位标记。

![查看memory.limit_in_bytes 和 memory.max_usage_in_bytes](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725432.png)

`memory` 目录是这个cgroup结构的最顶层，在没有其他 cgroup 的情况下，它将保存所有运行中的进程的内存信息。

如果想限制进程的内存使用，则需要创建一个新的cgroup，然后将进程分配给它。

## 0x02 创建 Cgroup

在 `memory` 目录中创建子目录，会创建一个cgroup，内核会自动将文件填充到该目录下，这些文件就是这个cgroup的参数和统计信息。

```bash
cd /sys/fs/cgroup
mkdir memory/miao2sec && ls memory/miao2sec

```

![创建 cgroup](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725411.png)

> 🚥**提示**
>
> 删除 cgroup 使用`sudo rm -rf <dir-name>`会提示`Operation not permitted`而失败，正确的命令为 `sudo rmdir <dir-name>`[^3]

这些文件的具体含义已经超出本文的范围了，但其中的一些文件包含我们可以操作的参数，以定义控制组的限制，而其他文件则传递有关控制组当前资源使用情况的统计信息。

我们可以做一个合理的猜测，例如，`memory.usage_in_bytes` 是描述控制组当前使用的内存量的文件。

![查看 emory.usage_in_bytes](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725383.png)

当我们启动一个容器时，运行时会为其创建新的cgroup。我们可以使用`lscgroup`工具来查看主机上的cgroup。

由于cgroup的数量太多了，我们可以在启动容器前后分别进行快照。

> 🚥**提示**
>
> 1. 安装Docker 请参考官方文档[^4]。
> 2. ubuntu 上可以通过cgroup-tools来获取lscgroup，centos可以通过安装libcgroup-tools来获取lscgroup。
> 3. Docker使用runc作为其底层容器运行时，因此在安装Docker时，runc已经包含在其中，无需单独安装。

1. 在容器启动之前，对主机上的cgroup进行快照。

    ```bash
    lscgroup memory:/ > before.memory
    ```

2. 新开一个终端，在新终端中新建一个容器。

    ```bash
    docker create --name alpine alpine:latest
    ```

    ![创建 alpine 容器](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725314.png)
3. 导出容器`alpine`的rootfs

    ```bash
    docker export -o rootfs.tar alpine
    mkdir rootfs
    tar -xf rootfs.tar -C ./rootfs
    ```

4. 使用runc生成一个`config.json`容器配置文件。

    ```bash
    runc spec
    ```

5. 有了`rootfs` 和`config.json`过后，我们就可以使用runc启动容器。

    ```bash
    runc run miao2sec
    ```

    ![使用 runc 启动容器](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725358.png)
6. 在容器启动之后，回到原来的终端上，对主机上的cgroup 进行快照。

    ```bash
    lscgroup memory:/ > after.memory
    ```

7. 对比容器启动前后的快照，发现启动一个容器后，新创建了一个cgroup

    ```bash
    diff before.memory after.memory
    ```

    ![对比容器启动前后的快照](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725469.png)
8. 新创建的cgroup与 `memory cgroup`的根目录相关，根目录通常为`/sys/fs/cgroup/memory`。当容器仍在运行，我们可以从主机上直接查看新创建的cgroup的内容。

    ```bash
    ls /sys/fs/cgroup/memory/user.slice/miao2sec/
    ```

    ![查看容器新创建的cgroup内容](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725305.png)
9. 在容器内部，我们也可以通过查看 `/proc` 目录来访问cgroup列表。其中，`$$`表示当前进程的`PID`。

    ```bash
    cat /proc/$$/cgroup
    ```

    ![在容器中查看当前进程的cgroup](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725450.png)

请注意，从主机的角度上来看，memory cgroup正是我们所找到的。一旦我们拥有一个cgroup，就可以通过写入适当的文件来修改其参数。

我们可能对 cgroup列表中的 `user.slice` 部分感到疑惑。这与systemd有关系。它为自己的资源控制方法自动创建了一些cgroup结构，如果对此很感兴趣，请阅读Red Hat的官方文档中的相关描述[^5]。

## 0x03 设置资源限制

我们可以通过查看`memory.limit_in_bytes`文件的内容来查看cgroup可用的内存大小。默认情况下，内存是没有限制的，因此这个很大的数字表示是`PAGE_COUNTER_MAX`，在64位平台上是`LONG_MAX/PAGE_SIZE`。当平台的`PAGE_SIZE`不同时，cgroup内存的默认值也不同[^6][^7]。

如果一个进程可以消耗无限量的内存，那么同一主机上其他的进程可能会被饿死。这可能是由应用程序中的内存泄漏无意中导致的，也可能是由于资源耗尽攻击[^8]，利用内存泄漏故意尽可能多地使用内存。

通过对一个进程可以访问的内存和其他资源设置限制，可以减轻这种攻击的影响，并确保其他进程可以正常运行。

若要限制容器的`cgroup` 内存，我们需要先停止容器。然后修改`config.json`文件。

1. 在`config.json`的`.linux.resources`部分新增`memory.limit`字段。
    ![新增 memory.limit 字段](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725391.png)
2. 重新启动容器来重载配置。现在，我们发现`memory.limit_in_bytes`的值是我们所配置的最接近的KB值了。

    ![容器内查看 memory.limit_in_bytes ](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725400.png)

    ![容器外查看 memory.limit_in_bytes ](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725333.png)

## 0x04 将进程分配给cgroup

1. 创建 memory类型的cgroup，并命名为miao2sec；

    ```bash
    mkdir /sys/fs/cgroup/memory/miao2sec
    ```

2. 查看该cgroup中的进程可用的内存量；

    ```bash
    cat /sys/fs/cgroup/memory/miao2sec/memory.limit_in_bytes
    9223372036854771712
    ```

3. 修改进程可用的内存量；

    ```bash
    echo 10000 > /sys/fs/cgroup/memory/miao2sec/memory.limit_in_bytes
    ```

4. 查看修改是否生效；

    ```bash
    cat /sys/fs/cgroup/memory/miao2sec/memory.limit_in_bytes
    8192
    ```

5. 新开一个终端，使用 sh；

    ```bash
    /bin/sh
    sh-4.2#
    ```

6. 在原来的终端中查看 sh的进程ID；

    ```bash
    ps -C sh
      PID TTY          TIME CMD
    31485 pts/2    00:00:00 sh
    ```

7. 查看属于该cgroup的进程ID列表，发现为空；

    ```bash
    cat /sys/fs/cgroup/memory/miao2sec/cgroup.procs
    ```

8. 就像设置资源限制一样，若要将进程分配给cgroup，只需要将进程ID写入该cgroup的cgroup.procs文件中；

    ```bash
    echo 31485 > /sys/fs/cgroup/memory/miao2sec/cgroup.procs
    ```

9. 查看是否成功将进程分配给了cgroup；

    ```bash
    cat /sys/fs/cgroup/memory/miao2sec/cgroup.procs
    31485
    ```

10. 查看 sh 进程的cgroup 信息，发现已经该进程已经属于miao2sec了

    ```bash
    cat /proc/31485/cgroup | grep memory
    10:memory:/miao2sec
    ```

11. sh现在已经是miao2sec的成员了，其内存限制在10KB以下。这并不是一个很大的可供使用的空间，因此若在sh内运行ls命令都会超过cgroup的内存限制，接着就被杀掉了。

    ```bash
    sh-4.2# ls
    Killed
    ```

![ls 因被限制内存而被 Killed 掉](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725250.png)

## 0x05 使用Cgroup的Docker

我们已经知道了，如何通过修改特定类型资源的 cgroup 文件来操作 cgroup。在 Docker 中，可以很容易地看到这一过程的实际应用。

> 🚦提示
>
> 我们需要在Linux主机（虚拟机）上直接运行Docker。如果使用Docker for Mac/Windows，这些示例将无法生效。因为Docker for Mac/ Windows是在一个虚拟机中运行的，Docker 守护进程和容器在该虚拟机内使用一个单独的内核运行。

Docker会自动创建各种类型的`cgroup`，我们可以在`cgroup`的结构中查找名为`docker`的文件和子目录。

```bash
ls */docker | grep docker
```

![查看 docker 创建的 cgroup](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725342.png)

当我们启动一个容器时，Docker会在`docker` 的`cgroup`中创建另一组`cgroup`。接下来，我们来验证一下：

1. 在后台启动一个一次性的容器，并给它一个内存限制，让他休眠一段时间。

    ```bash
    docker run --rm --memory 100M -d alpine sleep 10000
    ```

    ![运行一个一次性的容器](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725371.png)
2. 查看 `docker` 的`cgroup` 结构，发现Docker已经为这个容器创建了一个新的`cgroup`，并且以容器ID作为`cgroup`名称

    ```bash
    ls /sys/fs/cgroup/memory/docker/e93d5ba1ed3ded6fa7d89f5acc002a669c3a058c207e14bfbfb694f5eeb24266
    ```

    ![查看 docker 创建的 cgroup](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725441.png)
3. 然后查看容器的`cgroup`的内存限制，发现这个数字不是默认值，已经被设置过了。

    ```bash
    cat /sys/fs/cgroup/memory/docker/e93d5ba1ed3ded6fa7d89f5acc002a669c3a058c207e14bfbfb694f5eeb24266/memory.limit_in_bytes
    ```

    ![查看 cgroup 对 docker 容器的内存限制](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725265.png)
4. 确认 `sleep` 确实是 group 的成员。

    ```bash
    cat /sys/fs/cgroup/memory/docker/e93d5ba1ed3ded6fa7d89f5acc002a669c3a058c207e14bfbfb694f5eeb24266/cgroup.procs
    ps -eaf | grep sleep
    ```

    ![确定 sleep 是该 group 成员](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725294.png)
5. 此外，我们还可以通过以下两种方法查看进程所属的group

    ```bash
    cat /proc/7848/cgroup
    ps -o cgroup 7848
    ```

    ![查看进程的 cgroup](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725279.png)

## 0x06 Cgroup V2

自 2016 年以来，Linux 内核中一直有 `cgroup` 的第 2 版本，而 Fedora 在 2019 年中旬成为第一个默认使用 `cgroup v2` 的 Linux 发行版。

> 🚦**提示**
>
> 1. cgroup 的版本与Linux内核版本有关。若要查看cgroup 版本请使用以下命令[^9]：
>
>     ```bash
>     stat -fc %T /sys/fs/cgroup/
>     ```
>
>     ![查看 cgroup 版本](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725324.png)
>
>     若输出为`tmpfs`，则为v1版本，若输出为`cgroup2fs`，则为 v2 版本。其中：
>     - `-f`：展示文件系统状态，而不是文件状态
>     - `-c`：使用自定义的格式，而不是默认格式输出
>     - `%T`：人类可读性的文件系统类型
> 2. 查看Linux 内核版本
>
>     ```bash
>     uname -r
>     ```
>
>     ![查看内核版本](https://gitee.com/xw1150/m2sec_image/raw/master/img/202403031725479.png)

2020年时，最流行的容器运行时的实现都假设`cgroup`的版本为1，不支持版本 2，并且正在做相关工作，Akihiro Suda 在The current adoption status of cgroup v2 in containers[^10]这篇博客里中很好的总结了这些工作。

cgroup v1 和cgroup v2最大的不同在于：

- 在cgroup v1 中，一个进程可以加入两个不同的组，例如：`/sys/fs/cgroup/memory/mygroup` 和 `/sys/fs/cgroup/pids/yourgroup`。
- 在cgroup v2 中，一个进程不能加入不同的组，但事情会更简单一些，例如：将进程加入：`/sys/fs/cgroup/ourgroup` ，这样，进程就可以受到`ourgroup` 的所有控制器的约束了。

cgroup v2还更好地支持无根容器，以便可以对它们应用资源限制。我们在第9章对Rootless容器进行讲解，这里不作过多解释。

> 🚦
>
> cgroup v2提供了对Rootless容器更好的支持，这主要得益于其设计和功能改进。以下是一些原因：
>
> 1. **更灵活的结构：** Cgroup v2引入了更灵活的结构，允许容器结构更好地与主机的cgroup结构集成。这使得在不同结构的cgroup中更容易实现资源的细粒度控制和限制。
> 2. **统一的结构：** Cgroup v2采用了统一的结构，将不同类型的资源（如CPU、内存）整合在同一结构中。这样，可以更一致地对容器应用资源进行限制，而无需涉及多个结构。
> 3. **Rootless容器支持：** Cgroup v2专门考虑了Rootless容器的需求，使得容器在无需特权的情况下仍能够有效地应用资源限制。这对于提高容器的安全性和可移植性非常重要。

Docker自20.10版本起支持cgroup v2。在cgroup v2上运行Docker还需要满足以下条件：[^11]

- containerd≥ v1.4
- runc≥ v1.0.0-rc91
- Linux 内核≥v4.15（推荐≥v5.2）

请注意，cgroup v2模式与cgroup v1模式略有不同：

| 不同点                    | docker 命令（标志）                            | cgroup v1  | cgroup v2 |
| ---------------------- | ---------------------------------------- | ---------- | --------- |
| 默认cgroup 驱动            | `dockerd --exec-opt native.cgroupdriver` | `cgroupfs` | `systemd` |
| 默认的cgroup namespace 模式 | `docker run --cgroupns`                  | `host`     | `private` |
| docker run 的标志         | `--oom-kill-disable`和`--kernel-memory`   | 支持         | 禁用        |

## 0x07总结

`cgroup`限制了不同Linux进程可用的资源。要利用`cgroup`，并不一定需要使用容器，但Docker和其他容器运行时提供了一个便利的接口来使用它们：在运行容器的地方可以轻松设置资源限制，而这些限制由`cgroup`进行监管。

限制资源可以防范一类攻击，这些攻击试图通过消耗过多的资源来破坏我们的部署，从而使合法的应用程序陷入资源匮乏的场景。建议在运行容器应用程序时设置内存和CPU限制。

既然我们已经了解了容器中资源是如何受限的，我们接下来就可以学习关于构成容器的基本构建块的其他部分了：namespace 和 修改 root 目录。继续阅读第4章，了解它们是如何工作的。

[^1]: [https://www.youtube.com/watch?v=MHv6cWjvQjM](https://www.youtube.com/watch?v=MHv6cWjvQjM "https://www.youtube.com/watch?v=MHv6cWjvQjM")

[^2]: [https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/memory.html](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/memory.html "https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/memory.html")

[^3]: [https://www.cnblogs.com/hanzeng1993/p/16004432.html](https://www.cnblogs.com/hanzeng1993/p/16004432.html "https://www.cnblogs.com/hanzeng1993/p/16004432.html")

[^4]: [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/ "https://docs.docker.com/engine/install/")

[^5]: [https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/7/html/resource\_management\_guide/sec-default\_cgroup\_hierarchies](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/resource_management_guide/sec-default_cgroup_hierarchies "https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/resource_management_guide/sec-default_cgroup_hierarchies")

[^6]: [https://github.com/torvalds/linux/blob/ea4424be16887a37735d6550cfd0611528dbe5d9/mm/memcontrol.c#L5337](https://github.com/torvalds/linux/blob/ea4424be16887a37735d6550cfd0611528dbe5d9/mm/memcontrol.c#L5337 "https://github.com/torvalds/linux/blob/ea4424be16887a37735d6550cfd0611528dbe5d9/mm/memcontrol.c#L5337")

[^7]: [https://tracker.ceph.com/issues/42059](https://tracker.ceph.com/issues/42059 "https://tracker.ceph.com/issues/42059")

[^8]: [https://en.wikipedia.org/wiki/Resource\_exhaustion\_attack](https://en.wikipedia.org/wiki/Resource_exhaustion_attack "https://en.wikipedia.org/wiki/Resource_exhaustion_attack")

[^9]: [https://blog.csdn.net/renzhe1301/article/details/126262427](https://blog.csdn.net/renzhe1301/article/details/126262427 "https://blog.csdn.net/renzhe1301/article/details/126262427")

[^10]: [https://medium.com/nttlabs/cgroup-v2-596d035be4d7](https://medium.com/nttlabs/cgroup-v2-596d035be4d7 "https://medium.com/nttlabs/cgroup-v2-596d035be4d7")

[^11]: [https://docs.docker.com/config/containers/runmetrics/](https://docs.docker.com/config/containers/runmetrics/ "https://docs.docker.com/config/containers/runmetrics/")
