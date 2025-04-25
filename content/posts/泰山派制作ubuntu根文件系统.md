+++
date = '2024-03-18T17:14:12+08:00'
draft = false
title = '泰山派制作ubuntu根文件系统'
+++
创建临时文件夹解压根文件系统

```shellscript
mkdir temp
sudo tar -xpf ubuntu-base-22.04.4-base-arm64.tar.gz -C temp/
```

准备网络

```shellscript
sudo cp -b /etc/resolv.conf temp/etc/resolv.conf
```

准备qemu

```shellscript
sudo cp /usr/bin/qemu-aarch64-static temp/usr/bin/
```

换源

```shellscript
vim sources.list
```

```shellscript
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy-backports main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy-proposed main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy-security main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy-updates main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy-backports main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy-proposed main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy-security main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ jammy-updates main multiverse restricted universe
```



```shellscript
sudo cp sources.list temp/etc/apt/sources.list
```

进入根文件系统进行操作

```shellscript
vim rootfs-mount.sh
```

```shellscript
#!/bin/bash

function mnt() {
    echo "MOUNTING"
    sudo mount -t proc /proc ${2}/proc
    sudo mount -t sysfs /sys ${2}/sys
    sudo mount -o bind /dev ${2}/dev
    sudo mount -o bind /dev/pts ${2}/dev/pts

    sudo chroot ${2}
}

function umnt() {
    echo "UNMOUNTING"
    sudo umount ${2}/proc
    sudo umount ${2}/sys
    sudo umount ${2}/dev
    sudo umount ${2}/dev/pts
}


if [ "$1" == "-m" ] && [ -n "$2" ] ;
then
    mnt $1 $2
elif [ "$1" == "-u" ] && [ -n "$2" ];
then
    umnt $1 $2
else
    echo ""
    echo "Either 1'st, 2'nd or both parameters were missing"
    echo ""
    echo "1'st parameter can be one of these: -m(mount) OR -u(umount)"
    echo "2'nd parameter is the full path of rootfs directory"
    echo ""
    echo "For example: ch-mount.sh -m /media/sdcard"
    echo ""
    echo 1st parameter : ${1}
    echo 2nd parameter : ${2}
fi

```

```shellscript
sudo chmod +x rootfs-mount.sh
```

```shellscript
sudo bash rootfs-mount.sh -m temp
```

```shellscript
sudo bash rootfs-mount.sh -u temp
```

chroot配置

```shellscript
export LC_ALL=C.UTF-8
apt update
apt upgrade
```

```shellscript
apt install dialog apt-utils perl language-pack-en-base \
 sudo vim ssh network-manager wpasupplicant ifupdown wireless-tools\
 net-tools iputils-ping rsyslog bash-completion
```

```shellscript
vim /etc/resolv.conf
```

```shellscript
nameserver 8.8.8.8
nameserver 114.114.114.114
```

添加用户

```shellscript
useradd -s '/bin/bash' -m -G adm,sudo,video user
#设置密码
passwd user
```

清理

```shellscript
apt clean
```

```shellscript
#退出chroot
exit
```

```shellscript
sudo bash rootfs-mount.sh -u temp
```

压缩

```shellscript
1、tar
解包：tar xvf FileName.tar
打包：tar cvf FileName.tar DirName
（注：tar是打包，不是压缩！）

2、.gz
解压1：gunzip FileName.gz
解压2：gzip -d FileName.gz
压缩：gzip FileName

3、.tar.gz 和 .tgz
解压：tar zxvf FileName.tar.gz
压缩：tar zcvf FileName.tar.gz DirName

4、.bz2
解压1：bzip2 -d FileName.bz2
解压2：bunzip2 FileName.bz2
压缩： bzip2 -z FileName

5、.tar.bz2
解压：tar jxvf FileName.tar.bz2
压缩：tar jcvf FileName.tar.bz2 DirName

6、.bz
解压1：bzip2 -d FileName.bz
解压2：bunzip2 FileName.bz
压缩：未知

7、.tar.bz
解压：tar jxvf FileName.tar.bz
压缩：未知

8、.Z
解压：uncompress FileName.Z
压缩：compress FileName

9、.tar.Z
解压：tar Zxvf FileName.tar.Z
压缩：tar Zcvf FileName.tar.Z DirName

10、.zip
解压：unzip FileName.zip
压缩：zip FileName.zip DirName

11、.rar
解压：rar x FileName.rar
压缩：rar a FileName.rar DirName
```

打包

```shellscript
mkdir rootfs
dd if=/dev/zero of=linuxroot.img bs=1M count=6000
mkfs.ext4 linuxroot.img
sudo mount linuxroot.img rootfs/
sudo cp -rfp ubuntu/*  rootfs/
sudo umount rootfs/
e2fsck -p -f linuxroot.img
resize2fs  -M linuxroot.img
```
