>1.DPDK优缺点
>优点：
-  kernel bypass：通过UIO绕过传统内核协议栈直接到用户态，省略了内核态不必要的拷贝
- Hugepage：大页技术在使用TLB时相比传统4kb页大大减少了表项目数，进而大大提高了TLB的命中率
- CPU Affinity：CPU亲和性和CPU优化参数isolcpus设置CPU独占能大大减少多核线程的切换，提升效率
- Cache：通过按核分配独有的数据结构和网卡队列（多队列）避免多核同时访问共享资源，并且数据结构进行cache line对齐方式解决缓存一致性的问题。
- DDIO：相比传统的DMA直接写内存的ringbuffer,DDIO技术还会提前预取内存数据到三级缓存上，有效的减少了cache写回和缓存加载内存的时间
- NUMA：NUMA结构提高了总线等资源的数量，每个socket上都是独自使用，减缓了总线等使用的竞争进而提升效率（NUMA结构要避免跨NUMA访问内存）
>缺点：
- 协议栈上移，增加了开发成本
- 低负荷服务器不实用，会造成cpu空转
>网络收发包的三种方式
- 内核收发包
- dpdk：协议栈上移到用户态（kernel bypass）
- RDMA：协议栈下沉到硬件（kernel bypass）

>2.EAL初始化做了哪些事？
- rte_eal_cpu_init()：检测核心（lcore）哪些可用
- rte_parse_arge()：解析用户的命令行参数
- rte_mp_channel_init()：创建主从进程通信线程
- 网卡设备以及设备驱动初始化
- 内存管理整体构建
- 在所有slave lcore上启动线程

>3.IOVA的模式
- IOVA：硬件使用的地址有两种模式，一种为PA，一种为VA
- PA：物理地址与虚拟地址直接1:1转换，转换后的虚拟地址是非连续的
```
需要root权限（DPDK需要维护虚拟地址和物理地址IOVA的对应关系）；
需要提前保留内存;
igb_uio/uio_pci_generic仅支持PA模式
```
- VA：物理地址重新映射到虚拟地址（将实际的物理地址通过IOMMU重新映射成连续的虚拟地址），映射之后的虚拟地址是连续的
```
需要硬件或设备平台支持IOMMU
Vfio两者模式都支持
```

>3.中断机制


>4.dpdk网卡驱动初始化
- 总线注册：rte_bus封装总线类型（dpaa/fslmc/ifpga/PCI/vdev/vmbus），总线类型注册通过宏RTE_REGISTER_BUS挂载到rte_bus_list列表上.
- 驱动注册：igb_uio/uio_pci_generic/vfio-pci，以上是dpdk提供的三种PMD网卡驱动，基于内核UIO实例化，与总线注册相同，都是在main函数之前通过宏 RTE_PMD_REGISTER_PCI 挂载到pci_driver_list链表上.
- 设备扫描：rte_bus_scan()遍历rte_bus_list，最终回调pci_scan_one()获取pcie设备地址、驱动类型等相关信息，最终将设备按pcie地址升序（由小到大）排列插入链表rte_pci_bus.device_list。22.07版本与19.11相比对网卡黑白名单检测流程做了改动
- 驱动匹配：遍历rte_pci_bus.device_list，为每个设备探测驱动；原理是每个设备遍历驱动遍历pci_driver_list，每个驱动调用一次rte_pci_probe_one_driver->rte_pci_match接口内部根据id_table（设备厂商设置）进行设备与驱动的匹配，找到驱动后将驱动保存到设备pci结构中进行关联。
- 驱动初始化：驱动匹配成功后，进行一些校验后会调用dr->probe()，probe首先分配一个rte_eth_dev结构的一个实例，接着回调dev_init()对设备进行初始化.

>5.dpdk网卡收发包流程
- 创建mbuf_pool：rte_pktmbuf_pool_create
- 配置队列的个数以及接口的配置信息：rte_eth_dev_configure
- 使用之前创建的mbuf_pool初始化每个接收队列：rte_eth_queue_setup
- 初始化发送队列：rte_eth_tx_queue_setup
- 启动设备：rte_eth_dev_start
![[28541347_1526221162OwFd.jpg]]
>dpdk内存管理
- mempool
- mbuf
- ring

2.ACL
3.RSS

### Hyperscan
>1.Hyperscan匹配模式
```
块模式：匹配完整的数据
流模式：跨包匹配，Hyperscan可以保存当前数据匹配的状态，并以其作为接收到新数据时的初始匹配状态.
```