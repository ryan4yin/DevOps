>个人笔记，不保证正确

# Arch Linux 安装 QEMU-KVM 环境

本文的目标是搭建一个 QEMU/KVM 学习环境，带 GUI。

## 一、安装 QUEU/KVM

QEMU/KVM 环境需要安装很多的组件，它们各司其职：

1. qemu: 模拟各类输入输出设备（网卡、磁盘、USB端口等）
    - qemu 底层使用 kvm 模拟 CPU 和 RAM，比软件模拟的方式快很多。
1. libvirt: 提供简单且统一的工具和 API，用于管理虚拟机，屏蔽了底层的复杂结构。（支持 qemu-kvm/virtualbox/vmware）
1. ovmf: 为虚拟机启用 UEFI 支持
1. virt-manager: 用于管理虚拟机的 GUI 界面（可以管理远程 kvm 主机）。
2. virt-viewer: 通过 GUI 界面直接与虚拟机交互（可以管理远程 kvm 主机）。
3. dnsmasq vde2 bridge-utils openbsd-netcat: 网络相关组件，提供了以太网虚拟化、网络桥接、NAT网络等虚拟网络功能。
    - dnsmasq 提供了 NAT 虚拟网络的 DHCP 及 DNS 解析功能。
    - vde2: 以太网虚拟化
    - bridge-utils: 顾名思义，提供网络桥接相关的工具。
    - openbsd-netcat: TCP/IP 的瑞士军刀，详见 [/network/network-tools/socat 和 netcat](/network/network-tools/socat%20和%20netcat.md)，这里不清楚是哪个网络组件会用到它。


安装命令：

```shell
# archlinux/manjaro
sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat

# ubuntu,参考了官方文档，但未测试
sudo apt install qemu-kvm libvirt-daemon-system virt-manager virt-viewer virtinst bridge-utils

# centos,参考了官方文档，但未测试
sudo yum groupinstall "Virtualization Host"
sudo yum install virt-manager virt-viewer virt-install 
```

完了还需要安装 ebtables 和 iptables 两个网络组件，这两个工具也是用来做网络虚拟化的。
了解 docker 的应该知道，docker 就是使用 iptables 实现的容器虚拟网络。

最后，安装虚拟机磁盘映像工具 [libguestfs](https://libguestfs.org/):

```shell
# archlinux/manjaro
sudo pacman -S libguestfs

# ubuntu
sudo apt install libguestfs-tools

# centos
sudo yum install libguestfs-tools
```

libguestfs 可用于直接修改/查看虚拟机映像：

1. `virt-df centos.img`: 查看硬盘使用情况
2. `virt-ls centos.img /`: 列出目录文件
3. `virt-copy-out -d domain /etc/passwd /tmp`：在虚拟映像中执行文件复制
4. `virt-list-filesystems /file/xx.img`：查看文件系统信息
5. `virt-list-partitions /file/xx.img`：查看分区信息
6. `guestmount -a /file/xx.qcow2(raw/qcow2都支持) -m /dev/VolGroup/lv_root --rw /mnt`：直接将分区挂载到宿主机
7. `guestfish`: 交互式 shell，可运行上述所有命令。 

另外还有两个最近（1.42+）从 libguestfs 中分离出来的镜像转换工具(这两个工具目前（2020.8.16）没有 arch/manjaro 安装包，只能手动编译。)：

1. `virt-v2v`: 将其他格式的虚拟机(比如 ova) 转换成 kvm 虚拟机。
2. `virt-p2v`: 将一台物理机转换成虚拟机。

libguestfs 的详细说明后面再给出。

## 二、启动 QEMU/KVM

通过 systemd 启动 libvirtd 后台服务：

```shell
sudo systemctl enable libvirtd.service
sudo systemctl start libvirtd.service
```

## 三、让非 root 用户能正常使用 kvm

qumu/kvm 装好后，默认情况下需要 root 权限才能正常使用它。
为了方便使用，首先编辑文件 `/etc/libvirt/libvirtd.conf`:

1. `unix_sock_group = "libvirt"`，取消这一行的注释，使 `libvirt` 用户组能使用 unix 套接字。
1. `unix_sock_rw_perms = "0770"`，取消这一行的注释，使用户能读写 unix 套接字。

然后新建 libvirt 用户组，将当前用户加入该组：

```shell
newgrp libvirt
sudo usermod -aG libvirt $USER
```

最后重启 libvirtd 服务，应该就能正常使用了：

```shell
sudo systemctl restart libvirtd.service
```

## 四、启用嵌套虚拟化

如果你需要在虚拟机中运行虚拟机，那就需要启用内核模块 kvm_intel 实现嵌套虚拟化。

```shell
# 临时启用 kvm_intel 嵌套虚拟化
sudo modprobe -r kvm_intel
sudo modprobe kvm_intel nested=1
# 修改配置，永久启用嵌套虚拟化
echo "options kvm-intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf
```

验证嵌套虚拟化已经启用：

```shell
$ cat /sys/module/kvm_intel/parameters/nested 
Y
```


## 五、创建/导入/导出/克隆虚拟机

qemu/kvm 目前使用的映像格式是 qcow2（QEMU copy-on-write format 2），而 vmware 使用的是 vmdk.
另外 vmware 还支持将虚拟机导出为 ova 文件。

>虽然 virtualbox 也支持 ova，但是测试发现它和 vmware 的 ova 并不通用。

普通虚拟机的创建流程没啥好说的，使用 virt-manager 的 GUI 界面很简单地就能创建好。

### 1. 导入 vmware 镜像

直接从 vmware ova 文件导入 kvm，这种方式转换得到的镜像应该能直接用（网卡需要重新配置）：

```shell
virt-v2v -i ova centos7-test01.ova -o local -os /vmhost/centos7-01  -of qcow2
```

将 vmware 的  vmdk 文件转换成 qcow2 格式，然后再导入 kvm（网卡需要重新配置）：

```shell
# 转换映像格式
qemu-img convert -p -f vmdk -O qcow2 centos7-test01-disk1.vmdk centos7-test01.qcow2
# 查看转换后的映像信息
qemu-img info centos7-test01.qcow2
```

直接转换 vmdk 文件得到的 qcow2 镜像，启会报错，比如「磁盘无法挂载」。
根据 [Importing Virtual Machines and disk images - ProxmoxVE Docs](https://pve.proxmox.com/pve-docs/chapter-qm.html#_importing_virtual_machines_and_disk_images) 文档所言，需要在网上下载安装 MergeIDE.zip 组件，
另外启动虚拟机前，需要将硬盘类型改为 IDE，才能解决这个问题。


### 2. 导入 img 镜像

img 镜像文件，就是所谓的 raw 格式镜像，也被称为裸镜像，IO 速度比 qcow2 快，但是体积大，而且不支持快照等高级特性。
如果不追求 IO 性能的话，建议将它转换成 qcow2 再使用。

```shell
qemu-img convert -f raw -O qcow2 vm01.img vm01.qcow2
```

## 六、在本地使用 cloud-init 进行虚拟机初始化

>还未测试通过

参考 [../ProxmoxVE/README.md](../ProxmoxVE/README.md)，在本机的 KVM 环境中，也可以使用 cloud-init 来初始化虚拟机。

这需要用到一个工具：[cloud-utils](https://github.com/canonical/cloud-utils)

```shell
sudo pacman -S cloud-utils
```

`cloud-utils` 提供 cloud-init 相关的各种实用工具，
其中有一个 `cloud-localds` 命令，可以通过 cloud 配置生成一个非 cloud 的 bootable 磁盘映像，供本地的虚拟机使用。

首先编写 `user-data`:

```yaml
#cloud-config
password: passw0rd
chpasswd: { expire: False }
ssh_pwauth: True
```

```shell
cloud-localds --hostname ubuntu-1 seed.img user-data
```

这样就生成出了一个 seed.img，创建虚拟机时同时需要载入 seed.img 和 cloud image，cloud-image 自身为启动盘，这样就大功告成了。
示例命令如下：

```shell
virt-install \
  --name ubuntu20.04 \
  --memory 2048 \
  --disk ubuntu-server-cloud-amd64.img,device=disk,bus=virtio \
  --disk seed.img,device=cdrom \
  --os-type linux \
  --os-variant ubuntu20.04 \
  --virt-type kvm \
  --graphics none \
  --network network=default,model=virtio \
  --import
```

也可以使用 virt-viewer 的 GUI 界面进行操作。

这样设置完成后，cloud 虚拟机应该就可以启动了，但是初始磁盘应该很小。可以直接手动扩容 img 的大小，cloud-init 在虚拟机启动时就会自动扩容分区：

```shell
qemu-img resize ubuntu-server-cloudimg-amd64.img 30G
```

## 进阶

1. 通过命令行操作 qemu/kvm
2. 使用 ceph/iscsi 等分布式文件系统做虚拟机的存储。

## 参考

- [Complete Installation of KVM, QEMU and Virt Manager on Arch Linux and Manjaro](https://computingforgeeks.com/complete-installation-of-kvmqemu-and-virt-manager-on-arch-linux-and-manjaro/)
- [virtualization-libvirt - ubuntu docs](https://ubuntu.com/server/docs/virtualization-libvirt)
- [RedHat Docs - KVM](https://developers.redhat.com/products/rhel/hello-world#fndtn-kvm)
- [在 QEMU 使用 Ubuntu Cloud Images](https://vrabe.tw/blog/use-ubuntu-cloud-images-with-qemu/)
