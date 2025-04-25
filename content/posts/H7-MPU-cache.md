+++
date = '2024-04-13T21:46:48+08:00'
draft = false
title = 'H7-MPU-cache'
+++
Core读Cache：
- Cache hit:直接从cache中读取数据
- Cache miss:从cache中读取数据失败,两种方式读取：
  - read through: 直接从内存中读出,不使用cache
  - read allocate: 把数据从内存加载到cache中,然后再从cache中读取
Core写Cache：
- Cache hit:
  - Write through: 直接写到内存中并同时放到cache里面
内存和cache同步更新
  - Write back:数据更新时只写入cache,只在数据被替换出cache,被修改的cache数据才会写入内存(写入速度快);
- Cache miss:
  - Write allocate:先把要写的数据载入到cache,对cache写后，更新到内存
  - no Write allocate:直接写入内存,不使用cache
- 软件cache维护:
![alt text](/images/image.png)
- 配置MPU:
![alt text](/images/image-1.png)