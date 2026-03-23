---
date: '2026-03-23T12:07:07+08:00'
draft: false
title: 'A7z更新系统或内核无法开机的原因和解决方法'
---
## 📌 问题现象

在使用 U-Boot + extlinux 启动 Linux（ARM64）时，出现如下错误：

```
Loading Ramdisk to 480cf000, end 49fff5a1 ... OK
ERROR: reserving fdt memory region failed (addr=48000000 size=1000000)
```

同时对比正常情况：

```
Loading Ramdisk to 491e5000 ... OK
```

可以发现：

* ❌ 异常：initrd 被加载到 `0x480xxxxx`
* ✅ 正常：initrd 位于 `0x49xxxxxx`

---

## 🎯 问题本质

该问题的核心是：

> **U-Boot 自动分配 initrd 地址时，与设备树（DTB）中的 reserved-memory 区域发生冲突**

---

## 🧠 内存冲突分析

### 1️⃣ 设备树中的保留内存

```dts
reserved-memory {
    secmon@48000000 {
        reg = <0x0 0x48000000 0x0 0x01000000>;
        no-map;
    };
};
```

👉 表示：

```
0x48000000 ~ 0x49000000 （16MB）为保留内存
```

---

### 2️⃣ 实际 initrd 加载位置

```
initrd: 0x480cf000 ❌
```

👉 明显落入 reserved-memory 区域 → 冲突

---

## 🔍 根因分析

通过 `printenv` 发现关键变量：

```bash
kernel_addr_r=0x40080000
ramdisk_addr_r=0x4FF00000
bootm_size=0xa000000
```

---

### 🚨 关键问题：`bootm_size`

```
bootm_size = 0x0a000000 = 160MB
```

U-Boot 限制了可用内存范围：

```
可用范围：
0x40080000 ~ 0x4A080000
```

---

### 📉 U-Boot 实际行为

U-Boot 在该范围内：

* 加载 kernel
* 加载 fdt
* 自动放置 initrd（从高地址向下分配）

计算过程类似：

```
ram_top ≈ 0x4A080000
initrd_size ≈ 32MB

→ 0x4A080000 - 0x02000000 ≈ 0x48000000
```

👉 直接落入 reserved-memory ❌

---

### ❗ 为什么 `ramdisk_addr_r` 没生效？

虽然设置了：

```bash
ramdisk_addr_r=0x4FF00000
```

但：

```
0x4FF00000 > 0x4A080000（bootm_size 上限）
```

👉 **超出允许范围，被 U-Boot 强制重新分配**

---

## 📊 结论

| 项目                | 结论              |
| ----------------- | --------------- |
| initrd 地址错误       | ❌ 不是随机          |
| ramdisk_addr_r 无效 | ❌ 被覆盖           |
| DTB 问题            | ❌ 不是根因          |
| 真正原因              | ✅ bootm_size 限制 |

---

## 🚀 解决方案

---

### ✅ 方案一：增大 `bootm_size`（推荐）

```bash
setenv bootm_size 0x20000000
boot
```

👉 推荐值：

| 内存大小  | bootm_size |
| ----- | ---------- |
| 512MB | 0x20000000 |
| 1GB   | 0x40000000 |

---

### 🎯 效果

```
可用内存扩大
→ initrd 可放到 0x4FF00000
→ 避开 0x48000000
→ 启动成功 ✅
```

---

### ✅ 方案二：手动加载（绕过 extlinux）

```bash
setenv kernel_addr_r  0x42000000
setenv ramdisk_addr_r 0x4A000000
setenv fdt_addr_r     0x43000000

load mmc 0:1 ${kernel_addr_r} /boot/vmlinuz
load mmc 0:1 ${ramdisk_addr_r} /boot/initrd.img
load mmc 0:1 ${fdt_addr_r} /boot/devicetree.dtb

booti ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}
```

---

### ⚠️ 方案三：修改 DTB（不推荐优先）

* 修改 `reserved-memory`
* 或删除相关节点

👉 风险：

* 可能影响 secure firmware / 外设

---

### ⚠️ 方案四：修改 U-Boot 源码（进阶）

可修改：

```c
CONFIG_SYS_BOOTMAPSZ
bootm_size 默认值
```

或禁用：

```c
CONFIG_SYS_BOOT_RAMDISK_HIGH
```

---

## 🧪 验证方法

### 查看内存信息：

```bash
bdinfo
```

---

### 查看环境变量：

```bash
printenv bootm_size
```

---

### 观察 initrd 地址：

```
Loading Ramdisk to XXXXXXXX
```

---

## 🏁 总结

> 本问题本质是：**U-Boot 的 bootm 内存限制导致 initrd 被错误分配到 reserved-memory 区域**

---

### ✅ 核心要点

* `ramdisk_addr_r` 不一定生效
* extlinux 会触发自动分配逻辑
* `bootm_size` 决定可用内存范围
* U-Boot 不会自动避开 reserved-memory（部分 BSP）

---

### 🎯 最终建议

```bash
setenv bootm_size 0x20000000
```

👉 一步解决，简单有效 ✅

---

## 💡 一句话总结

> **不是地址设错，而是 U-Boot 不让你用那个地址**

---

如果后续需要做“永久生效”（编译进 U-Boot），可以在此基础上继续优化。
