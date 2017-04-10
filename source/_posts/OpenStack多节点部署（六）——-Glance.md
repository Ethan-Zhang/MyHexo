title: OpenStack多节点部署（六）—— Glance
date: 2013-02-27 17:17:31
tags: [OpenStack]
---
* [__OpenStack多节点部署（一）——服务器选型__](/2013/01/31/OpenStack多节点部署（一）——-服务器选型/)
* [__OpenStack多节点部署（二）——操作系统安装__](/2013/02/01/OpenStack多节点部署（二）——操作系统安装/)
* [__OpenStack多节点部署（三）——网络配置__](2013/02/04/OpenStack多节点部署（三）——网络配置/)
* [__OpenStack多节点部署（四）——KeyStone__](2013/02/04/OpenStack多节点部署（四）——KeyStone/)
* __OpenStack多节点部署（五）——Nova__
* [__OpenStack多节点部署（六）——glance__](2013/02/27/OpenStack多节点部署（六）——-Glance/)

>本章将为大家介绍OpenStack的镜像管理系统glance，负责存储和管理OpenStack的镜像和虚拟机的快照等。本章将是OpenStack多节点部署系列的最后一章了，在此也感谢大家一直以来对我的支持。还是那句话，stacker们大家可以在留言区多交流，我看到会及时回复的。
glance的安装部署比较简单，前半部分花一些时间讲解glance部署步骤，后面的篇幅主要介绍各类操作系统镜像制作的方方面面。

首先安装glance的软件包
```
sudo apt-get install glance glance-api glance-client glance-common glance-registry python-glance
```
修改glance api的配置文件，主要是填写在keystone中设置的用户名密码等
```
sed -i '/admin_tenant_name/ s/%SERVICE_TENANT_NAME%/service/' /etc/glance/glance-api-paste.ini
sed -i '/admin_user/ s/%SERVICE_USER%/glance/' /etc/glance/glance-api-paste.ini
sed -i '/admin_password/ s/%SERVICE_PASSWORD%/' /etc/glance/glance-api-paste.ini
```
修改glance api的配置文件，增加keystone的支持
```
sed -i '$a [paste_deploy]' /etc/glance/glance-api.conf
sed -i '$a flavor = keystone' /etc/glance/glance-api.conf
```
修改glance组件注册文件，修改数据库地址及令牌
```
sed -i '/sql_connection/s/=.*$/= mysql:\/\/glancedbadmin:glancesecret@192.168.3.1\/glance' /etc/glance/glance-registry.conf
```
重启glance
```
sudo restart glance-api
sudo restart glance-registry
```
验证glance是否安装成功，成功会返回0
```
glance index
```
另外说一下glance的配置位置，如果大家的镜像和虚拟机快照不是非常多的话，建议就和nova的控制节点装在一起就行了，如果快照特别多，可以考虑glance单配一台服务器。
下面给大家分享镜像制作方面的一些经验。镜像制作一定要单开一台服务器或者在客户机client上制作，如果其放在控制节点或计算节点上，制作过程中由于会占用服务器大量资源导致其他服务拖慢卡顿。制作完成后拷贝镜像文件到glance image服务器，或在客户机上安装glance-client服务即可。
首先创建镜像盘，这个所有镜像的制作步骤都一样
```
kvm-img create -f qcow2 server.img 5G
```
镜像文件有多种格式，常见的是qcow2和raw，qcow2是增量式的，raw是非增量式的。
制作Ubuntu镜像
```
sudo kvm -m 256 -cdrom ubuntu-12.04-server-amd64.iso -drive file=server.img,
if=virtio,index=0 -boot d -net nic -net user
```
按照正常安装系统的步骤安装，可一路点默认设置下来，重启后安装ssh和cloud-init模块。网上的安装教程里面多半是这样的
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install openssh-server cloud-init
#Remove the network persistence rules from /etc/udev/rules.das their presence will result in
#the network interface in the instance coming up as an interfaceother than eth0.
sudo rm -rf /etc/udev/rules.d/70-persistent-net.rules
```
这个是错误的，cloud-init不能按照默认配置安装，要修改器安装脚本，否则安装后cloud-init访问不到OpenStack的metadata元数据服务器
正确的安装方法为
```
sudo apt-get update
sudo apt-get upgrade
echo "cloud-init cloud-init/datasources string NoCloud, OVF,Ec2" > /tmp/debconf-selections
/usr/bin/debconf-set-selections /tmp/debconf-selections
rm -f /tmp/debconf-selections
apt-get -y install cloud-init
sudo rm -rf /etc/udev/rules.d/70-persistent-net.rules
```
其实ubuntu的官网上有做好的UEC镜像的下载[地址](http://cloud-images.ubuntu.com/releases/12.04.2/release-20130222/)。可直接`wget http://cloud-images.ubuntu.com/releases/12.04.2/release-20130222/ubuntu-12.04-server-cloudimg-amd64-disk1.img`下载12.04 64位的镜像
制作CentOS或Fedora镜像
因为CentOS、Fedora没有cloud-init模块，所以要用curl从metadata元数据服务器获得主机名等信息
```
echo >> /root/.ssh/authorized_keys
curl -m 10 -s http://169.254.169.254/latest/meta-data/public-keys/0/openssh-
key | grep 'ssh-rsa' >> /root/.ssh/authorized_keys
echo "AUTHORIZED_KEYS:"
echo "************************"
cat /root/.ssh/authorized_keys
echo "************************"
```
另外还要修改/etc/sysconfig/network-scripts/ifcfg-eth0，将网卡的硬件串号等信息删除，设为DHCP模式
```
DEVICE="eth0"
BOOTPROTO=dhcp
NM_CONTROLLED="yes"
ONBOOT="yes"
```
删除网卡网口绑定规则，否则将导致重启后网络不可用
```
rm -rf /etc/udev/rules.d/70-persistent-net.rules
```
最后上传镜像
```
glance add name="xxxx-linux" is_public=true container_format=ovf disk_format=qcow2 < server.img
```
制作windows镜像
```
kvm-img create -f qcow2 windowsserver.img 20G
sudo kvm -m 1024 -cdrom 7601.17514.101119-1850_x64fre_server_eval_zh-cn-GRMSXEVAL_CN_DVD.iso -drive file=windowsserver2008-enterprise.img,if=virtio -boot d -drive file=virtio-win-0.1-30.iso,index=3,media=cdrom -device virtio-net-pci -net nic -net user
```
由于windows server 2008默认没有带virtio的驱动，所以在启动镜像安装的时候我们要带上这个驱动盘，[下载地址]( http://alt.fedoraproject.org/pub/alt/virtio-win/latest/images/bin)。进入系统后，要将virtio驱动的网卡和磁盘驱动都装好，切记别忘了安装网卡驱动，安装完成后会显示两个网卡，镜像在nova再次启动后就只会剩下virtio驱动的虚拟网卡。
最后一步，上传镜像文件，大功告成！
```
glance add name="windows" is_public=true container_format=ovf disk_format=qcow2 < windowsserver.img
```
