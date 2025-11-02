# 绿联UGOS Pro双网口LACP链路聚合（端口聚合后）后虚拟机UI提示仅能使用NAT和仅主机模式，通过SSH实现桥接模式的过程记录

目前使用DXP6800Pro，双网口LACP链路聚合后虚拟机仅能使用NAT和仅主机模式，这对于我从esxi迁移过来来讲是不可用和不可接受的，对于任何购买此售价产品的专业用户来讲也是难以满足需求的，询问客服得到的反馈是“暂不支持，需要提需求等反馈”。我是特意为了虚拟机购入了1Tx2 的m.2组了RAID1，又升级内存到32Gx2的，让我没法bridge这肯定受不了，于是开始自行找办法。
之前看线上各处的宣传说系统发行版本是Debian，虚拟机使用的是KVM，于是SSH连接查看了配置，因前期使用的是威联通、群晖，习惯性的以为微信客服可以协助确认系统各配置文件的存放位置、依赖关系，结果客服回复是“这个涉及到底层这边是不提供技术支持的，没办法满足您的需求”，于是只能结合个人经验提心吊胆的尝试（我的数据已经321原则备份，如各位数据未经妥善备份，还是不要像我这样莽），最终实现了虚拟机桥接，记录一下供各位有同样需求的苦主可以在官方更新前使用。

**各位需要注意的是，此项仅为有运维经验人员的操作记录，并非手把手教程（如果其他教程博主有意愿写成手把手教程的，无需单独授权，注明出处即可），也不构成指引，因此默认受众为RHCSA知识具备者，数据无价，各位在进行调整前务必了解本人键入全部命令的实际含义再操作。**

# 整体调整思路：
将网络流量路径变为：虚拟机（不再经过libvirt管理虚拟网络）-> ssh命令手动建立的网桥 (br0) -> UGOS的UI建立的LACP的bond接口 (bond0) -> 物理交换机
IP地址将不再配置在bond0接口上，而是转移到 br0 网桥接口上。bond0接口本身只负责数据链路层的聚合，不持有IP。
# 操作流程：
## 1.查看/etc/network/interfaces得到如下输出内容
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/ifcfg-*

# The loopback network interface
auto lo
iface lo inet loopback
```
看来配置都在/etc/network/interfaces.d/ifcfg-*下面，观察一下/etc/network/interfaces.d的文件内容
```
root@XXXXXX:/etc/network# cd /etc/network/interfaces.d/ && ls -la
total
drwxr-xr-x 1 root root 4096 Nov  2 13:56 .
drwxr-xr-x 1 root root 4096 Mar 25  2025 ..
drw-r----- 2 root root 4096 Nov  2 15:56 bak
-rw-r----- 1 root root  223 Nov  2 16:36 ifcfg-bond0
-rw------- 1 root root    0 Oct 19 21:27 .ifcfg-bond0.lock
-rw------- 1 root root    0 Oct 13 23:36 .ifcfg-eth0.lock
-rw------- 1 root root    0 Oct 13 23:36 .ifcfg-eth1.lock
-rw------- 1 root root    0 Oct 19 13:18 .ifcfg-thunderbolt0.lock
```
果然看到ifcfg-bond0文件，文件夹中的eth0和1的锁文件不知为何没有被移除，猜测可能是和bak文件夹一起供UGOS系统用来进行fallback回溯的？但因为无法联系到实际开发人员交流，本着能跑就别动的思想，未进行删除。
查看ifcfg-bond0，目前是根据UGOS UI配置了IP
```
auto bond0
iface bond0 inet static
address 192.168.166.X
netmask 255.255.255.0
gateway 192.168.166.1
dns-nameservers 223.5.5.5 223.3.3.3
hwaddress 6c:1f:f7:XX:XX:XX
bond-slaves eth0 eth1
bond-mode 802.3ad
bond-lacp-rate fast
bond-miimon 100
bond-updelay 100
bond-downdelay 0
iface bond0 inet6 dhcp
```
## 2.修改ifcfg-bond0，改成bond0接口本身只负责数据链路层的聚合，不持有IP
```
auto bond0
iface bond0 inet manual
    bond-slaves eth0 eth1
    bond-mode 802.3ad
    bond-lacp-rate fast
    bond-miimon 100
    bond-updelay 100
    bond-downdelay 0
    hwaddress 6c:1f:f7:8d:f5:22
```
## 3.新增ifcfg-br0，持有IP并bridge至bond0，为了和其他端口保持一致，新增后我也touch了一个锁文件。
```
auto br0
iface br0 inet static
    address 192.168.166.X
    netmask 255.255.255.0
    gateway 192.168.166.1
    dns-nameservers 223.5.5.5 223.3.3.3
    bridge_ports bond0      
    bridge_stp off          
    bridge_fd 0
iface br0 inet6 dhcp
```
## 4.重启网络服务
```
systemctl restart networking.service
```
## 5.发现网络中断了
本想通过HDMI连接，但觉得插拔网线有点困难，正好是Mac手边有雷电线，通过雷电网桥重新ssh。执行ip addr 和brctl show，确认br0已经创建成功，bond0按照预期工作。对端交换机检查LACP状态，显示激活正常。
执行ip route发现，未有默认网关，强制设置默认网关为br0
```
ip route add default via 192.168.166.1 dev br0
```
## 6.Shell网络正常，等待几分钟后，UGOS UI和UGLink恢复正常。

## 7.配置虚拟机使用br0

首先尝试使用UI配置，发现UI只能选择libvirt管理的网络，无法配置interface为bridge。
遂继续ssh连接，尝试执行virsh list --all
```
 Id   Name                                   State
------------------------------------------------------
 1    XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 2    XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 3    XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 4    XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 5    XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 6    XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 7    XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 8    XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 9    XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 10   XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 11   XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
 12   XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   running
```
与创建的虚拟机数量一致，执行
```
virsh edit XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```
进入单个虚拟机xml配置编辑页面，文件清晰可读，title与配置的虚拟机名称相同。
找到形如下列的interface段。
```
<interface type='network'> 
  <mac address='52:54:00:ab:cd:ef'/>
  <source network='vnet-bridge0'/> 
  <model type='virtio'/>
  ...
</interface>
```
修改interface的type为bridge，source改为bridge='br0'，如下
```
<interface type='bridge'>  <-- type改为 'bridge'
  <mac address='52:54:00:ab:cd:ef'/>
  <source bridge='br0'/>  <-- source改为 bridge='br0' 
  <model type='virtio'/>
  ...
</interface>
```
## 8.执行完整的关闭后开启动作，使配置文件生效
```
virsh shutdown XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
virsh start XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```
## 9.测试各项虚拟机功能、UGOS系统功能表现正常。
