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
它不实现实际的 virtio 数据路径，而是充当 vhost-user 协议中的主设备，将此处理卸载到 DPDK 进程中的 vhost-user 库。

![virtio device model](path/to/your/image.png)
virtio 内存区域最初由客户机分配。

相应的 virtio 驱动程序通过 virtio 规范中定义的 PCI BAR 配置接口正常与 virtio 设备交互。

virtio-device-model（QEMU 内部）使用 vhost-user 协议来配置 vhost-user 库，以及设置 irqfd 和 ioeventfd 文件描述符。

客户机分配的 virtio 内存区域由 vhost-user 库（即 DPDK 应用程序）映射（使用 mmap 系统调用）。

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

