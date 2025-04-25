+++
date = '2025-04-23T19:32:29+08:00'
draft = false
title = 'linux下压缩解压命令'
+++

```
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
