在大多数情况下，容器在运行Linux操作系统的计算机中运行，了解Linux的一些基本特性将有所帮助，以便您可以了解它们如何影响安全性，特别是它们如何应用于容器。我将介绍系统调用、基于文件的权限和功能，最后讨论特权升级。 如果您熟悉这些概念，可以跳到下一章。

这一点非常重要，因为*容器运行主机可见的Linux进程*。 容器化进程使用系统调用，并以与常规进程相同的方式需要权限和特权。 但是容器为我们提供了一些新的方法来控制这些权限在运行时或容器镜像构建过程中的分配方式，这将对安全性产生重大影响。

## 系统调用

应用程序在用户空间中运行，用户空间的特权级别低于操作系统内核。如果应用程序想要做一些事情，比如访问文件、使用网络进行通信，它必须要求内核代表应用程序执行此操作。 用户空间代码用于对内核发出这些请求的编程接口称为系统调用或syscall。目前在linux的内核中有大约300多个不同的系统调用，数量根据Linux内核的版本而变化。 这里有几个例子:

- read：从文件中读取数据
- write：将数据写入文件
- open：打开文件以供后续读取或写入
- execve： 运行可执行程序
- chown：更改文件的所有者
- clone：创建新流程

应用程序开发人员很少需要直接担心系统调用，因为它们通常包含在更高级别的编程抽象中。 作为应用程序开发人员，您可能会遇到的最低级别的抽象是glibc库或Golang syscall包。 在实践中，这些通常也被更高层次的抽象包裹。

应用程序代码以完全相同的方式使用系统调用，无论它是否在容器中运行，但正如您将在本书后面看到的那样，一个主机上的所有容器共享一个内核--即它们正在对同一个内核进行系统调用--这一事实会产生安全影响。

并非所有应用程序都需要所有系统调用，因此-遵循最小特权原则-有Linux安全功能允许用户限制不同程序可以访问的系统调用集。 您将在第8章中看到如何将这些应用于容器。

我将在第5章回到用户空间和内核级特权的主题。 现在让我们转向Linux如何控制文件权限的问题。

## 文件权限

在任何Linux系统上，无论您是否运行容器，文件权限都是安全的基石。 有一种说法，在Linux中，一切都是一个文件。 应用代码、数据、配置信息、日志等--它们都保存在文件中. 即使是屏幕和打印机等物理设备也表示为文件。 权限在文件上确定允许哪些用户访问这些文件以及他们可以对文件执行哪些操作。 这些权限有时称为自由访问控制或DAC。

让我们仔细检查一下。如果您在Linux终端上花费了很多时间，您可能已经运行ls-l命令来检索有关文

件及其属性的信息。

![image-20240228094515492](https://gitee.com/xw1150/m2sec_image/raw/master/img/202402280945520.png)

图2-1 Linux flie权限示例

在图2-1中的示例中，您可以看到一个名为`myapp`的文件，该文件由名为"liz"的用户拥有，并与组"staff"相关联。"权限属性告诉您用户可以对此文件执行哪些操作，具体取决于他们的身份。 此输出中有九个字符表示权限属性，您应该以三个为一组来考虑这些属性:

- 第一组三个字符描述了拥有文件的用户的权限（本例中为"liz"）。
- 第二个组为文件组的成员（此处为"staff"）授予权限。
- 最终集显示任何其他用户（不是"liz"或"staff"成员）拥有的权限。

用户可以对此文件执行三个操作：读取、写入或执行，具体取决于是否设置了r、w和x位。 每个组中的三个字符表示打开或关闭的位，显示这三个操作中的哪一个是允许的—破折号表示该位未设置。

在此示例中，只有文件的所有者可以对其进行写入，因为w位仅在第一组中设置，表示所有者权限。 所有者可以执行该文件，该组的任何成员"工作人员也是如此。"允许任何用户读取文件，因为r位在所有三个组中设置。

有一个很好的机会，你已经熟悉这些r，w和x位，但这不是故事的结束。 使用setuid、setgid和sticky位会影响权限。 从安全角度来看，前两个很重要，因为它们可以允许进程获得额外的权限，攻击者可能会将其用于恶意目的。

## setuid and setgid

通常，当您执行文件时，启动的进程将继承您的用户ID。 如果文件设置了set uid位，则进程将具有文件所有者的用户ID。 以下示例使用非root用户拥有的sleep可执行文件的副本:

```bash
vagrant@vagrant:~$ ls -l `which sleep`
-rwxr-xr-x 1 root root 35000 Jan 18 2018 /bin/sleep
vagrant@vagrant:~$ cp /bin/sleep ./mysleep
vagrant@vagrant:~$ ls -l mysleep
-rwxr-xr-x 1 vagrant vagrant 35000 Oct 17 08:49 mysleep
```

`ls`输出显示副本由名为`vagrant`的用户拥有。 通过执行`sudo sleep 100`在root下运行它，在第二个终端中，您可以查看正在运行的进程—100意味着在进程终止之前，您将有100秒的时间来执行此操作（为了清楚起见，已经删除了一些不必要的行）

```bash
vagrant@vagrant:~$ ps ajf
 PPID PID PGID SID TTY TPGID STAT UID TIME COMMAND
 1315 1316 1316 1316 pts/0 1502 Ss 1000 0:00 -bash
 1316 1502 1502 1316 pts/0 1502 S+ 0 0:00 \_ sudo ./mysleep 100
 1502 1503 1502 1316 pts/0 1502 S+ 0 0:00 \_ ./mysleep 100
```

UID为0表示sudo进程和mysleep进程都在根UID下运行。 现在让我们尝试打开setuid位:

```bash
vagrant@vagrant:~$ chmod +s mysleep
vagrant@vagrant:~$ ls -l mysleep
-rwsr-sr-x 1 vagrant vagrant 35000 Oct 17 08:49 mysleep
```

再次运行`sudo ./mysleep 100`，并从第二个终端再次查看正在运行的进程:

```bash
vagrant@vagrant:~$ ps ajf
PPID PID PGID SID TTY TPGID STAT UID TIME COMMAND
1315 1316 1316 1316 pts/0 1507 Ss 1000 0:00 -bash
1316 1507 1507 1316 pts/0 1507 S+ 0 0:00 \_ sudo ./mysleep 100
1507 1508 1507 1316 pts/0 1507 S+ 1000 0:00 \_ ./mysleep 100
```

Sudo进程仍以root身份运行，但这次mysleep从文件的所有者处获取了其用户

ID。

此位通常用于为程序提供所需的权限，但通常不会扩展到普通用户。 如经常用到的命令`ping`。它需要打开原始网络套接字的权限才能发送其ping消息。（用于授予此权限的机制是一种功能，可在下文中"Linux功能"中查看。）管理员可能会为他们的用户运行ping而感到高兴，但这并不意味着他们愿意让用户为他们可能想到的任何其他目的打开原始网络套接字。 相反，ping executable通常与setuid位集一起安装并由root用户拥有，以便ping可以使用通常与root用户相关联的特权。

我在上一句话中仔细选择了措辞。后面你会看到，在这一部分中，ping实际上穿越了一些环节，以避免以root身份运行。在我讲这个之前，让我们看一下setuid位是如何运作的。 

你可以通过取你自己的一个非root用户的副本，来实验运行ping所需的权限。你要ping的是否是一个可以到达的地址并没有真正的关系；重点是看看ping是否有足够的权限来打开原始网络套接字。请检查你是否能够按预期运行ping：

```bash
vagrant@vagrant:~$ ping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
^C
--- 10.0.0.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1017ms
```

确定您可以以非root用户身份运行ping后，获取副本并查看是否也可以运行:

```bash
vagrant@vagrant:~$ ls -l `which ping`
-rwsr-xr-x 1 root root 64424 Jun 28 11:05 /bin/ping
vagrant@vagrant:~$ cp /bin/ping ./myping
vagrant@vagrant:~$ ls -l ./myping
-rwxr-xr-x 1 vagrant vagrant 64424 Nov 24 18:51 ./myping
vagrant@vagrant:~$ ./myping 10.0.0.1
ping: socket: Operation not permitted
```

当您复制可执行文件时，文件所有权属性将根据您操作的用户ID进行设置，并且setuid位不会结转。 作为普通用户运行此`myping`没有足够的权限来打开原始套接字。 如果您仔细检查权限位，您可以看到原始ping具有`s`或setuid位，而不是常规`x`。

您可以尝试将文件的所有权更改为root（您需要sudo才能允许执行此操作），但仍然可执行文件没有足够的权限，除非您以root身份运行

```bash
vagrant@vagrant:~$ sudo chown root ./myping
vagrant@vagrant:~$ ls -l ./myping
-rwxr-xr-x 1 root vagrant 64424 Nov 24 18:55 ./myping
vagrant@vagrant:~$ ./myping 10.0.0.1
ping: socket: Operation not permitted
vagrant@vagrant:~$ sudo ./myping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
^C
--- 10.0.0.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1012ms
```

现在在可执行文件上设置setuid位，然后重试:

```bash
vagrant@vagrant:~$ sudo chmod +s ./myping
vagrant@vagrant:~$ ls -l ./myping
-rwsr-sr-x 1 root vagrant 64424 Nov 24 18:55 ./myping
vagrant@vagrant:~$ ./myping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
^C
--- 10.0.0.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2052ms
```

正如下文中"Linux功能"中看到的那样，还有另一种方法可以为`myping`提供足够的权限来打开套接字，而无需可执行文件具有与root关联的所有权限。

现在，这个ping的运行副本工作，因为它具有setuid位，这允许它以root身份操作，但是如果您使用第二个终端查看使用ps的过程，您可能会对结果感到惊讶:

```bash
vagrant@vagrant:~$ ps uf -C myping
USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
vagrant 5154 0.0 0.0 18512 2484 pts/1 S+ 00:33 0:00 ./myping localhost
```

如你所见，尽管setuid位已打开并且文件是由root所有，但是进程并未以root身份运行。这是怎么回事呢？答案是，在现代版本的ping中，可执行文件是以root身份开始运行的，但它会明确设置它所需的能力，然后将其用户ID重置为原始用户的ID。这就是我在本节早些时候所说的“穿越了一些环节”。

> 如果您想更详细地了解这一点，可以使用strace查看ping（或myping） 执行进行的系统调用。 找到shell的进程ID，然后在以root身份运行的second终端中`strace -f -p <shell 进程ID>`将跟踪来自该shell内的所有系统调用，包括在其中运行的任何可执行文件。 查找setuid()系统调用，它重置用户ID。 您将看到，在一些setcap()系统调用设置线程需要的功能后不久，会发生这种情况。

并非所有可执行文件都以这种方式写入重置用户ID。 您可以使用本章前面部分的sleep副本来查看更多正常的setuid行为。 更改对root的所有权，设置setuid位（当您更改所有权时，这会重置），然后以非root用户身份运行它:

```bash
vagrant@vagrant:~$ sudo chown root mysleep
vagrant@vagrant:~$ sudo chmod +s mysleep
vagrant@vagrant:~$ ls -l ./mysleep
-rwsr-sr-x 1 root vagrant 35000 Dec 2 00:36 ./mysleep
vagrant@vagrant:~$ ./mysleep 100
```

在另一个终端中，可以使用ps查看此进程正在root的用户ID下运行:

```bash
vagrant@vagrant:~$ ps uf -C mysleep
USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
root 6646 0.0 0.0 7468 764 pts/2 S+ 00:38 0:00 ./mysleep 100
```

现在您已经尝试了setuid位，您可以很好地考虑它的安全含义。

**Setuid的安全影响**

想象一下，如果你将setuid设置在，比如说，bash上。任何运行它的用户都会在一个作为root用户运行的shell中。实际上，这并不像看上去那么简单，因为大多数shell的行为很像ping，它们会重置他们的用户ID，以避免被用于这样的简单权限提升。但是，编写一个基于自身设置setuid然后，在已经变为root的状态下，调用shell的程序是非常容易的。

由于setuid提供了一条危险的权限升级路径，一些容器镜像扫描器（在第7章中涵盖）会报告具有setuid位设置的文件的存在。你也可以阻止它被使用`--no-new-privileges`标志在docker运行命令。 

setuid位的时间可以追溯到权限更为简单的时期——你的进程要么有root权限，要么没有。setuid位提供了一种为非root用户授予额外权限的机制。Linux内核的2.2版本引入了对这些额外权限更精细的控制通过能力。

## Linux功能

今天的Linux内核中有超过30种不同的功能。 可以将功能分配给线程，以确定该线程是否可以执行某些操作。 例如，线程需要`CAP_NET_BIND_SERVICE`功能才能绑定到低编号（低于1024）端口。 `CAP_SYS_BOOT`的存在使得任意可执行文件没有重新启动系统的权限。 加载或卸载内核模块需要`CAP_SYS_MODULE`。

我之前提到过，ping工具以root身份运行的时间足够长，可以为自己提供所需的功能，允许线程打开原始网络套接字。 此特定功能称为`CAP_NET_RAW`。

您可以使用getpcaps命令查看分配给进程的功能。

例如，由非root用户运行的进程通常不具有功能:

```
vagrant@vagrant:~$ ps
 PID TTY TIME CMD
22355 pts/0 00:00:00 bash
25058 pts/0 00:00:00 ps
vagrant@vagrant:~$ getpcaps 22355
Capabilities for '22355': =
```

如果您以root身份运行进程，则完全是另一回事:

```bash
vagrant@vagrant:~$ sudo bash
root@vagrant:~# ps
 PID TTY TIME CMD
25061 pts/0 00:00:00 sudo
25062 pts/0 00:00:00 bash
25070 pts/0 00:00:00 ps
root@vagrant:~# getpcaps 25062
Capabilities for '25062': = cap_chown,cap_dac_override,cap_dac_read_search,
cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap
cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,
cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,
cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,
cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,
cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override
cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read+ep
```

文件可以直接分配功能。早些时候，你看到一个ping的副本不能在一个非root用户下运行，除非有setuid位。还有另一种方法：将它需要的功能直接分配给可执行文件。拿一个ping的副本并检查它是否有正常的权限（没有setuid位）。这是不允许打开套接字的:

```bash
vagrant@vagrant:~$ cp /bin/ping ./myping
vagrant@vagrant:~$ ls -l myping
-rwxr-xr-x 1 vagrant vagrant 64424 Feb 12 18:18 myping
vagrant@vagrant:~$ ./myping 10.0.0.1
ping: socket: Operation not permitted
```

使用`setcap`将`CAP_NET_RAW`功能添加到文件中，这将授予它打开原始网络套接字的权限。 您将需要root权限才能更改功能。 更确切地说，您只需要`CAP_SETFCAP`功能，但这是自动授予root的:

```bash
vagrant@vagrant:~$ setcap 'cap_net_raw+p' ./myping
unable to set CAP_SETFCAP effective capability: Operation not permitted
vagrant@vagrant:~$ sudo setcap 'cap_net_raw+p' ./myping
```

这对ls显示的权限没有影响，但您可以使用getcap检查功能

```bash
vagrant@vagrant:~$ ls -l myping
-rwxr-xr-x 1 vagrant vagrant 64424 Feb 12 18:18 myping
vagrant@vagrant:~$ getcap ./myping
./myping = cap_net_raw+p
```

此功能允许ping的副本操作:

```
vagrant@vagrant:~$ ./myping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
^C
```

遵循最小特权原则，只授予流程完成其工作所需的功能是一个好主意。 运行容器时，您可以选择控制允许的功能，如第8章所示。

现在您已经熟悉了权限和特权的基本概念Linux，接下来介绍一个关于特权升级的相关概念。

## 特权升级

“**权限升级**”一词意味着超越你应该拥有的权限，这样你就可以采取你不应该被允许的行动。为了提升他们的权限，攻击者利用系统漏洞或糟糕的配置为自己授予额外的权限。 往往，攻击者开始时是一个非特权用户，希望在机器上获得root权限。提升权限的常见方法是寻找已经作为root运行的软件，然后利用软件中已知的漏洞。例如，web服务器软件可能包含一个漏洞，允许攻击者远程执行代码，比如Struts漏洞。如果web服务器作为root运行，任何被攻击者远程执行的东西都将以root权限运行。因此，尽可能以非特权用户运行软件是个好主意。 

正如你稍后在这本书中将学到的，默认情况下，容器作为root运行。这意味着，与传统的Linux机器相比，在容器中运行的应用程序更有可能作为root运行。一个可以控制容器内部进程的攻击者仍然需要以某种方式逃离容器，但一旦他们实现了这一点，他们就会在主机上成为root，不需要任何进一步的权限升级。第9章将会更详细地讨论这个问题。

 即使一个容器是以非root用户运行的，也有可能基于你在本章早些时候看到的Linux权限机制进行权限升级：

- 包含有setuid二进制的容器镜像 
- 作为非root用户运行的容器被授予的额外能力 

你将在本书后面了解到关于缓解这些问题的方法。

## Summary

在本章中，我们已经学习了一些基本的Linux机制，这些机制对于理解本书后面的章节至关重要。 它们还以多种方式在安全性方面发挥作用，容器安全控件都是建立在这些基础之上的。

现在我们已经掌握了一些基本的Linux安全控制，现在是时候开始研究组成容器的机制了，这样就可以自己理解主机上和容器中的root是如何一回事了。
