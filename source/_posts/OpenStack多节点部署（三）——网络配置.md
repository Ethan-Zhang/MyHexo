title: OpenStack多节点部署（三）——网络配置
date: 2013-02-04 04:11:47
tags: [OpenStack]
---
* [__OpenStack多节点部署（一）——服务器选型__](/2013/01/31/OpenStack多节点部署（一）——-服务器选型/)
* [__OpenStack多节点部署（二）——操作系统安装__](/2013/02/01/OpenStack多节点部署（二）——操作系统安装/)
* [__OpenStack多节点部署（三）——网络配置__](2013/02/04/OpenStack多节点部署（三）——网络配置/)
* [__OpenStack多节点部署（四）——KeyStone__](2013/02/04/OpenStack多节点部署（四）——KeyStone/)
* __OpenStack多节点部署（五）——Nova__
* __OpenStack多节点部署（六）——glance__

　　上一章，我和大家讨论了操作系统安装的方方面面，这章将带来OpenStack的网络配置。
初接触OpenStack的人，在看了部署文档之后，可能会被Nova-Network，Fix-ip，Floating-ip等概念弄的一头雾水，下面会一一详细道来。
![Nova-Network](http://7xpwqp.com1.z0.glb.clouddn.com/2013-02-04-01)
　　上图大家可能看的不是很明白，其实OpenStack的nova-network部署可以分成3个网段。公网网段，内网网段和虚拟机网段。公网网段指的是可以直接访问到互联网的网段（但是此IP不一定非要从公司外部也能访问，这里的内外是从OpenStack网络系统出发而不是从公司网络的视角出发，请注意），也就是Floating IP配置的网段，我把它绑定在了Eth0。内网网段指的是OpenStack系统内各服务器之间互联的顶层交换机所划的网段，这里将其设置为192.168.3.X，此网段无需出口，我把它绑定在了Eth1，在公司的网络也就是公网网络是访问不到的。虚拟机网段指的是虚拟机运行时其IP所在的网段，也就是Nova-Network中提到的Fix-IP，由于NOVA底层是采用libvirt，所以OpenStack内部默认将其设置为桥接网络br100，这里将其桥接在内网网络上，也就是Eth1。
　　在服务器选型的时候，我们已经选择了千兆双网卡的机型。这样公网网段和内网网段就可以绑定到两个不同的网口。有的实验环境中，比如沙克老师的文章将两个公网和内网两个网段都配置在了一个网卡上，在实验环境是可以的，但是生产环境不推荐，因服务器如果和外部网络数据交换量大的时候，会使内部服务如nova-scheduler，nova-volume等服务带来延迟。
　　我的服务器网络配置
```Bash
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
address 10.2.15.3
netmask 255.255.255.0
broadcast 10.2.15.255
gateway 10.2.15.1
dns-nameservers 8.8.8.8

auto eth1
iface eth1 inet static
address 192.168.3.1
netmask 255.255.255.0
network 192.168.3.0
broadcast 192.168.3.255
```
　　上面提到的Flat DHCP Network是Nova-Network服务的一种模式。另外还有Flat，VLan等，这里就不一一介绍了。Flat DHCP指的是所有虚拟机处在同一个平坦的网段上，且DHCP服务自动为其分配IP。
　　配置Fixed-IP命令
```Bash
sudo nova-manage network create private --fixed_range_v4=192.168.3.32/27 --num_networks=1 --bridge=br100 --bridge_interface=eth1 --network_size=32
```
　　使用此命令，我们可以详细配置虚拟机网段Fixed-IP所在的网段，掩码，桥接网络名称，桥接网络端口等参数，后面在NOVA章将详细介绍。
