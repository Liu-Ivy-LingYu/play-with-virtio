[Introduction to Virtio Networking and vhost-net](https://www.redhat.com/en/blog/introduction-virtio-networking-and-vhost-net)
## Virtio spec: 
定义了如何在 Front-end 和 Back-end 之间构建控制路径和数据路径的标准，例如：数据路径规定采用环形队列缓冲区布局。
## vhost protocol：
QEMU 根据 VirtIO Spec 实现了控制路径，而数据路径则可以 Bypass QEMU，使用 vhost-net（in kernel）、vhost-user(in userspace) 等实现。
也可以offlaod到smartNIC。

[Deep Dive into Virtio Networking and vhost-net](https://www.redhat.com/en/blog/deep-dive-virtio-networking-and-vhost-net)

*Tun/tap：虚拟的点到点网络设备，用户空间可以用来交换数据包
Tap: 二层
tun：三层 /dev/net/tun*

![Virtio net device emulated by qemu](path/to/your/image.png)

流程图总结了 2 个关键流程：
1. virtio-net Device 初始化流程：包括 Device discovery 和 Device configuration。对于 virtio-net Driver 而言，virtio-net Device 就是一个 PCI 设备，遵守标准的 PCI 设备初始化流程，并且 Driver 和 Device 使用 PCI 协议进行通信。
2. virtio-net Driver 发包流程：
    1. 当 virtio-net Driver 发送数据包时，将数据包放入 Send Queue，将数据包的访问地址放入 Available Ring，触发一次 Available Buffer Notification 到 KVM，然后 VM Exit，由 QEMU 接管 CPU，执行 I/O 控制逻辑，将此数据包传递到 virtio-net Device，然后 virtio-net Device 降数据包通过 Tap 设备传入 Kernel Network Stack；
    2. 完成后，QEMU 将 New Buffer Head Index 放入到 Used Ring 中，同时 virtio-net Device 发出一个 vIRQ 中断注入到 KVM，然后 KVM 发出一个 Used Buffer Notification 到 virtio-net Driver，此时 virtio-net Driver 就会从 Used Ring 中取出 Buffer Head Index；
至此一次发送操作就完成了。

![vhost-net framework](path/to/your/image.png)
## Vhost-net: 由kernel提供后端（
- 更低的延迟（latency）：比普通 virtio-net 低 10%。
- 更高的吞吐量（throughput）：比普通 virtio-net 高 8 倍，达到 7～8 Gigabits/Sec）
vhost-net 的核心思想就是将 Data Path Bypass QEMU，定义了一种新的 Data Path 传输方式，使得 **GuestOS virtio-net Driver** 和 **HostOS Kernel vhost-net** 可以直接通信。

- Control Plane：virtio-net Driver 和 virtio-net Device 之间使用了 PCI 传输协议，**virtio-net Device 和 vhost-net 之间使用 ioctl() 接口**，如下图中蓝线部分。
- Data Plane：**virtio-net Driver 和 vhost-net 直通**，如下图中虚线部分。而 vhost-net 和 Kernel Network Stack 之间使用了 **Tap** 虚拟网卡设备连接，如下图中红实线部分。
- Notification：作为 virtio-net Driver、vhost-net 和 KVM 之间交互方式，用于实现 “vCPU 硬中断“ 的功能。

vhost-net 的本质是一个 HostOS Kernel Module，实现了 vhost-net Protocol 标准，当 Kernel 加载 vhost-net 后，会暴露一个设备接口文件 **/dev/vhost-net**。

![vhost-net process](path/to/your/image.png)
当 QEMU 在 vhost-net 模式下启动时，QEMU（virtio-net Device）首先会 open() 该文件，并通过 ioctl() 与 vhost-net 进行交互，继而完成 vhost-net 实例的 Setup（初始化）。初始化的核心是**建立 virtio-net Driver 和 vhost-net 之间的 Data Path 以及 Notification**，包括：
- **Hypervisor Memory Layout（虚拟化内存布局）**：用于 Data Path 传输，让 vhost-net 可以在 KVM 的内存空间中找到特定 VM 的 Virtqueues 和 Vrings 的地址空间。
- **ioeventfd 和 irqfd（文件描述符）**：用于 Notification 传输，让 vhost-net 和 KVM 之间可以在 Kernel space 中完成 Notification 的直接交互，不再需要通过 User space 中的 QEMU。

同时，在 vhost-net 实例初始化的过程中，vhost-net 会创建一个 Kernel Thread（**vhost worker thread**），名为 vhost-$pid（pid 是 QEMU 进程 PID），QEMU 进程就和 vhost-net 实例以此建立了联系。vhost worker thread 用于处理 I/O Event（异步 I/O 模式），包括：
- 使用 ioeventfd 轮询 Tap 事件和 Notify 事件，并转发数据。
- 使用 irqfd 允许 vhost-net/KVM 通过对其进行写入来将 vCPU 的 vIRQ 注入到 virtio-net Driver。

综上，在 vhost-net 实例初始化完成之后，virtio-net Device 将不再负责数据包的处理（对 Virtqueues 和 Vrings 的读/写操作），取而代之的是 vhost-net。vhost-net 可以直接访问 VM 的 Virtqueues 和 Vrings 内存空间，也可以通过 KVM 与 virtio-net Driver 进行 Notification 交互。

![vhost-user arch](path/to/your/image.png)
## vhost-user（由 DPDK 实现的用户态后端）
vhost-user 是一个基于 DPDK vhost-user Library 开发的后端，运行在 User space，应用了 DPDK 所提供的 CPU 亲和性，大页内存，轮询模式驱动等数据面转发加速技术。

区别于 vhost-net Protocol，vhost-user Library 实现了一种新的 vhost-user Protocol，两者间最大的区别体现在通信信道的实现方式上。
 - vhost-net：使用 /dev/vhost-net 字符设备和 ioctl() 实现了 User Process（QEMU）和 Kernel（vhost-net）之间的 Control Plane 通信信道。
 - vhost-user：使用 **Unix Socket** 实现了 User Process（QEMU 和 DPDK App）之间的 Control Plane 通信信道。

QEMU 通过 Unix Socket 来完成对 vhost-user 的 Data Plane 配置，包括：
 - 特性协商配置：QEMU（virtio-net Device）与 vhost-user 协商两者间功能特性的交集，确认哪些功能特性是有效的。
 - 内存区域配置：**vhost-user 使用 mmap() 来映射分配给 QEMU 的内存区域**，建立两者间的直接数据传输通道。
 - Virtqueues/Vrings 配置：QEMU 将 Virtqueues/Vrings 的数量和访问地址发送给 vhost-user，以便 vhost-user 访问。
 - Notification 配置：vhost-user 仍然使用 ioeventfd 和 irqfd 来完成和 KVM 之间的通知交互。
