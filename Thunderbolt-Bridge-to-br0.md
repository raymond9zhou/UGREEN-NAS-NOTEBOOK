# 绿联UGOS Pro双网口LACP链路聚合（端口聚合后）后配置雷电桥接至聚合口
为了我的两个Mac连接nas快一点，我通过雷电连接至我的nas，但是UGOS Pro系统又是只支持非常不符合使用习惯的操作方式：需要重新登录至一个单独的雷电IP，不能直接桥接网络，导致需要同时准备两个IP的连接逻辑，询问客服得到的反馈不出意外是“暂不支持，需要提需求等反馈”。于是继续自己动手丰衣足食吧！

**各位需要注意的是，此项仅为有运维经验人员的操作记录，并非手把手教程（如果其他教程博主有意愿写成手把手教程的，无需单独授权，注明出处即可），也不构成指引，因此默认受众为RHCSA知识具备者，数据无价，各位在进行调整前务必了解本人键入全部命令的实际含义再操作。**
## 编辑/etc/network/interfaces.d/ifcfg-thunderbolt0，这里假设聚合口为br0
```
allow-hotplug thunderbolt0
iface thunderbolt0 inet manual
    post-up ip link set thunderbolt0 master br0 || true
    post-up ip link set thunderbolt0 up || true
#auto thunderbolt0
#iface thunderbolt0 inet manual
#iface thunderbolt0 inet6 manual
```
