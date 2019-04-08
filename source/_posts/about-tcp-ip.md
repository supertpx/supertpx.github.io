---
title: 关于TCP/IP协议
date: 2019-04-03 15:11:53
categories: 
- 网络
---

>TCP/IP协议详解，两个重点：协议的层次，client和server之间的三握四挥

### TCP/IP协议模型

{% asset_img tcp模型.jpg TCP模型 %}

<!-- more -->

和传统的OSI网络模型的7个层次相比，TCP模型进行了进一步的封装，共有四个层次，由上至下依次如下：

- 应用层：向用户提供一组常用的应用程序，比如电子邮件、文件传输访问、远程登录等。远程登录TELNET使用TELNET协议提供在网络其它主机上注册的接口。TELNET会话提供了基于字符的虚拟终端。文件传输访问FTP使用FTP协议来提供网络内机器间的文件拷贝功能。
- 传输层：提供应用程序间的通信。其功能包括：
一、格式化信息流；
二、提供可靠传输。为实现后者，传输层协议规定接收端必须发回确认，并且假如分组丢失，必须重新发送。
- 网络层：负责相邻计算机之间的通信。其功能包括三方面：
一、处理来自传输层的分组发送请求，收到请求后，将分组装入IP数据报，填充报头，选择去往信宿机的路径，然后将数据报发往适当的网络接口。
二、处理输入数据报：首先检查其合法性，然后进行寻径--假如该数据报已到达信宿机，则去掉报头，将剩下部分交给适当的传输协议；假如该数据报尚未到达信宿，则转发该数据报。
三、处理路径、流控、拥塞等问题。
- 网络接口层：这是TCP/IP软件的最低层，负责接收IP数据报并通过网络发送之，或者从网络上接收物理帧，抽出IP数据报，交给IP层。

### TCP三握四挥

TCP在建立固定连接时，client与server之前会有三次握手；而在释放连接时，两者之间又会有四次挥手。

TCP传送数据格式如图所示：

{% asset_img TCP数据格式.png TCP数据格式 %}

其中，我们需要关注的地方有，位码即tcp标志位，有6种标示：SYN(synchronous建立联机)、ACK(acknowledgement 确认)、PSH(push传送)、FIN(finish结束)、RST(reset重置)、URG(urgent紧急)，还有两个数字位：Sequence number(序号)、Acknowledge number(确认号)。

#### 三次握手

{% asset_img tcp三次握手.png tcp三次握手 %}

TCP三次握手过程如下：
a.第一次握手：Client将标志位SYN置为1，随机产生一个值seq=J，并将该数据包发送给Server，Client进入SYN_SENT状态，等待Server确认。
b.第二次握手：Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位SYN和ACK都置为1，ack number=J+1，随机产生一个值seq=K，并将该数据包发送给Client以确认连接请求，Server进入SYN_RCVD状态。
c.第三次握手：Client收到确认后，检查ack number是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack number=K+1，并将该数据包发送给Server，Server检查ack number是否为K+1，ACK是否为1，如果正确则连接建立成功，Client和Server进入ESTABLISHED状态，完成三次握手，随后Client与Server之间可以开始传输数据了。

#### 四次挥手

{% asset_img tcp四次挥手.png tcp四次挥手 %}

TCP四次挥手过程如下：
a.第一次挥手：Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态。
b.第二次挥手：Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态。
c.第三次挥手：Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态。
d.第四次挥手：Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手。

#### why三握四挥

因为当Server端收到Client端的SYN连接请求报文后，可以直接发送SYN+ACK报文。其中ACK报文是用来应答的，SYN报文是用来同步的。但是关闭连接时，当Server端收到FIN报文时，很可能并不会立即关闭SOCKET，所以只能先回复一个ACK报文，告诉Client端，"你发的FIN报文我收到了"。只有等到我Server端所有的报文都发送完了，我才能发送FIN报文，因此不能一起发送。

#### why 2MSL

按道理，四个报文都发送完毕，我们可以直接进入CLOSE状态了，但是我们必须假象网络是不可靠的，有可以最后一个ACK丢失。所以TIME_WAIT状态就是用来重发可能丢失的ACK报文。在Client发送出最后的ACK回复，但该ACK可能丢失。Server如果没有收到ACK，将不断重复发送FIN片段。所以Client不能立即关闭，它必须确认Server接收到了该ACK。Client会在发送出ACK之后进入到TIME_WAIT状态。Client会设置一个计时器，等待2MSL的时间。如果在该时间内再次收到FIN，那么Client会重发ACK并再次等待2MSL。所谓的2MSL是两倍的MSL(Maximum Segment Lifetime)。MSL指一个片段在网络中最大的存活时间，2MSL就是一个发送和一个回复所需的最大时间。如果直到2MSL，Client都没有再次收到FIN，那么Client推断ACK已经被成功接收，则结束TCP连接。

