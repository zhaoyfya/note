### 序言
&emsp;&emsp;本文主要以MTU（Maxitum Transmission Unit）为核心展开介绍一些相关概念。围绕这些概念详细说明NISG在使用时该如何进行MTU的设置，以及网卡offload的说明更好的理解offload应该如何使用。

### MTU与其相关概念
#### MTU是什么
&emsp;&emsp;MTU（Maxitum Transmission Unit）即最大传输单元，其作用于链路层，通俗的讲就是在数据帧在通过网卡时的最大值，同时该值也是上层一次传输数据的最大值。

#### MTU的由来
&emsp;&emsp;标准的以太网帧（64字节~1518字节），去掉以太网头14个字节，再去掉尾部的校验和4个字节，留给<font color="#dd0000">上层协议</font>也就是<font color="#dd0000">1500</font>（1518-14-4）字节，这个就是<font color="#dd0000">MTU</font>的由来。而当标准的以太网帧长度小于64字节时（Data小于46字节），会以0填充保证以太网帧满足的其下限标准。
![](Pasted%20image%2020230810163910.png)
&emsp;&emsp;标准以太网帧的下限64字节与上限1518字节是因为早期以太网工作方式下根据效率而计算出来的，感兴趣的可以自行搜索。

#### Jumbo Frame
&emsp;&emsp;超过以太网标准长度1518字节的帧称为巨型帧（jumbo frame），而巨型帧的出现是由于现如今的网络带宽早已不是早期的10Mbps，为适应目前的带宽并提高效率，一些设备厂商规定了巨型帧（9000字节）。
如果一条物理链路的两端MTU不一致，则会发生什么情况？这就需要说明一下MRU。

#### MRU与MTU
&emsp;&emsp;MRU（Maximum Receive Unit）即最大接收单元，目前没有权威的标准定义，一台主机或路由器的MTU与MRU可以不一致。在网络链路上有收和发两个方向，MTU是出方向相关的参数，MRU就是收方向相关的参数，一般情况下MTU=MRU。

&emsp;&emsp;假设目前一条链路的两端MTU不一致，一侧是9000，另一侧是1500，当1500侧发送数据的时候，9000侧接收时没有问题的。但是反过来9000发送时，1500侧则会默默的丢该。原因就是因为9000 > 1500侧MRU了。

#### MMS与MTU
&emsp;&emsp;MSS（Maximum Segment Size）是指TCP协议所允许的从对方收到的最大报文长度，即TCP数据包每次能够传输的最大数据分段，只包含TCP Payload，不包含TCP Header和TCP Option。

&emsp;&emsp;MSS是TCP用来限制application层最大的发送字节数。为了达到最佳的传输效能，TCP协议在建立连接的时候通常要协商双方的MSS值，这个值TCP协议在实现的时候往往根据MTU值来计算（需要减去IP包头20字节和TCP包头20字节），所以通常MSS为1460=1500(MTU)- 20(IP Header) -20 (TCP Header)。

&emsp;&emsp;两个终端直连的情况下双方仅需要根据自身的MTU计算出MSS进行协商即可，若两端的链路上存在转发节点，则每到一个节点会根据该节点上的MTU计算出MSS修改报文中的MSS后在进行转发，其本质上也是获取链路上最小的MTU。
![](Pasted%20image%2020230810164725.png)

#### PMTU与MTU
&emsp;&emsp;PMTU（path maximum transmission unit），Path MTU就是指传输路径的MTU，无需分片就能穿过某路径的数据包最大长度。

&emsp;&emsp;Path MTU Discovery即路径MTU发现，就是在发送端到接收端的传输路径上的网元MTU设置不一致时，决定该路径可用MTU的，其实是整条路径上的最小MTU值。以PMTU作为IP包长发送数据，既高效又能避免分片。
![](Pasted%20image%2020230810165202.png)
>&emsp;&emsp;Path MTU Discovery其原理本质上是在发送TCP/UDP数据包时，给每个数据包打上DF=1的标志（设置报文不允许被分片），然后先获取本地出接口MTU进行首次发送。当数据包被发送出去之后会有两种情况：<br />
&emsp;&emsp;1.发送端的MTU就是链路上最小MTU，这种情况下能保证每个数据包正常通过链路上的各个节点的到达接收端，这也是为什么提倡设备MTU设置为默认值1500的原因。<br />
&emsp;&emsp;2.在链路上存在节点设备的MTU小于发送端的MTU，此时又因为数据包设置了DF=1不允许进行分片，所以该数据包就会被中间节点所丢弃。丢弃后该节点会给发送端回一个ICMP的Destination Unreachable，Fragment Needed（目标不可达，需要分片）的消息。并且这个ICMP差错报文会包含该节点的MTU值，当发送端接收后，就会根据得到的MTU值进行调整后重新再发送一个数据包，注意再次发送的这个数据包也会设置DF=1不允许分片，因为发送端不确定一条链路上具体有多少个节点，每当数据包遇到一个节点的MTU小于当前数据包的长度时就会触发这个机制。

![](Pasted%20image%2020230810165212.png)
&emsp;&emsp;那PMTU如何开启呢？<br />
&emsp;&emsp;系统通过内核参数/proc/sys/net/ipv4/ip_no_pmtu_disc来判断是否禁用PMTU，一般情况下系统默认开启Path MTU Discovery功能,即ip_no_pmtu_disc=0。
>注：当链路中存在防火墙等安全设备时，需要给ICMP添加放行策略，否则会把ICMP过滤掉导致PMTU功能失效。

### 网卡offload功能
>&emsp;&emsp;MTU的本质作用还是在于规范数据从网卡发出时的大小，若发送数据大于出接口MTU就需要对数据包进行进行切割（IP分片/TCP分段）<br />
> IP分片：IP是否分片取决于物理硬件，即**MTU**的限制，IP分片是在网络层进行处理<br />
> TCP分段：TCP分段则是通过三次握手协商出**MMS**后在传输层对数据进行处理

#### 既然有了IP分片为什么还需要TCP分段呢?
&emsp;&emsp;假设有一份数据较大，且在TCP层不分段，如果这份数据在发送的过程中出现丢包现象，TCP会发生重传，那么重传的就是这一大份数据（虽然IP层会把数据切分为MTU长度的N多个小包，但是TCP重传的单位却是那一大份数据）<br />
&emsp;&emsp;如果TCP把这份数据，分段为N个小于等于MSS长度的数据包，到了IP层后加上IP头和TCP头，还是小于MTU，那么IP层也不会再进行分包。此时在传输路上发生了丢包，那么TCP重传的时候也只是重传那一小部分的MSS段。效率会比TCP不分段时更高。

#### 如果TCP分段的话，IP一定不分片吗?
&emsp;&emsp;这就需要用到上面提到的PMTU机制，TCP发送数据时会进行PMTU进行链路探测后，选取链路节点上最小的MTU，在传输层分段后网络层不进行分片也能保证每个数据包满足链路上各个节点的MTU进行转发，但是如果PMTU功能被禁止或探测的ICMP包被过滤掉导致探测失败，即使传输层TCP对数据进行分段处理后，也可能会因为链路上某些节点MTU过小而导致转发过程中被IP分片。

#### 网卡offload
&emsp;&emsp;无论是IP分片或是TCP分段都会占用CPU，消耗CPU性能，为了降低系统 CPU 消耗的同时，提高处理的性能。将本来该操作系统进行的一些数据包处理（如分片、重组等）放到网卡硬件中去做。目前大部分网卡都支持offload功能，主要分发送（GSO/TSO）和接收（GRO/LRO）。
&emsp;&emsp;TSO/LRO是将分片和重组功能卸载到网卡，这样减轻了CPU的负荷，但需要硬件的支持。而GSO和GRO是软件上的实现，并不需要硬件支持。相对于硬件的TSO/LRO仅支持TCP而言，软件上的实现更具通用性。但无论是软件实现的GSO/GRO或是硬件实现的TSO/LRO，本质都是将分片工作下沉后让协议栈的处理更轻量化从而减轻CPU的负载提升性能，也能有效的减少内核协议栈处理包的次数，所以采用offlad功能将大大提高网络安全设备的吞吐量和性能。
##### GSO与TSO
>&emsp;&emsp;TSO（TCP Segmentation Offload）是一种利用网卡对大数据包进行分片。

![](Pasted%20image%2020230810170305.png)
>&emsp;&emsp;GSO（Generic Segmentation Offload）是延缓分片技术。首先查询网卡是否支持TSO功能，如果硬件支持TSO则使用网卡的硬件分片能力执行分片；如果网卡不支持 TSO 功能，则将分片的执行，延缓到了将数据推送到网卡的前一刻执行。

![](visio1.png)

##### GRO与LRO
>&emsp;&emsp;LRO（Large Receive Offload）是将网卡接收到的多个数据包合并成一个大的数据包，然后再传递给网络协议栈处理的技术。

![](Pasted%20image%2020230810171709.png)
>&emsp;&emsp;GRO （Generic Receive Offload）是 LRO 的软件实现，只是GRO 的合并条件更加的严格和灵活。

![](visio2.png)

### MTU的设置
>&emsp;&emsp;在实际的网络环境中可能存在不同厂商的设备，而不同厂商或同一厂商的不同型号设备对MTU的定义和MTU分片机制不尽相同，常出现MTU引起的网络问题。

#### 内核源码分析MTU
![](Pasted%20image%2020230810171957.png)
>&emsp;&emsp;如上图所示，无论是转发或者是本地发送，本质上内核最终调用的都是ip_output()，与MTU相关的重点是ip_finish_output()，该接口根据当前skb获取出接口的MTU，判断是否开启PMTU，若开启则会发送ICMP进行探测，然后修改成PMTU。

#### 测试验证MTU
##### 验证1
![](Pasted%20image%2020230810172043.png)
![](verify1.1.png)
>&emsp;&emsp;上图中上侧是dev1抓包情况，下侧是dev2抓包情况。从抓包上看dev1侧发出的包长8028，正好满足dev1侧的mtu。而dev2侧未抓到任何包，说明1500的mtu无法接收8028的包。
![](verify1.2.png)
![](verify1.3.png)
>&emsp;&emsp;ifconfig eth2查看dev2收包情况可以发现RX errors和frame相同，ethtool -S eth2观察只有rx_long_length_errors同步增长，该错误统计大概意思是接收帧大于接收帧长。由此可见当接收数据长度远大于mru时是无法正常接收数据帧。
##### 验证2
![](Pasted%20image%2020230810172238.png)
![](verify2.png)
>&emsp;&emsp;从抓包情况看resquest是按照dev1侧的mtu=1500分片。replay也是按照dev2侧的mtu=1500进行分片。双方都正常接收了对方发来的数据。
>&emsp;&emsp;但这里有一个问题，为什么dev2侧1000的mtu却能接收到1500字节的数据包呢？这个问题与验证4一起放到最后进行说明。
##### 验证3
![](Pasted%20image%2020230810172333.png)
![](verify3.png)
>&emsp;&emsp;从抓包情况看resquest是按照dev1侧的mtu=1500分片。而replay是按照dev2侧的mtu=1000进行分片。双方都正常接收了对方发来的数据。
##### 验证4
![](Pasted%20image%2020230810172410.png)
![](verify4.png)
>&emsp;&emsp;当我们将dev1侧mtu修改为800时，从抓包情况来看request是按照dev1侧的mtu=800进行分片，replay是按照dev2侧的mtu=1000进行分片。双方都正常接收对方发来的数据包，同验证3一样dev1侧800的mtu接收了996字节的数据。

&emsp;&emsp;首先通过验证1/2/3/4可以看出在发包时是以本端物理接口mtu为标准进行分片的，这也印证了内核是通过查询路由表获取下一跳接口的mtu来确定是否需要分片。但是验证1和验证3与4的结论却相悖，为什么验证3的接收帧长1500 > mtu1000 和验证4的接收帧长996 > mtu800都能正常接收，而验证1的8000的接收帧长却无法被1500的mtu接收呢？

&emsp;&emsp;经过反复验证，最终得出结论是：内核在收包时冥冥之中有一个默认值，就是当你mtu小于1500时，你可以接收小于等于1500字节的数据帧，即使数据帧长大于你当前的mtu。

>这种现象的原因可能有两种：<br />
&emsp;&emsp;1.内核收包时ringbuffer存在一个默认的最小值是1500，它保证你最少能接收1500字节的数据，这个时候mtu就不管用了。<br />
&emsp;&emsp;2.因为标准以太网的特性，在网卡收包时会对数据帧进行CRC校验，当收到大于mtu且小于1500的数据帧时以太网能正常校验（即验证3和4的情况），但数据帧大于1500时以太网无法正常校验，且对端mtu又不能接收。<br />
&emsp;&emsp;对于这两种猜测我认为后者比较符合，因为内核ringbuffer的大小设置里并没有一个比较接近1500的值，而以太网帧用明确1500的默认值。

注：验证过程中物理接口的offload全部关闭，分片都是由内核完成。
### 总结
1. MTU是最大传输单元，作用于链路层,MRU与MTU对应，但没有权威明确标准定义。<br />
2. MMS是TCP根据MTU进行计算得出，三次握手协商后用于TCP分段。<br />
3. PMTU是为了保证链路MTU稳定而设计的，其原理就是通过icmp进行探测得出链路最小MTU保证数据在整条链路上可以通过各个节点。<br />
4. 网卡的offload功能可以讲内核的IP分片与TCP分段卸载到网卡上从而提升CPU效率。<br />
5. 对于NISG设备MTU的选择，如无特殊需求保持默认的1500即可，若需要修改mtu应遵守基本原则保证对接的两个设备以太网接口MTU配置一致。