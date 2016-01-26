title: OpenStack解决非UEC镜像的虚拟机cloud-init不工作不能自动修改主机名称不能注入user data
date: 2012-10-31 11:26:13
tags: [OpenStack, cloud-init]
---
　　最近几天因为系统定制化的需求，集中研究了一下OpenStack的user_data的相关细节OpenStack虚拟机的用户客制化方法（User Data），上一遍博文最后提到了只成功测试了ubuntu的cloudimage，而自己用mirrors上下载的iso镜像创建image后即使安装了cloud-init这个包，也会出现诸如在OpenStackdashboard中创建虚拟机时所填的主机名并未导入，user_data不可用等问题，查看cloud-init的log会显示。
<!--more-->

```Bash
2012-10-30 14:57:43,189 - cloud-init[INFO]:cloud-init start running: Tue, 30 Oct 2012 06:57:43 +0000. up 6.99seconds
2012-10-30 14:57:43,249 - __init__.py[DEBUG]: searching for datasource in ['DataSourceNoCloudNet', 'DataSourceConfigDriveNet','DataSourceOVFNet', 'DataSourceMAAS']
2012-10-30 14:57:43,269 - __init__.py[DEBUG]: Did not find datasource. searched classes: ['DataSourceNoCloudNet','DataSourceConfigDriveNet', 'DataSourceOVFNet','DataSourceMAAS']
```
　　而正常的log应该会显示
```Bash
2012-10-31 10:56:38,008 - cloud-init[INFO]: cloud-init startrunning: Wed, 31 Oct 2012 02:56:37 +0000. up 8.83 seconds
2012-10-31 10:56:38,050 - __init__.py[DEBUG]: searching for datasource in ['DataSourceNoCloudNet', 'DataSourceOVFNet','DataSourceEc2']
2012-10-31 10:56:53,270 - DataSourceEc2.py[DEBUG]: Using metadatasource: 'http://169.254.169.254'
2012-10-31 10:56:53,329 - DataSourceEc2.py[DEBUG]: crawl ofmetadata service took 0s
2012-10-31 10:56:53,329 - __init__.py[DEBUG]: found data sourceDataSourceEc2
```
　　日志显示找不到meadata source 169.254.169.254，这是一个内部的local source
昨晚在网上查找了大量资料，发现有关cloud-init的资料比较少，最后在rackspace的网站上发现了一份编辑镜像的代码，终于找到了方法。
　　按照OpenStack创建ubuntu镜像的官方文档，我们在安装系统后仅仅需要
```Bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install openssh-server cloud-init
#Remove the network persistence rules from /etc/udev/rules.das their presence will result in
#the network interface in the instance coming up as an interfaceother than eth0.
sudo rm -rf /etc/udev/rules.d/70-persistent-net.rules
```
　　然后实际上cloud-init包的安装是需要配置的，配置方法为
```Bash
echo "cloud-init cloud-init/datasources string NoCloud, OVF,Ec2" >
/tmp/debconf-selections
/usr/bin/debconf-set-selections /tmp/debconf-selections
rm -f /tmp/debconf-selections
apt-get -y install cloud-init
```
　　debconf-set-selections命令可以在包安装的时候对包进行必要的配置，配置参数NoCloud表示instance运行在一个简单的没有metadata服务的系统上，boot是后连接系统的local metadata sourcehttp://169.254.169.254:80。进行上述配置后，重新制作镜像，上述问题经试验一切正常。
***
　　稍后可能在研究下CentOS和Fedora上的解决方法供大家参考。 
