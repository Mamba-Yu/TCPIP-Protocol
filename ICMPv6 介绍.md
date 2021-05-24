# ICMPv6 介绍

## 1 概述

- ICMPv6（Internet Control Message Protocol for the IPv6）是IPv6的基础协议之一，定义在RFC2463中

- 协议类型号，IPv6 Next Header = 58

- 用于向源节点传递报文转发的信息或者错误

- ICMPv6定义的报文被广泛地应用于其它协议中，如邻居发现(Neighbor Discovery)、PathMTU发现机制等

  

- 合并了IPv4中的ICMP(控制报文协议)、IGMP(组成员协议)、ARP(地址解析协议)、RA(路由广播)等协议的功能
- ICMPv6协议控制着IPv6网络中的地址生成、地址解析、路由选择、以及差错控制等关键环节

## 2 报文格式

### 差错报文

- 最高位为 0， ICMPv6 Type=[0,127]
- 用于报告在转发IPv6数据包过程中出现的错误
- 目标不可达（Destination Unreachable） **Type = 1**

> Code = 0 : 没有到达目标的路由
>
> Code = 1 : 与目标的通信被管理策略禁止
>
> Code = 3 : 地址不可达
>
> Code = 4 : 端口不可达

- 数据包超长（Packet Too Big）**Type = 2**  Code = 0

- 超时（Time Exceeded）**Type = 3**

> Code = 0 : 在传输中超越了跳限制 
>
> Code = 1 : 分片重组超时

- 参数问题（Parameter Problem）**Type = 4** 

> Code = 0 : 遇到错误的报头字段
>
> Code = 1 : 遇到无法识别的下一个报头类型
>
> Code = 2 : 遇到无法识别的IPv6选项

### 信息报文

- 最高位为 1，ICMPv6 Type=[128,255]
- 提供诊断功能和附加的主机功能，比如多播侦听发现（MLD）和邻接点发现（ND）
- 回送请求报文（Echo Request）**Type = 128  Code = 0**
- 回送应答报文（Echo Reply）     **Type = 129  Code = 0**
- **多播侦听发现协议（MLD）**

> 多播听众查询       Type = 130  Code = 0
>
> 多播听众报告       Type = 131  Code = 0
>
> 多播听众退出       Type = 132  Code = 0

- **邻居发现协议（ND）**

> 路由器请求        Type = 133  Code = 0
>
> 路由器公告        Type = 134  Code = 0
>
> 邻居请求            Type = 135  Code = 0
>
> 邻居公告            Type = 136  Code = 0
>
> 重定向                Type = 137  Code = 0

## 3 ND介绍

- RFC2461中定义了邻居发现协议，使用ICMPv6报文如何进行地址解析，如何跟踪邻居的状态，怎样进行重复地址检测，主机如何发现路由器以及重定向等

### 地址解析

>  在三层完成地址解析，主要带来以下几个好处：
>
> - 地址解析在三层完成，不同的二层介质可以采用相同的地址解析协议
>
> - 可以使用三层的安全机制（例如IPSec）避免地址解析攻击
> - 使用组播方式发送请求报文，减少了二层网络的性能压力

- **邻居请求（Neighbor Solicitation）        Type = 135  Code = 0**

> Target Address是需要解析的IPv6地址，因此该处不准出现组播地址。
>
> 邻居请求发送者的链路层地址会被放在Options字段中。

- **邻居通告（Neighbor Adivertisment）   Type = 136  Code = 0**

> R标志（Router flag）   : 发送者是否为路由器，如果1则表示是；
>
> S标志（Solicited flag） : 发送邻居通告是否是响应某个邻居请求，如果1则表示是；
>
> O标志（Overide flag） : 邻居通告中的消息是否覆盖已有的条目信息，如果1则表示是；
>
> Traget Address表示所携带的链路层地址对应的IPv6地址。
>
> 被请求的链路层地址被放在Options字段中，其格式仍然采用TLV格式。

### 邻居状态

> **INCOMPLETE \ REACHABLE \ STALE \ DELAY \ PROBE**
>
> 1. A先发送NS，并生成邻居缓存条目，状态为 Incomplete；
>
> 2. 若B回复NA，则 Incomplete->Reachable，否则10s后 Incomplete->Empty，即删除条目；
>
> 3. 经过 ReachableTime（默认30s），条目状态 Reachable->Stale；
>
> 4. 或者在 Reachable状态，收到B的非请求NA，且链路层地址不同，则马上 -> Stale；
>
> 5. 在 Stale 状态若A需要向B发送数据，则 Stale->Delay，同时发送NS请求；
>
> 6. 在 Delay_First_Probe_Time（默认5秒）后，Delay->Probe，其间若有NA应答，则Delay->Reachable；
>
> 7. 在Probe状态，每隔 RetransTimer（默认1秒）发送单播NS，发送 MAX_UNICAST_SOLICIT 个后再等RestransTimer，有应答则->Reachable，否则进入Empty，即删除表项。

### 重复地址检测 (DAD)

- 在接口使用某个地址之前进行的，主要是为了探测是否有其它的节点使用了该地址

- 一个地址在分配给一个接口之后且通过重复地址检测之前称为“tentative地址”

- 基本思想：节点向一个自己将使用的tentative地址所在的组播组发送一个NS（Target域包含这个tentative地址），如果收到某个其他站点回应的NA，证明该地址已被网络上使用，节点将不能使用地址通信。

- NS的源地址是::，即“未指定地址”（unspecified address），因为发送者地址事实上尚未指定
- 两种方式发现重复地址：

> NS接收者如果发现其中的Target域中包含的地址对它而言是一个tentative地址，那么它就知道自己的这个tentative地址是重复的，从而放弃使用这个地址作为接口地址

> NS接收者如果发现其中的Target域中包含的地址对它而言不是一个tentative地址，那么它就会向Target所在的Node-Solicited组播组发送一个NA，该消息中的Target域会包含收到的NS中的Target地址。这样，NS发起者收到这个消息后就会发现它的tentative地址是重复的，从而弃用该地址

### 路由器发现

**路由器通告（Router Advertisement）**  **Type = 134  Code = 0**

>  每台路由器为了让二层网络上的主机和路由器知道自己的存在，定时都会组播发送RA

**路由器请求（Router Solicitation）**         **Type = 133  Code = 0**

> 主机接入网络后希望尽快获取前缀进行通讯，主机可以立刻发送RS，网络上的路由器将回应RA

### 重定向     

- 网关路由器发现报文从其它网关路由器转发更好，它就会发送重定向报文告知报文的发送者，让报文发送者选择另一个网关路由器，Type为137，Code为0
- Target Address是更好的路径下一跳地址
- Destination Address是需要重定向转发的报文的目的地址

### PMTU协议

- IPv6报文仅在源节点进行分片，在目的节点进行组装，在转发的过程中是不进行分片操作的
- 为了所有的报文都能在路径上畅通无阻，那么分片的报文大小不能超过路径上最小的MTU，即路径MTU
- PMTU发现的机制，是通过ICMPv6的Packet Too Big报文来完成的
- 只有数据包超过路径上的最小MTU时，PMTU发现机制才有意义，因为如果报文很小，小于路径上最小的MTU，就不可能产生Packet Too Big报文
- PMTU最小为1280bytes（IPv6要求链路层所支持的MTU最小为1280bytes）
- 最大PMTU由链路层决定，如隧道，可以支持很大的MTU



