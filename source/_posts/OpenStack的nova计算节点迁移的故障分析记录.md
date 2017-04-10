title: OpenStack的nova计算节点迁移的故障分析记录
date: 2013-02-19 11:31:01
tags: [OpenStack]
---
>前些日子一直在忙活将OpenStack的那几台服务器从部门机房搬迁到集团的大机房去，导致了外网ip的变动，出现了许多问题，也是经过了2个不免之夜，终于将问题都解决了，在这里将过程记录一下，和大家分享。

外网ip从原来的10.2.116.14变更为10.2.15.3，回想了一下安装过程，设计到此IP的主要有3个地方，KeyStone的Endpoint，Nova的配置文件nova.conf，以及设置Nova-network的floating_ip。将其一一修改为新IP之后重启，出现错误：`TRACE nova libvirtError: Domain not found: no domain with matching name 'instance-0000005f'`。
查找instance-0000005f这个文件，发现其位置在`/var/lib/nova/instances/instance-0000005f`这个目录下有libvirt.xml这个文件，这个文件在nova-compute启动时会最为虚拟机的参数配置文件，查找发现VNC的配置IP还是原IP`<graphics type='vnc' port='-1' autoport='yes' keymap='en-us' listen='10.2.116.14'/>`，将其修改为新的ip。
重启nova-compute问题依旧，仔细研读出错的log信息，发现错误头为libvirtError想到会不会是libvirt的错误，然后到`/etc/libvirt/qemu`目录下查找，发现目录为空。证明libvirt启动时没有发现配置文件，正如上面的错误所示，`no domain with matching name instace-00xxxx`。研读nova源码发现，nova-compute启动时会根据``/var/lib/novainstances/instance-0000005f/libvirt.xml`去创建libvirt所使用的标准domain文件，估计是缓存或是bug等原因，导致只修改了VNC的IP后还是不能正确生成libvirt的domain文件，所以我们用libvirt的virsh命令手动生成。
```
cd /etc/libvirt/qemu
virsh define instance-0000005f.xml
```
之后手动添加其配置信息
```
<!--
WARNING: THIS IS AN AUTO-GENERATED FILE. CHANGES TO IT ARE LIKELY TO BE
OVERWRITTEN AND LOST. Changes to this xml configuration should be made using:
  virsh edit instance-0000005f
or other application using the libvirt API.
-->

<domain type='kvm'>
  <name>instance-0000005f</name>
  <uuid>e8f73117-1a19-4d3e-a8d5-f846199d3b75</uuid>
  <memory>524288</memory>
  <currentMemory>524288</currentMemory>
  <vcpu>1</vcpu>
  <os>
    <type arch='x86_64' machine='pc-1.0'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
  </features>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <devices>
    <emulator>/usr/bin/kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none'/>
      <source file='/var/lib/nova/instances/instance-0000005f/disk'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </disk>
    <interface type='bridge'>
      <mac address='fa:16:3e:1a:66:d3'/>
      <source bridge='br100'/>
      <model type='virtio'/>
      <filterref filter='nova-instance-instance-0000005f-fa163e1a66d3'>
        <parameter name='DHCPSERVER' value='192.168.4.33'/>
        <parameter name='IP' value='192.168.4.40'/>
      </filterref>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <serial type='file'>
      <source path='/var/lib/nova/instances/instance-0000005f/console.log'/>
      <target port='0'/>
    </serial>
    <serial type='pty'>
      <target port='1'/>
    </serial>
    <console type='file'>
      <source path='/var/lib/nova/instances/instance-0000005f/console.log'/>
      <target type='serial' port='0'/>
    </console>
    <input type='tablet' bus='usb'/>
    <input type='mouse' bus='ps2'/>
    <graphics type='vnc' port='-1' autoport='yes' listen='10.2.15.3' keymap='en-us'>
      <listen type='address' address='10.2.15.3'/>
    </graphics>
    <video>
      <model type='cirrus' vram='9216' heads='1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </memballoon>
  </devices>
</domain>
```
经分析，是修改IP导致了nova-volume服务找不到对应的volume了，但是我们在nova-volume被没有相关的配置文件，几经思考之后决定查看openstack的系统数据库，发现在block_device_mapping和volumes两个表中有跟IP相关的列对应，将其修改为新的IP，之后再次重启nova-compute服务，终于一切正常。

***
2013-02-21 新增
***
上面nova-volume服务出现错误，主要是两个数据表中和服务IP有关联的项目所用的IP都是默认的外网IP端10.2.16.14，而我们服务器搬迁了之后正好更改的就是这个外网地址。我们可以修改nova.conf中增加`--iscsi_ip_address=192.168.3.1`，这样申请的volume就会用这个内网地址写到数据库中，如再遇服务器搬迁修改外网IP等问题，不会对系统内部产生影响。

期间也走了许多弯路，出现问题并不可怕，要冷静客观。主要还是要有逻辑分析能力，能找到解决问题的思路。
