---
layout: post
title: 让物理服务器变成OvS
category: openvswitch
---

OpenvSwitch是个虚拟服务，但是可以通过将OvS与Host上的物理网卡绑定以获得一个搭载openvswitch功能的物理“交换机”。

1. 创建bridge. `ovs-vsctl add-bridge br0`
2. 添加Host上的物理网卡到br0. `ovs-vsctl add-bridge br0 eth0`  注意，这一步会导致本来与IP stack连接的eth0现在连接到br0上，暂时失去与outer world的联系。
3. `ovs-vsctl add-bridge br0 eth1`如果br0需要连接不止一个物理网卡，则需要开启STP
4. 清空eth0的配置. `ifconfig eth0 0`
5. 配置br0的ip/netmask, gateway。
6. 修改eth0 & eth1所连host的default gateway。

这时候，就可以给ovs设定controller了。

