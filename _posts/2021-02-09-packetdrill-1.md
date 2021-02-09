---
title: 网络协议栈测试神器 - packetdrill
date: 2021-02-09
categories:
- 网络
tags:
- packetdrill
---

## 简介
`packetdrill`是 Google 开源的一个 测试脚本工具，可以用于测试TCP、UDP、IP网络协议栈，其是由基于时间序的脚本行组成，按时间顺序逐条执行。

它的语言设计十分接近于`tcpdump`和`strace`，包含四种类型的语句：  

* 数据包。使用类似于`tcpdump`的语法，支持TCP、UDP、ICMP数据包，同时也提供了常见TCP选项的配置，包括 SACK、MSS、window scale等。  

* 系统调用。使用类似于`strace`的语法。  

* Shell 命令。通过`&#x60; command &#x60;`进行调用，可以进行系统参数配置或断言验证网络协议栈状态。  

* Python 脚本。通过`%{ command }%`进行调用，可以输出或者断言验证 TCP 状态。  

## 编译、安装

参考官方GitHub [packetdrill](https://github.com/google/packetdrill)  

## 基础使用

通过一个简单的例子，来看看 packetdrill 的使用。  

下面一段脚本是用于测试 TCP 三次握手的过程。  

```
0       socket(..., SOCK_STREAM, IPPROTO_TCP) = 3
+0      setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
+0      bind(3, ..., ...) = 0
+0      listen(3, 1) = 0

+0      < S  0:0(0) win 1000 <mss 1000>
+0      > S. 0:0(0) ack 1 <...>
+0      < .  1:1(0) ack 1 win 1000
```

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/packetdrill_1/tcp_handshake.png){:height="500" width="500"}


脚本一共做了两件事情：  

* 通过系统调用创建并配置 Socket ，绑定端口开始监听。  

`0       socket(..., SOCK_STREAM, IPPROTO_TCP) = 3`  

我们创建了一个 Stream 类型的 Socket，并指定协议为 TCP。   

`0` 表示绝对时间，即从第0秒开始。  
`…`是默认缺省，表示系统自动配置填充。  
`= 3` 表示函数调用结果断言为 3 ，即创建的 Socket 对应的FD。  

剩下的代码就是配置并监听。

> 这里可能会有一个疑问，为什么新建 Socket 对应的 FD 是 3 ？
> 对于一个进程而言，默认会打开 stdin（0） 、 stdout（1） 、stderr（2），那么正常情况下，第一个新打开的 FD 正好是 3。

* 三次握手  

通过三行脚本来发起和校验三次握手的过程。  

```
// 向内核网络协议栈注入SYN包
+0      < S  0:0(0) win 1000 <mss 1000>
// 断言：从协议栈收到SYN+ACK包，且ACK序号为1.
+0      > S. 0:0(0) ack 1 <...>
// 注入返回的ACK包，完成三次握手
+0      < .  1:1(0) ack 1 win 1000
```

`+0` 相对时间，上一条命令执行结束的 0 秒后，开始执行当前命令。  
`<` 往内核网络协议栈中，注入数据包。  
`>` 从内核网络协议栈中，收到数据包，并断言收到的数据包与定义的包是否一致，例如 ack 是否为1。  
`S` SYN报文，对应 TCP 数据包中的 Flag。  
`0:0(0)`开始序号：结束序号（数据包长度）。首包 sequence 是0，SYN数据包没有数据payload，所以长度是0，结束序号 = 开始序号 + payload length = 0。  
`ack 1 win 1000` ACK序号为1，窗口大小为1000。  


## TCP 网络状态  

TCP是可靠的，需要保持连接状态来维持连接，连接状态的变化往往是我们非常关注的部分。packetdrill 支持对连接状态的检查和查看，对应的 TCP 连接信息定义在了 `tcp.h` 文件中，内容如下：  

```

/* Data returned by the TCP_INFO socket option. */
struct _tcp_info {
	__u8	tcpi_state;
	__u8	tcpi_ca_state;
	__u8	tcpi_retransmits;
	__u8	tcpi_probes;
	__u8	tcpi_backoff;
	__u8	tcpi_options;
	__u8	tcpi_snd_wscale:4, tcpi_rcv_wscale:4;
	__u8	tcpi_delivery_rate_app_limited:1;

	__u32	tcpi_rto;
	__u32	tcpi_ato;
	__u32	tcpi_snd_mss;
	__u32	tcpi_rcv_mss;

	__u32	tcpi_unacked;
	__u32	tcpi_sacked;
	__u32	tcpi_lost;
	__u32	tcpi_retrans;
	__u32	tcpi_fackets;

	/* Times. */
	__u32	tcpi_last_data_sent;
	__u32	tcpi_last_ack_sent;     /* Not remembered, sorry. */
	__u32	tcpi_last_data_recv;
	__u32	tcpi_last_ack_recv;

	/* Metrics. */
	__u32	tcpi_pmtu;
	__u32	tcpi_rcv_ssthresh;
	__u32	tcpi_rtt;
	__u32	tcpi_rttvar;
	__u32	tcpi_snd_ssthresh;
	__u32	tcpi_snd_cwnd;
	__u32	tcpi_advmss;
	__u32	tcpi_reordering;

	__u32	tcpi_rcv_rtt;
	__u32	tcpi_rcv_space;

	__u32	tcpi_total_retrans;

	__u64	tcpi_pacing_rate;
	__u64	tcpi_max_pacing_rate;
	__u64	tcpi_bytes_acked;    /* RFC4898 tcpEStatsAppHCThruOctetsAcked */
	__u64	tcpi_bytes_received; /* RFC4898 tcpEStatsAppHCThruOctetsReceived */
	__u32	tcpi_segs_out;	     /* RFC4898 tcpEStatsPerfSegsOut */
	__u32	tcpi_segs_in;	     /* RFC4898 tcpEStatsPerfSegsIn */

	__u32	tcpi_notsent_bytes;
	__u32	tcpi_min_rtt;
	__u32	tcpi_data_segs_in;	/* RFC4898 tcpEStatsDataSegsIn */
	__u32	tcpi_data_segs_out;	/* RFC4898 tcpEStatsDataSegsOut */
	__u64   tcpi_delivery_rate;

	__u64	tcpi_busy_time;      /* Time (usec) busy sending data */
	__u64	tcpi_rwnd_limited;   /* Time (usec) limited by receive window */
	__u64	tcpi_sndbuf_limited; /* Time (usec) limited by send buffer */

	__u32	tcpi_delivered;
	__u32	tcpi_delivered_ce;

	__u64	tcpi_bytes_sent;     /* RFC4898 tcpEStatsPerfHCDataOctetsOut */
	__u64	tcpi_bytes_retrans;  /* RFC4898 tcpEStatsPerfOctetsRetrans */
	__u32	tcpi_dsack_dups;     /* RFC4898 tcpEStatsStackDSACKDups */
	__u32	tcpi_reord_seen;     /* reordering events seen */
};

```

当我们需要在连接的各个阶段查看状态变化时，可以通过 python 脚本进行打印：  

`+0      %{ print("RTO @1: ", tcpi_rto) }% `

脚本尝试在上一条命令执行结束后，立刻开始执行一段 python 脚本，打印 TCP 的超时重传时间。  

## Shell 命令  

packetdrill 支持 Shell 命令调用，实现了在测试执行过程中，插入一些额外操作，例如系统配置获取和设置。  

```
+0	`sysctl -q net.ipv4.tcp_timestamps=0`
```

上面的命令尝试在前一条脚本执行结束后，立刻执行一段 Shell 命令，命令调用 sysctl 修改 TCP 配置，关闭了时间戳。  

## 时间模式  

由于大部分网络协议对时间都比较敏感，因此 packetdrill 的每一行脚本都需要严格定义时间。如果事件发生的时间与预期不符，会进行报错，并提示事件实际发生的时间。  

在时间的定义上， packetdrill 也提供了多种模式的支持：  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/packetdrill_1/time_model.png){:height="500" width="500"}

在使用中，可能会出现定义的时间与实际时间存在偏差的情况，不同测试模式以及环境，偏差的多少往往也会不同，有时会难以控制。这种情况下，可以调整时间偏差的容忍值：  

```
// 在 time 时间偏差内都是可以容忍的
./packetdrill --tolerance_usecs={time}
```


## 本地/远程测试  

packetdrill 支持两种测试模式：本地模式与远程模式。  

* 本地模式  

本地模式可在单机环境下进行，packetdrill 使用本地单机和一个 TUN 虚拟网络设备进行数据包传输和测试，可用于测试系统调用、Sockets、TCP/IP协议层。  

本地模式是 packetdrill的默认模式，调用起来也十分简单：  

```
./packetdrill test.pkt
```

* 远程模式  

在远程模式下，我们运行两个独立的 packetdrill 进程，它们在两个不同的机器上，通过 LAN 进行数据传输，可用于测试整个网络系统，包括系统调用、Sockets、TCP、IP、NIC硬件和驱动、无线网络等。  

在使用远程模式时，需要定义好两端所扮演的角色，分为客户端和服务端。  

首先需要开启服务端，接收客户端连接请求：  

```
./packetdrill --wire_server
```

而在创建客户端进程时，需要指定角色及服务端参数：  

```
./packetdrill --wire_client --wire_server_ip={server_ip} test.pkt
```

两者通过 TCP 进行连接，并由客户端将脚本内容发送到服务端，两者共同协同完成整个脚本，整个过程主要用于测试客户端网络协议栈。  

## 总结  

packetdrill 语法并不复杂，能帮助我们学习和测试内核网络协议栈，模拟一些极端场景下的网络数据包。在 Google 发布的论文中，他们投入了超过18个月的时间，使用 packetdrill 来测试 Linux 内核网络模块，并且发现一些 bug，包括 DSACK undo、TCP Fast Open server等。
