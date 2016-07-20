---
title: "编译制作Linux kernel 3.18 rpm包(升级centos6.x虚拟机内核)"
author: opengers
layout: post
permalink: /linux/linux-kernel-make-rpm/
categories: linux
tags:
  - Congress
  - NSF
format: quote
---




#### 
	介绍



openstack平台需要使用各种Linux发行版镜像,其制作方法主要有两种,要么是基于各大Linux发行版ISO光盘手动制作,要么是使用官方提供的制作好镜像进行修改
之前制作的openstack centos6.5模板镜像,其内核为2.6.xx,现需要升级其内核到3.18.x,使用[这里](http://mirrors.neterra.net/elrepo/kernel/el6/x86_64/RPMS/,)提供的rpm包kernel-ml-3.18.3-1.el6.elrepo.x86_64.rpm升级内核遇到了下面几个问题,下面谈谈我的解决过程
<!-- more -->


#### 
	问题





###### 
	问题1.virtio驱动加载问题



原有的一台kvm虚拟机是2.6.xx内核，使用上面提到的rpm包升级虚拟机kernel之后,重启虚拟机出现错误:




    
    FATAL: Module scsi_wait_scan not found.





再进一步测试,就会发现,在物理机上升级内核,一切ok!
原因是KVM虚拟机使用了virtio_blk.ko这个半虚拟化驱动模块来使虚拟机支持scsi设备,而物理机升级时用不到virtio驱动,自然不会有问题,在kernel3.13版本以前,可以使用"blk_init_queue"这个函数加载virtio_blk.ko模块,而在kernel3.13版本之后,这个函数名变为"blk_mq_init_queue", 此函数名位于"/usr/share/dracut/modules.d/90kernel-modules/installkernel"文件中,可以看到,centos6.5系统中的函数名为"blk_init_queue" ![](http://images.cnitblog.com/blog2015/673203/201504/291602590059077.png) centos6系统中使用Dracut这个程序生成内核的initramfs.img, 而Dracut程序使用的是旧函数"blk_init_queue"(installkernel文件中),因此升级3.18.x内核后,Dracut程序生成的initramfs.img无法包含virtio_blk.ko模块,造成虚拟机启动报错 **解决问题的关键在于要确保virtio_blk.ko能够被加载** 如果我们单纯是需要解决升级内核后启动失败问题,那么就没必要编译内核rpm包,直接下载文章开始提到的内核rpm包,然后使用下面的步骤解决启动问题

    
    #安装rpm包
    rpm -ivh kernel-ml-3.18.3-1.el6.elrepo.x86_64.rpm
    
    #添加virtio_blk支持(新建conf文件)
    echo 'add_drivers+="virtio_blk"' >/etc/dracut.conf.d/force-vitio_blk-to-ensure-boot.conf
    #备份initramfs
    cp /boot/initramfs-3.18.3-1.el6.elrepo.x86_64.img /boot/initramfs-3.18.3-1.el6.elrepo.x86_64.img.bak
    #重新生成initramfs
    dracut -f /boot/initramfs-3.18.3-1.el6.elrepo.x86_64.img 3.18.3-1.el6.elrepo.x86_64
    
    #重启系统




###### 
	注:以上步骤可以解决虚拟机启动问题,如果你不需要制作centos6.5(3.18.x kernel)模板镜像,那么就不需要进行后续步骤





###### 
	问题2.云硬盘热拔插问题



解决了虚拟机启动问题,如果需要制作centos6.5(3.18kernel)模板镜像,那上面的方法是不行的,在openstack中使用此模板启动虚拟机后,其云硬盘的动态加载、移除功能无法使用,centos6.5(2.6.xx kernel)是可以动态加载云硬盘的, 检查3.18版本内核的配置文件/boot/config-3.18.3-1.el6.elrepo.x86_64(安装完kernel rpm包,就会生成此文件),其中并没有热拔插功能支持模块的配置项


###### 
	问题3. 挂载ceph文件系统



2.6.xx内核是不支持ceph文件系统挂载,Linux kernel从3.10版本开始支持ceph文件系统挂载,假如我们的模板镜像需要挂载ceph文件系统,那么也需要确保内核包含cephfs支持相关模块


#### 
	**制作centos6.5(3.18 kernel)模板镜像**





##### 
	1.准备一台虚拟机



首先需要有一个centos6.5(2.6.xx kernel)虚拟机,为了使编译出来的内核rpm包适应openstack虚拟机环境,最好使用一台KVM虚拟机,以下步骤都在此虚拟机环境中操作,我们需要在此环境中编译制作一个3.18 kernel的rpm包
进入虚拟机, 为了解决[问题1],需要修改文件/usr/share/dracut/modules.d/90kernel-modules/installkernel

    
    vim /usr/share/dracut/modules.d/90kernel-modules/installkernel
    #第四行中的"blk_init_queue" 替换为"blk_mq_init_queue"




##### 
	2.下载Linux内核源码(3.18)




    
    #安装编译环境
    yum groupinstall "Development Tools"
    yum install ncurses-devel
    
    #下载源码
    wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.18.12.tar.xz
    tar -xf linux-3.18.12.tar.xz -C /usr/src/
    cd /usr/src/linux-3.18.12/




##### 
	3.添加编译模块



我们在系统原有的内核(2.6.xx)配置文件的基础上建立新的编译选项，所以可以复制系统现有的配置文件/boot/config-2.6.32-431.23.3.el6.x86_64到源码目录/usr/src/linux-3.18.12下,再添加我们需要的编译参数来编译3.18.x内核

    
    #复制配置文件到源码解压目录
    cp /boot/config-2.6.32-431.23.3.el6.x86_64 /usr/src/linux-3.18-12/.config
    
    #支持热拔插模块需要的参数
    CONFIG_HOTPLUG_PCI=y
    CONFIG_HOTPLUG_PCI_FAKE=m
    CONFIG_HOTPLUG_PCI_ACPI=y
    CONFIG_HOTPLUG_PCI_ACPI_IBM=m
    
    #支持ceph文件系统挂载需要的参数
    CONFIG_CEPH_LIB=m
    CONFIG_CEPH_FS=m
    CONFIG_CEPH_FSCACHE=y
    CONFIG_CEPH_FS_POSIX_ACL=y




##### 
	4.制作内核rpm包



接下来需要根据上一步骤配置的.config文件编译kernel,生成rpm包

    
    #加载配置文件
    sh -c 'yes "" | make oldconfig'
    
    #制作rpm包
    make rpm
    
    #生成的内核rpm包目录位于/root/rpmbuild/RPMS/x86_64下




##### 
	6.修改grub.conf




    
    default=0    #default为新内核
    timeout=5
    splashimage=(hd0,0)/grub/splash.xpm.gz
    hiddenmenu
    title Red Hat Enterprise Linux Server (3.18.3-1.el6.elrepo.x86_64)
            root (hd0,0)
            kernel /vmlinuz-3.18.3-1.el6.elrepo.x86_64 ...




##### 
	7.重启系统
