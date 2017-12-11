---
layout: post
title: namespace初探
---

简单整理了一下namespace

linux中namespace分为6中：<br />

|**namespace** |**系统调用参数**     |**隔离内容**                          |
|--------------|:--------------------|:------------------------------------:|
|UTS           |  CLONE_NEWUTS       |主机名和域名                          |
|IPC           |  CLONE_NEWIPC       |信号量、消息队列和共享内存            |
|PID           |  CLONE_NEWPID       |进程编号                              |
|Network       |  CLONE_NEWNET       |网络设备、网络栈、端口等等            |
|Mount         |  CLONE_NEWNS        |挂载点（文件系统）                    |
|User          |  CLONE_NEWUSER      |用户和用户组                          |


namespace的API包括**clone()**、**setns()**以及**unshare()**，还有/proc下的部分文件。在使用这些API的时候，通常要指定上面六个参数中的一个或多个，通过 | （或）操作实现。<br />
- **clone（）**：创建新进程同时创建namespace<br />
- **setns（）**：加入一个已经存在的namespace<br />
- **unshare（）**：在原先的进程上进行namespace隔离<br />

### UTS（UNIX Time-sharing System） namespace：

提供了主机名和域名的隔离，使得每个容器都拥有独立的主机名和域名，在网络上可被视为一个独立的节点而不是宿主机上的一个进程。

### IPC（Interprocess Communication） namespace：

进程间通信的几种方式：<br />
1. 管道：管道是一种半双工的通信方式，数据只能单向流动，而且只能在具有亲缘关系的进程间使用，通常是指父子进程之间。<br />
2. 有名管道：这也是一种半双工的通信方式，但他允许无亲缘关系进程间的通信。<br />
3. 消息队列：消息队列是消息的链表，存放在内核中并由消息队列标识符标识。消息队列可以克服信号传递信息少、管道只能承载无格式字节流以及缓冲区大小受限等缺点。<br />
4. 信号量：信号量是一个计数器，可以用来控制多个进程对共享资源的访问。通常可以作为一种锁机制，主要是进程间以及同一进程内不同线程之间的同步手段。<br />
5. 信号：信号是一种比较复杂的通信方式，用户通知接收进程某个事件已经发生。<br />
6. 共享内存：共享内存就是映射一段能被其他进程所访问的内存，由一个进程创建，但多个进程哭访问，这是最快的IPC方式，通常可以与其他通信机制配合使用，来实现进程间通信和同步。<br />
7. 套接字通信：通过socket来同学，可用于不同机器间的进程间通信。<br />
容器内部进程间通信对于宿主机来说，实际上是具有相同PID namespace中的进程间通信，它由一个唯一的32位ID的标识符来进行区别，每个IPC资源都对应了一个这样的ID。IPC namespace实际上是包含了系统IPC标识符和实现POSIX消息队列的文件系统，一个IPC namespace下的进程相互可见，不同的namespace下进程互相不可见。<br />

### PID namespace：

它对进程PID重新编号，每个PID namespace都有自己的计数程序。内核为所有的PID namespace维护了一个树状结构，最顶层是系统启动时创建的，称之为root namespace，它创建的新的namespace都称为child namespace，都是树的子节点。通过这种方式，不同的PID namespace会形成衣服等级体系。父节点可以看到子节点中的进程，并可以通过信号等方式对子节点中的进程产生影响，而子节点不能看到父节点PID namespace中的任何内容。由此可以发现以下特点：<br />
1. 每个PID namespace中的第一个进程PID都是1，都像linux中的init进程一样拥有特权，起特殊作用，也就是dockerinit。<br />
2. 一个namespace中的进程，不可能通过kill或ptrace影响父节点或者兄弟节点中的进程。<br />
3. 如果你在新的PID namespace中重新挂载/proc文件系统，会发现目录下只显示同属一个PID namespace中的其他进程。<br />
4. 在root namespace中可以看到所有的进程，并且递归包含所有子节点中的进程。<br />

**注意点：**<br />
1. PID namespace中的init进程拥有信号屏蔽特权，如果init中没有写处理某个信号的代码逻辑，那么与init同一个PID namespace下的所有进程发送给他的信号都会被屏蔽，防止被误杀。父节点发送的信号如果不是SIGKILL（销毁进程）或SIGSTOP（暂停进程）也会被忽略，但如果父节点发送上述两个信号，子节点的init会强制执行，父节点有权终止子节点中的进程。<br />
2. unshare（）和unset（）只有在创建PID namespace时不会直接进入新的namespace，创建其他namespace时都会直接进入。unshare（）允许用户在原有进程执行namespace隔离，但创建PID namespace后，原进程并不会进入新的PID namespace中，接下来创建的子进程才会进入新的namespace并成为这个namespace的init进程。setns（）和unshare（）类似，调用者进程不会进入新的namespace，而随后创建的子进程将会以init进程的身份进入。<br />

### Mount namespace：

通过隔离文件系统挂载点来提供隔离文件系统的支持，它是linux历史上第一个namespace，所以它的标识符比较特殊，CLONE_NEWNS。进程创建mount namespace时，会把当前文件结构复制给新的namespace。新的namespace中的所有mount操作只影响自身的文件系统，不会对外部产生任何影响。<br />
挂载对象之间的关系有以下两种：<br />
1. 共享关系：如果两个挂载对象具有共享关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，反之亦然。<br />
2. 从属关系：如果两个挂载对象形成从属关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，但是反过来不行，这种关系中，从属对象是事件的接收者。<br />

一个挂载对象的挂载状态可能为如下的一种：<br />
1. 共享挂载<br />
2. 从属挂载<br />
3. 共享/从属挂载<br />
4. 私有挂载<br />
5. 不可绑定挂载<br />
传播事件的挂载对象称为共享挂载，接收传播事件的挂载对象为从属挂载，既不传播也不接收的挂载对象称为私有挂载，不可绑定挂载与私有挂载相似，但是不允许执行绑定挂载，即创建mount namespace时这块文件对象不可被复制。<br />

![_config.yml]({{ site.baseurl }}/images/namespace.png)

### Network namespace：

主要提供了关于网络资源的隔离，包括网络设备、IPv4和IPv6协议栈、IP路由表、防火墙、/proc/net目录、/sys/class/net目录、端口（socket）等等。一个物理的网络设备最多存在于一个network namespace中，但可以通过veth pair在不同的network namespace间创建通道，以达到通信的目的。一般情况下，物理网络设备都分配在最初的root namespace中，但如果有多块物理网卡就可以把其中的一块或者多块分配给新创建的network namespace。但值得注意的是，当拥有物理网卡的namespace被释放时，物理网卡会返回到root namespace中而不是返回被释放的namespace的父进程所在namespace中。network namespace并不是真正的网络隔离，而是把网络独立出来，给外部用户一种透明的感觉，仿佛在跟另外一个实体通信。为此，容器的经典做法就是创建一个veth pair，一端放在新的namespace中，一端放在原先的namespace中连接物理网络设备，再通过网桥把别的设备连接进来或者进行路由的转发。<br />
在建立起veth pair前，新旧namespace是通过pipe通信的，以Docker Daemon在启动容器dockerinit的过程为例，Daemon在宿主机上负责创建这个veth pair，通过netlink调用，把一端绑定到docker0网桥上，一端连进新建的network namespace中。建立的过程中，Daemon和dockerinit通过pipe通信，当Daemon完成创建veth pair之前，dockerinit在管道的另一端循环等待，直到Daemon传来关于veth设备的信息，并关闭管道。<br />

### User namespace：

主要隔离了安全相关的标识符和属性，包括用户ID、用户组ID、root目录、key（密钥）以及特殊权限。也就是说，一个普通用户的进程通过clone（）创建的新的user namespace中可以拥有不同的用户和用户组，而这个普通用户的进程对于新创建的namespace就属于拥有权限的超级用户。
