[How vhost-user came into being: Virtio networking and DPDK](https://www.redhat.com/en/blog/how-vhost-user-came-being-virtio-networking-and-dpdk)

为了解决性能问题，引入 vhost-user/virtio-pmd 架构。
## DPDK 概述

## DPDK OVS概述
Open vSwitch 通常使用内核空间数据路径转发数据包。这意味着 **OVS 内核模块包含一个简单的流表**，用于转发收到的数据包。不过，一小部分数据包（我们称之为异常数据包（Openflow 流中的第一个数据包））与内核空间中的现有条目不匹配，并被发送到用户空间 OVS 守护进程 (ovs-vswitchd) 进行处理。然后，守护进程将分析数据包并更新 OVS 内核流表，以便此流上的其他数据包将直接通过 OVS 内核模型转发表。

User space包括一个数据库(ovsdp-server)，OVS daemon(ovs-vswitchd)用于管理和控制switch Kernel space负责数据通路转发。

如果我们集成 OVS 和 DPDK，我们可以利用前面提到的 PMD 驱动程序并将之前的 **OVS 内核模块转发表**移动到用户空间。

![OVS DPDK](path/to/your/image.png)

## vhot-user/virtio-pmd
![vhost user/ virtio pmd](path/to/your/image.png)

通过允许主机用户空间通过共享内存绕过内核直接访问物理 NIC，并在客户机用户空间上使用 virtio-pmd 也绕过内核，整体性能可以提高 2 到 4 倍。

[A journey to the vhost-users realm](https://www.redhat.com/en/blog/journey-vhost-users-realm)

## Vhost-user library in DPDK

## QEMU virtio device model
模拟了一个virtio设备，该设备显示在客户机的特定PCI端口中，客户机可对该端口进行无缝探测和配置，此外，它还将ioeventfd映射到模拟设备的内存映射I / O空间，并将irqfd映射到它的全局系统中断（Global System Interrupt，GSI）。

它不实现实际的 virtio 数据路径，而是充当 vhost-user 协议中的主设备，将此处理卸载到 DPDK 进程中的 vhost-user 库。

![virtio device model](path/to/your/image.png)
virtio 内存区域最初由客户机分配。由 vhost-user 库（即 DPDK 应用程序）映射（使用 mmap 系统调用）。

相应的 virtio 驱动程序通过 virtio 规范中定义的 PCI BAR 配置接口正常与 virtio 设备交互。

virtio-device-model（QEMU 内部）使用 vhost-user 协议来配置 vhost-user 库，以及设置 irqfd 和 ioeventfd 文件描述符。

结果是 DPDK 应用程序可以直接从客户机内存读取和写入数据包，并使用 irqfd 和 ioeventfd 机制直接通知客户机。

## 客户机中的用户空间网络
为了能够直接在设备上运行用户空间网络应用程序，我们需要三个组件：

**VFIO**：VFIO 是一个用户空间驱动程序开发框架，允许用户空间应用程序直接与设备交互（绕过内核）

**Virtio-pmd** 驱动程序：是一个基于轮询模式驱动程序抽象构建的 DPDK 驱动程序，可实现 virtio 协议

**IOMMU** 驱动程序：IOMMU 驱动程序用于管理虚拟 IOMMU（I/O 内存管理单元），这是一种模拟设备，可为支持 DMA 的设备执行 I/O 地址映射。

### VFIO 基本上是一个用于构建用户空间驱动程序的框架，它提供：

将设备的配置和 I/O 内存区域映射到用户内存

基于 IOMMU 组的 DMA 和中断重新映射和隔离。我们将在本文的后面更深入地介绍 IOMMU 是什么以及它是如何工作的。现在，假设它允许创建映射到物理内存的虚拟 I/O 内存空间（类似于普通 MMU 映射非 IO 虚拟内存的方式），因此当设备想要 DMA 到虚拟 I/O 地址时，IOMMU 将重新映射该地址并可能应用隔离和其他安全策略。

基于 Eventfd 和 irqfd的信号机制，用于支持来自和发往用户空间应用程序的事件和中断。

### virtio 轮询模式驱动程序 
virtio-pmd是使用 PMD API 的众多驱动程序之一，它为使用 DPDK 编写的应用程序提供对 virtio 设备的快速和无锁访问，提供使用 virtio 的 virtqueues 进行数据包接收和传输的基本功能。

除了任何 PMD 都具有的所有功能外，virtio-pmd 驱动程序实现还支持：

接收时每个数据包的灵活可合并缓冲区和发送时每个数据包的分散缓冲区

多播和混杂模式

MAC/vlan 过滤

### vIOMMU and vhost-user integration
当 QEMU 中模拟的设备尝试 DMA 到客户机的 virtio I/O 空间时，它将使用 vIOMMU TLB 查找页面映射并执行安全的 DMA 访问。问题是，如果实际的 DMA 被卸载到外部进程（例如使用 vhost-user 库的 DPDK 应用程序），会发生什么情况？

当 vhost-user 库尝试直接访问共享内存时，它必须将所有地址（I/O 虚拟地址）转换为自己的内存。它通过**设备 TLB API 向 QEMU 的 vIOMMU 请求转换**来实现这一点。Vhost-user 库（以及 vhost-kernel 驱动程序）使用 **PCIe 的地址转换服务标准消息集**，使用**在配置 IOMMU 支持时创建的辅助通信通道（另一个 unix 套接字）**向 QEMU 请求页面转换。

总体而言，需要进行 3 次地址转换：

QEMU 的 vIOMMU 将 IOVA（I/O 虚拟地址）转换为 GPA（客户机物理地址）。

Qemu 的内存管理将 GPA（客户机物理地址）转换为 HVA（qemu 进程地址空间内的主机虚拟地址）。

Vhost-user 库将（QEMU 的）HVA 转换为其自己的 HVA。这通常很简单，只需在 vhost-user 库映射 QEMU 内存时将 QEMU 的 HVA 添加到 mmap(2) 返回的地址即可。

显然，所有这些转换都可能对性能产生重要影响，尤其是在使用动态映射的情况下。但是，静态大页面分配（这正是 DPDK 所做的）可以最大限度地减少这种性能损失。

![vhost-user virtio-net pmd arch](path/to/your/image.png)

关于这个相当复杂的图表，有几点需要提及：

客户机的物理内存空间是客户机认为是物理的内存，但显然，它位于 QEMU 的进程（主机）虚拟地址内。分配 virtqueue 内存区域时，它最终会位于客户机的物理内存空间中的某个位置。

当将 I/O 虚拟地址分配给包含 virtqueues 的内存范围时，会将与其关联的客户机物理地址 (GPA) 的条目添加到 vIOMMU 的 TLB 表中。

另一方面，QEMU 的内存管理系统知道客户机物理内存空间在其自身内存空间中的位置。因此，它能够将客户机物理地址转换为主机（QEMU）的虚拟地址。

当 vhost-user 库尝试访问它无法转换的 IOVA 时，它会通过辅助 unix 套接字发送 IOTLB 未命中消息。

IOTLB API 接收请求并查找地址，首先将 IOVA 转换为 GPA，最后将 GPA 转换为 HVA。然后，它通过主 unix 套接字将转换后的地址发送回 vhost-user 库。

最后，vhost-user 库必须进行最后一次转换。由于它将 qemu 的内存映射到自己的内存中，因此它必须将 qemu 的 HVA 转换为自己的 HVA 并访问共享内存。

![put everything together arch](path/to/your/image.png)
将此图与上图进行比较，我们添加了使用硬件 IOMMU、VFIO 和特定于供应商的 PMD 驱动程序将主机的 OVS-DPDK 应用程序连接到物理 NIC 所需的组件。

![example flow](path/to/your/image.png)

### control plane
1. 当主机 (OvS) 中的 DPDK 应用程序启动时，它会创建一个套接字（在服务器模式下），用于与 qemu 进行与 virtio 相关的协商。

2. 当 qemu 启动时，它会连接到主套接字，如果 vhost-user 提供了功能 VHOST_USER_PROTOCOL_F_SLAVE_REQ，它会创建第二个套接字并将其传递给 vhost-user，以便它连接并发送 iotlb 同步消息。

    - 当 QEMU <-> vhost-library 协商结束时，它们之间共享两个套接字。一个用于 virtio 配置，另一个用于 iotlb 消息交换。

3. 客户机启动，vfio 驱动程序绑定到 PCI 设备。它创建对 iommu 组的访问（取决于硬件拓扑）

4. 当 dpdk 应用程序在客户机中启动时，它会执行以下初始化步骤：

    - 初始化 PCI-vfio 设备。此外，vfio 驱动程序将 PCI 配置空间映射到用户内存。

    - 分配 virtqueues。

    - 使用 vfio，执行 virtqueue 内存空间的 DMA 映射请求，通过 IOMMU 内核驱动程序将 dma 映射添加到 vIOMMU 设备。

    - 然后，进行 virtio 功能协商。在这种情况下，用作 virtqueue 基地址的地址是 IOVA（在 I/O 虚拟地址空间中）。设置 eventfd 和 irqfd 的映射，以便中断和通知可以直接在客户机和 vhost-user 库之间路由，而无需 QEMU 的干预。

    - 最后，dpdk 应用程序为网络缓冲区分配一大块连续的内存。该内存区域的映射也通过 VFIO 和 IOMMU 驱动程序添加到 vIOMMU。

此时配置已完成，数据平面（虚拟队列和通知机制）已可供使用。

### data plane
为了传输数据包，需要执行以下步骤：

1. 客户机中的 DPDK 应用程序命令 virtio-pmd 发送数据包。它写入缓冲区并将其对应的描述符添加到可用描述符环中。

2. 主机中的 vhost-user PMD 正在轮询 virtqueue，因此它会立即检测到新的描述符可用并开始处理它们。

3. 对于每个描述符，vhost-user PMD 都会映射其缓冲区（即：将其 IOVA 转换为 HVA）。在极少数情况下，缓冲区内存是尚未映射到 vhost-user 的 IOTLB 中的页面，将向 QEMU 发送请求。但是，客户机中的 DPDK 应用程序分配静态大页面这一事实将 IOTLB 对 QEMU 的请求保持在最低限度。

4. vhost-user PMD 将缓冲区复制到 mbuf（DPDK 应用程序使用的消息缓冲区）中。

5. 描述符被添加到已用描述符环中。客户机中的 DPDK 应用程序会立即检测到这些描述符，同时轮询虚拟队列。

6. 主机中的 DPDK 应用程序会处理 mbuf。

[Virtqueues and virtio ring: How the data travels](https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels)