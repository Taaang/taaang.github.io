---
title: Linux 虚拟网络设备 - TUN/TAP
date: 2021-02-25
categories:
- 网络
tags:
- tun_tap
---

之前有尝试使用过`packetdrill`进行内核网络协议栈的测试，处于好奇想了解其底层是如何实现的，顺藤摸瓜，最终了解到 Linux 虚拟网络设备 TUN/TAP。  

## TUN / TAP  

`TUN/TAP`是 Linux 操作系统内核中的虚拟网络设备，为用户空间应用提供数据包的接收和传输能力，可以使应用实现简单的点到点数据包传输，与普通的物理网络传输不同，数据包的发送者和接收者都是用户空间应用。  

简单理解，`TUN/TAP`是虚拟化的网络设备，与物理网络设备的不同点在于，其一端是内核网络协议栈，另一端是用户空间应用。其中，`TUN`与`TAP`的区别在于，`TUN`工作在三层，可以操作 IP 数据包；`TAP`工作在二层，可以操作以太网帧。  

我们可以简单对比物理设备、`TUN/TAP`在数据传输过程中的不同：  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/tuntap/1.png){:height="500" width="500"}

## 设备配置  

一般 Linux 系统会创建默认`tun`字符设备文件，在目录`/dev/net`下。如果没有的话也没关系，我们可以使用`mknod`手动创建一个。  

```
// 创建一个 tun 字符设备文件，c 表示字符设备，10和200分别表示主次设备号
mknod tun c 10 200
```

> 主次设备号必须是 `10` 和 `200`吗？
>  
> 是的，和内核的设备码定义有关。内核定义设备码中， TUN/TAP 设备的主设备号就是`10`，次设备号是`200`。详细信息可以参考内核源码附带的文档，在`admin-guide/devices.txt`中，记录了所有设备码的定义。

紧接着，我们需要创建`tun`虚拟网络设备。  

```
//新增虚拟网络设备tun，模式为tun，命令会默认使用/dev/net/tun作为字符设备文件
ip tuntap add dev tun mode tun
```

此时，通过`ifconfig`就可以看到新的网络设备`tun`。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/tuntap/2.png){:height="500" width="500"}

> `tun`字符设备文件或虚拟网络设备的名字是固定的吗？
>  
> 并不是，字符设备文件和虚拟网络设备都可以使用其他名字。我做了一个简单测试，修改了`packetdrill`源码（platforms.h）中定义的`tun`设备路径和名字。重新编译后，使用新名字的`tun`设备依然有效。下面的代码例子也证明了虚拟网络设备可以使用其他名字代替。

## 接口开发  

我们可以通过`mknod`创建属于自己的字符设备文件，并通过`ip tuntap`创建虚拟网络`tun`设备，并通过正常的设备读写接口进行操作。  

举个栗子，我们来通过`tun`设备实现 ICMP 请求应答过程，同时我们换个方式，通过代码来创建设备，先来看看整个过程。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/tuntap/3.png){:height="500" width="500"}

首先，需要创建了新的设备描述文件`/dev/net/mmq`  

```
//创建tun设备描述文件
mknod mmq c 10 200
```

并通过代码创建虚拟网络设备`mmq-eth`，并与设备描述文件`/dev/net/mmq`进行关联。  

```
int create_tun_device(char *dev_name, int flag) {
        struct ifreq ifr;
        int fd, err_code;

        // 创建 tun fd
        if ((fd = open("/dev/net/mmq", O_RDWR)) < 0) {
                printf("Tun device fd create error. (%d)\n", fd);
                return fd;
        }

        // 配置 ifreq，它是一个用于socket ioctl配置的结构
        memset(&ifr, 0, sizeof(ifr));
        strcpy(ifr.ifr_name, dev_name);
        ifr.ifr_flags |= flag;

        // 配置 tun 设备
        if ((err_code = ioctl(fd, TUNSETIFF, &ifr)) < 0) {
                printf("Tun device ioctl configure error. (%d)\n", err_code);
                close(fd);
                return err_code;
        }

        return fd;
}
```

通过得到的`tun`设备文件描述符，我们可以进行基础的读写，完成整个 ICMP 请求和应答过程。  

```
void main(int argc, char* argv[]) {

        //创建 tun 设备
        int tun_fd = create_tun_device("mmq-eth", IFF_TUN);
        if (tun_fd < 0) {
                return -1;
        }

        //不断尝试从设备中读取信息
        int ret_length = 0;
        unsigned char buf[1024];
        while (1) {
                ret_length = read(tun_fd, buf, sizeof(buf));
                if (ret_length < 0) {
                        break;
                }

                //分析报文
                unsigned char src_ip[4];
                unsigned char dst_ip[4];
                memcpy(src_ip, &buf[16], 4); //16~19为IP源地址
                memcpy(dst_ip, &buf[20], 4); //20~23为IP目标地址

                //返回icmp应答报文
                memcpy(&buf[16], dst_ip, 4);
                memcpy(&buf[20], src_ip, 4);
                buf[24] = 0; //ICMP报文类型设置为应答
                ret_length = write(tun_fd, buf, ret_length);
        }

        //关闭设备
        close(tun_fd);
}
```

程序成功运行后，我们可以在网络设备列表中找到对应的设备。  

```
ifconfig -a
```

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/tuntap/4.png){:height="500" width="500"}

此时，我们需要往`mmq-eth`发送ICMP请求报文，需要先开启设备，并将在 IP 路由表中创建一条到该设备的 IP路由记录。  

```
//开启网络设备
ifconfig mmq-eth up
//增加IP路由记录
route add -host 10.10.10.1 dev mmq-eth
```

再通过`ping`命令发送报文，就可以看到整个 ICMP 请求应答过程。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/tuntap/5.png){:height="500" width="500"}

（GitHub 代码地址：[Taaang/tun_tap_test · GitHub](https://github.com/Taaang/tun_tap_test)）

## 总结  

`TUN/TAP`为用户空间应用，提供了网络协议栈中不同层的数据包传输，`tun`提供简单的点对点数据包传输，`tap`则在以太网层进行数据帧传输，其主要应用于 VPN、 协议加密和压缩、隧道等。  
