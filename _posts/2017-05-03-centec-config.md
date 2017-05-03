---
layout: post
title: 盛科交换机配置
category: tech
---
{% include JB/setup %}

今天从老师那边拿到盛科V150交换机，需要验证下openflow的正常转发功能。所以我需要做的事情有：

1. 给交换机配置好IP以及Telnet/SSH，以便远程控制；
2. 两个端口分别连接服务器，配置好SDN控制器的IP端口等；
3. 服务器A ping B，抓包看是否走的Openflow协议；

整体比较顺，但中间也遇到了些问题，这里记录下。

**给交换机配IP和开启SSH/Telnet**

首先你得有一个win7的电脑，因为我是笔记本，需要用到串口转usb线，所以要安装相应驱动。之后用串口线连到交换机后，发现在SecureCRT里无法输入，解决方法：
> Session Options -> Connection -> Serial -> Flow Control，将原先默认选中的 RTS/CTS取消掉，再重新connect开发板。

[参考链接](http://www.crifan.com/resolved_can_not_enter_under_the_serial_securecrt/)

然后给管理口配IP，开启SSH/Telnet服务。到一步后我可以从其它机器ping通交换机，但一直远程不上。后来**给交换机配了route**后方可。这里原因未知。

验证了使用ODL下的二层转发功能，目前没发现什么问题，毕竟高级功能也还没接触到。

盛科的很多命令和思科比较类似，但比较奇怪的一点是在`configure terminal`模式下竟然不能执行非conf t下的命令（很多show相关的），需要退出去查看，不是很方便。

最后，我想发个希望，之后最好能尽量少地接触配置机器（尤其交换机）这种事情，我大概很不适合做运维。

