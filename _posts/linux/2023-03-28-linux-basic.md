---
layout: post
title: "linux零散的配置和命令"
category: GNU-linux
date: 2023-03-28 08:00:00 +0800
---

## 安装openEuler 22.03 虚拟机

**注意:20.03版本的虚拟机貌似不好启用网卡相关的配置.需要使用openEuler 22.03版本.**

**虚拟网卡的类型需要使用NAT模式,网桥模式一般的笔记本都不支持.**

**注意在虚拟机配置nameserver,在虚拟机配置GATEWAY**

```shell
# /etc/resolv.conf
nameserver 192.168.122.1
```

```shell
# /etc/sysconfig/network-scripts/ifcfg-ens7
GATEWAY=192.168.122.1
```

## 安装CentOS6.0虚拟机

其实网卡已经添加好了，通过`ip addr`能够查看到，需要做的是让它开机自启即可。直接修改`/etc/sysconfig/network-scripts/ifcfg-eth0`文件。

## docker 使用

[docker使用指南](https://yeasy.gitbook.io/docker_practice/)

```shell
docker run -it -v /sys/fs/cgroup:/sys/fs/cgroup:rw -v /docker-run:/run:rw c7-systemd /sbin/init
docker run -it -v /docker-run:/run:rw docker-test /sbin/init
```

```shell
docker import oesystemd.tar.gz oesystemd:1
```

```shell
docker export ID -o oesystemd.tar.gz
```

```shell
docker load -i openeuler-22.03-lts:latest.tar.gz
```

### Dockerfile将systemd作为docket的一号进程

```shell
FROM openeuler-22.03-lts:latest
RUN yum install -y systemd
CMD ["/sbin/init"]
```
`docker build -t docker-systemd .`执行这个命令构建容器镜像。


## 关于锁屏

* 熄屏时间: gsettings set org.gnome.desktop.session idle-delay 1800
* 熄屏后是否锁定: gsettings set org.gnome.desktop.screensaver lock-enabled true
* 熄屏后到锁定的等待时间: gsettings set org.gnome.desktop.screensaver lock-delay 3600

## 关于普通用户的登陆shell

修改/etc/passwd的配置将/bin/sh修改为/bin/bash

## 关于标题栏

debian的标题栏太宽，将标题栏的宽度缩减到24个像素。

```css
/* .config/gtk-3.0/gtk.css */
headerbar {
    min-height: 16px;
    padding-left: 2px; /* same as childrens vertical margins for nicer proportions */
    padding-right: 2px;
    margin: 0px; /* same as headerbar side padding for nicer proportions */
    padding: 0px;
}
```

## NetworkManager

NetworkManager自动根据`/etc/sysconfig/net-scripts/`目录下的配置来配置网卡。需要注意的是，这里的配置文件是用户自己写的，可能有冗余，也可能跟内核实际的设备有差异，因此无法保证这里的配置能够正常生效。

**内核观测到的网卡设备，或者说实际的网卡设备：** 可以通过`ifconfig -a`获取到。

**udev可能会根据udev规则修改网卡的名称，修改后可能导致/etc/sysconfig/net-scripts/目录下的配置因为匹配不到网卡名无法生效。**

**dhclient只是会向dhserver获取申请动态IP。**

**eth0** 名字的网卡是内核的名字。<https://www.cnblogs.com/yinfutao/p/9634350.html>

## debian的intel网卡驱动安装

搜索firmware-iwlwifi下载安装。<https://packages.debian.org/search?keywords=firmware-iwlwifi>

## grub字体太小

修改grub的分辨率：/etc/default/grub: GRUB_GFXMODE=1280x720。执行`update-grub`使配置生效。

## samba

在debian配置samba：

1. 安装：apt install samba
2. 写配置文件：

```shell
# /etc/samba/smb.conf
[share]
   comment = einsler share
   path = /home/einsler
```

3. 添加用户：
    3.1 添加系统用户：useradd USERNAME
    3.2 将系统用户添加到samba：smbpasswd -a USERNAME
    3.3 在samba中启用这个用户：smbpasswd -e USERNAME

4. 测试是否ok：`smbclient //localhost/share -U einsler`。注意这里的share是步骤2中方括号里面配置的值。

**注意点：** 如果客户端测试ok,但是网络连接不上，就要考虑防火墙的问题了！腾讯云官方屏蔽了一些端口，需要我们将端口修改为自定义端口。

修改端口号为8445：

```shell
# /etc/samba/smb.conf
smb ports = 8445
```

## 其他博主的一些优秀案例

<https://qileq.com/article/202201270002/>
