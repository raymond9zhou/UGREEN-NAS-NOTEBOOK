# 绿联UGOS Pro双网口LACP链路聚合（端口聚合）后bond0不持有IP，br0持有IP后钉住默认网关

上次按照另外一个笔记方式处理后，发现每次重启都需要执行ip route add来添加默认网关，ifcfg配置文件中的post up脚本并不能起作用，我怀疑是系统本身在启动过程中加载了一次网关配置，但因为我改了bond0的配置，所以失效了，绿联技术也不告诉我网卡的启动逻辑于是只能使用非常僵硬的Systemd服务硬把网关钉上去。

**各位需要注意的是，此项仅为有运维经验人员的操作记录，并非手把手教程（如果其他教程博主有意愿写成手把手教程的，无需单独授权，注明出处即可），也不构成指引，因此默认受众为RHCSA知识具备者，数据无价，各位在进行调整前务必了解本人键入全部命令的实际含义再操作。**

## 1、编辑 /etc/systemd/system/force-br0-gateway.service 文件
```
[Unit]
Description=Force Add Default Gateway for br0 (Bypass UGOS)
After=network.target network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 15
ExecStart=/sbin/ip route add default via 192.168.166.X dev br0
ExecStart=/bin/true
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
## 2、启用并启动服务
```
sudo systemctl daemon-reload
sudo systemctl enable force-br0-gateway.service
```
