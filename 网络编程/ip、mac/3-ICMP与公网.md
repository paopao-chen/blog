# ICMP
互联网控制报文协议，主要用于主机或路由器之间的错误信息或其他控制信息的交互

报文在互联网中传递，总不可避免的会出现问题，传递差错信息或通过一种方式获取一些非传输数据的信息，所以就有了ICMP

ICMP介于传输层和网络层之间，ICMP的报文封装在IP包中，但它的功能主要服务于传输层

![](img/5.jpg)

ICMP的报文有很多类型，不同类型有不同的代码。常用类型有：

- 主动请求 8 
- 主动应答 0
- 终点不可达为 3
- 源抑制为 4
- 超时为 11
- 重定向为 5

## ping

ping就是主动请求的查询报文，并获取主动应答的ICMP协议。所以，ping包也是符合ICMP协议格式，但是后面增加了自己的格式

对ping的主动请求，进行网络抓包，成为ICMP ECHO REQUEST，主动请求回复成为ICMP ECHO REPLY。比起原生的ICMP多了两个字段，
一个是标识符，表示发送的第几个包，一个是发送请求的时间，用来计算往返时间

## 差错报文类型

### 终点不可达
终点不可达报文类型为3，代码又分很多种，表示各种不可达的原因

- 网络不可达代码为 0
- 主机不可达代码为 1
- 协议不可达代码为2
- 端口不可达代码为 3
- 需要进行分片但设置了不分片位代码为4


### 源站抑制
让发送方放慢发送速度，一般是路由器丢包太严重，会回复发送包的主机

### 时间超时
一般是超过了网络包的生存时间TTL，但是还没到，路由器会返回

### 路由重定向
路由器回复让主机下次发给另一个路由器，别绕路了

## 使用
ICMP的使用途径很多，通过各种技巧，你能探测到最小MTU，能探测到途径路由器的所有IP等等

# 网关Gateway
![](img/7.jpg)

前面讲IP、MAC、ARP、ICMP、Ping，交换机和路由器，都是在局域网下讲解的，如果我们连入公网，他们是怎样运作的？

在任何一个机器上，如果你想访问一个IP，都会先判断，这个目标IP是否和当前机器IP在同一网段下，如果在同一网段下，则将源和目的IP封装报文，查询缓存或者大吼一声获取到目的IP的MAC地址，并发个目的IP

**如果不是同一网段下，就需要发给默认网关**

你的电脑通常会配有默认网关的地址，Gateway 的地址一定是和源 IP 地址是一个网段的，一般不是第一个地址就是第二个地址，例如：192.168.0.1，一般都是通过私网地址访问网关

当发现目的IP不在当前网段下，主机也就无法获取到目的IP的MAC地址，它只能把源IP和目的IP封装一下，将目的MAC写为网关的MAC地址，然后将报文交给网关处理。

网关通常是一个路由器(内嵌了网关处理程序)，或者由一台主机充当。它是一个第三层设备，接收到包之后，它发现目的MAC和自己MAC相同，就会拆包，并查看目的IP做处理

通常我们把网关称作路由器，其实路由器还有更精准的定义：它是一个有着五个网口或网卡的设备，每个网口都连接一个局域网

任何想被发到其他网段的包都会被路由器处理，它会使用路由算法给包加上新的MAC地址和IP地址，并发送出去

路由算法主要分两大类，静态路由和动态路由

## 静态路由

