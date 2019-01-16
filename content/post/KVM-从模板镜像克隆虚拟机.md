---
title: KVM-从模板镜像克隆虚拟机
date: 2017-09-04 17:18:00
---
### 1.准备工作
克隆虚拟机你需要以下文件:
 - 一个定义文件（用于描述内存大小，CPUs的个数等等）
 - 一个镜像模板（以安装好操作系统）


### 2.创建预定义文件和镜像模板
定义文件为一个xml格式的文件，模板是一个虚拟机disk image.通过以下方法创建:

- 创建一个基本的虚拟机并安装好操作系统
- 关闭该虚拟机
```bash
virsh shutdown basevm
```
- 导出XML格式文件，并复制基础镜像为template.qcow2
```
virsh dumpxml basevm > /var/lib/libvirt/images/template.xml
cp /var/lib/libvirt/images/basevm.qcow2 /var/lib/libvirt/images/template.qcow2
```
- 修改template.xml文件中镜像文件的指定位置
```
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none'/>
  <source file='/var/lib/libvirt/images/template.qcow2'/>
  <target dev='vda' bus='virtio'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
</disk>
```
- 通过 virt-sysprep 命令处理template.qcow2
这将重置镜像并移除SSH kesy, 为网卡创建新的MAC地址，修改udev persistent net rules, 清除log文件等
```
virt-sysprep -a /var/lib/libvirt/images/template.qcow2
```
- 【可选】如果你不再需要basevm，删除它。
```
virsh undefine basevm
rm /var/lib/libvirt/images/basevm.qcow2
```
### 3.从模板克隆新的虚拟机
- 通过template.xml, template.qcow2 创建虚拟机
```
virt-clone --connect qemu:///system                    \
  --original-xml /var/lib/libvirt/images/template.xml  \
  --name newvm                                         \
  --file /var/lib/libvirt/images/newvm.qcow2
```