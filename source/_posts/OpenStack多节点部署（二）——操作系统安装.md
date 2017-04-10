title: OpenStack多节点部署（二）——操作系统安装
date: 2013-02-01 13:08:25
tags: [OpenStack]
---
* [__OpenStack多节点部署（一）——服务器选型__](/2013/01/31/OpenStack多节点部署（一）——-服务器选型/)
* [__OpenStack多节点部署（二）——操作系统安装__](/2013/02/01/OpenStack多节点部署（二）——操作系统安装/)
* [__OpenStack多节点部署（三）——网络配置__](2013/02/04/OpenStack多节点部署（三）——网络配置/)
* [__OpenStack多节点部署（四）——KeyStone__](2013/02/04/OpenStack多节点部署（四）——KeyStone/)
* __OpenStack多节点部署（五）——Nova__
* [__OpenStack多节点部署（六）——glance__](2013/02/27/OpenStack多节点部署（六）——-Glance/)

　　上一章，为大家介绍了搭建云计算运营系统的服务器选型，相信经过这段时间，大家都已经成功采购到了符合业务量需求的服务器配置。下面将分享操作系统的安装过程中的思路：
　　如果您已经从WIKI得到了OpenStack的部署文档，比如`os-compute-starterguide-trunk`，您会发现文档中的安装演示步骤都是以Ubuntu为模板的。这也不足为怪，因为OpenStack基金会与Ubuntu的东家Canonical合作甚密；并且文档是以搭建实验开发平台为读者目标进行编攒的，以Ubuntu的apt-get管理方式进行OpenStack的安装确实也会给初学者带来许多方便。但是，如果您需要搭建一个多节点平台，一个真正需要运营的生产环境，还是建议使用CentOs，或者是商业系统RedHat等。因为毕竟Ubuntu开发中面向的群体主要还是桌面用户，对于服务器的支持并不是十分友好。刚开始接触OpenStack的时候，我也是按照教程使用的Ubuntu12.04作为操作系统，碰到了许多问题，拖延了部署的时间。
1. 安装后开机不进入系统，显示并定在了initramfs，这是因为服务器启动时间过长让Ubuntu系统错认为是某些硬件系统初始化失败，系统默认的超时时间过短，我们需要在`etc/default/grub`中修改`GRUB_CMDLINE_LINUX_DEFAULT="rootdelay=600"`，调大超时的时间，并且更新grub`sudo grub-update`
2. 在`/etc/network/interfaces`中修改网络配置后重启网络服务，发现IP并没有改为我们在配置文件中设置的IP，发现是因为启动了Gnome的桌面，桌面系统中的网络管理软件接管了网络服务，需要在桌面右上角手动将其禁用。
3. Ubuntu没完没了的更新，让你需要经常重启服务器，造成服务的中断。

　　种种问题因篇幅不一一赘述，虽然说Ubuntu在桌面电脑的表现上还是上佳的，但是服务器上我们还是选用较为稳定的CentOs6.3吧。
　　安装CentOS6.3最好为光盘安装，在mirrors下载CentOS-6.3-x86_64-bin-DVD1.iso，刻录一张DVD光盘。如果没有光驱，可以用ultraISO制作U盘安装盘，需要注意的是制作完U盘安装盘后还要将CentOS-6.3-x86_64-bin-DVD1.iso这个文件也拷贝到U盘的根目录下，并且要将bios的启动方式从bios方式改为EFI方式。
　　计算节点的分区可以选择默认模式，如果是控制节点推荐手动分区，因为要安装nova-volume服务，需要一个独立的LVM卷。
　　CentOS6.3网络设备的标识方法从Eth0改为了Em1，在做配置的时候需要注意。最近已经成功测试了在R510/710系列上利用自动化部署cobbler自动安装CentOS6.3系统，稍后有时间会给大家分享。

　　先写到这里，后面章节会给大家陆续介绍OpenStack各个组件在CentOS上的安装。
