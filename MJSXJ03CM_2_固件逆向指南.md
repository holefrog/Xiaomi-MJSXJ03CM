# MJSXJ03CM — 固件逆向与安全审计指南

> 适用固件：`3.5.8_20031301_c` | 芯片：安霸（Ambarella）
>
> **⚠️ 免责声明 (Disclaimer)**
> 本项目及相关文档仅用于技术研究、学习交流及实现设备的合法互操作性（如接入 Home Assistant）。
> 1. **非官方项目**：本项目与小米公司（Xiaomi Inc.）及其子品牌（米家/Mijia）无任何隶属、赞助或授权关系。
> 2. **风险自负**：本项目内容涉及对硬件固件的分析与修改。执行本仓库描述的操作可能导致您的设备保修失效、功能异常、甚至完全无法启动（变砖）。用户需自行承担所有风险。
> 3. **法律合规**：请确保您对所操作的设备拥有合法所有权，并遵守您所在司法管辖区的相关法律法规。本项目不支持、不鼓励任何非法修改、未经授权的数据获取或其他违法行为。
> 4. **无担保**：作者不对本项目提供任何形式的担保，包括但不限于对适用性、准确性、安全性或完整性的保证。因使用本项目内容导致的任何直接或间接损失，作者不承担任何责任。
> **如果您不同意上述条款，请立即停止使用本项目中的任何内容。**

---

## 一、固件文件概览

官方救砖包含两个文件，分别对应内核与根文件系统：

| 文件 | 内容 | 文件系统格式 |
|---|---|---|
| `tfk.bin` | Linux 内核 + 设备树 + initramfs | gzip 压缩内核，cpio initramfs |
| `tfr.bin` | 根文件系统 | UBI / UBIFS（NAND 专用） |

固件下载：[创米官方软件升级中心](https://www.imilab.com/wechat/software_upgrade.html)

---

## 二、binwalk 初步扫描

```bash
sudo apt install binwalk ubi-utils mtd-utils -y
binwalk tfk.bin
binwalk tfr.bin
```

### tfk.bin 关键特征

| 偏移量 | 内容 |
|---|---|
| `0x8578` | Flattened Device Tree（硬件管脚定义） |
| `0xA7A6D` | **x509 v3 证书**（防回滚签名，公钥固化在 Bootloader） |
| `0x38C18C` | Linux kernel 3.10.1 |
| `0x38DD88` | gzip 压缩内核本体 |
| `0x4D2CD8` | ASCII cpio initramfs |

> x509 证书的存在意味着防回滚依赖非对称加密签名，**修改版本号无法绕过校验**。

### tfr.bin 关键特征

| 偏移量 | 内容 |
|---|---|
| `0x8578` | Flattened Device Tree |
| `0x22100` | **UBI erase count header**（根文件系统起始） |

---

## 三、解包 tfr.bin 根文件系统

### 环境准备

```bash
sudo apt install binwalk ubi-utils mtd-utils -y
```

### 步骤 1：裁剪出 UBI 数据

跳过前 139520 字节的安霸 PTB 头部：

```bash
dd if=tfr.bin of=rootfs.ubi bs=4096 skip=139520 iflag=skip_bytes status=progress
```

### 步骤 2：加载 nandsim 模拟 NAND 闪存

**关键**：必须使用以下精确的 4 字节 ONFI ID 伪装参数，否则内核会回退到错误的默认几何参数（512B 页/16KB 块）导致挂载失败：

```bash
sudo rmmod nandsim 2>/dev/null
sudo modprobe nandsim \
  first_id_byte=0x2c \
  second_id_byte=0xda \
  third_id_byte=0x90 \
  fourth_id_byte=0x95
```

参数含义：Micron 256MB 容量、2048B Page、128KB 擦除块、64B OOB。

### 步骤 3：验证模拟参数（必须确认再继续）

```bash
mtdinfo /dev/mtd0
```

必须看到以下两行才能继续，否则重新执行步骤 2：

```
Minimum input/output unit size: 2048 bytes
Eraseblock size:                131072 bytes, 128.0 KiB
```

### 步骤 4：写入镜像并挂载

```bash
sudo flash_erase /dev/mtd0 0 0
sudo nandwrite -p /dev/mtd0 rootfs.ubi

sudo modprobe ubi
sudo ubiattach /dev/ubi_ctrl -m 0 -O 2048   # -O 2048 强制指定 VID 偏移，必须加

sudo mkdir -p /mnt/camera_root
sudo mount -t ubifs /dev/ubi0_0 /mnt/camera_root
ls -la /mnt/camera_root
```

---

## 四、关键文件审计

### 目录结构速查

| 路径 | 内容 |
|---|---|
| `/etc/shadow` | 用户密码（root 为空密码） |
| `/etc/inittab` | 开机启动配置，官方屏蔽了 ttyS0 和 telnetd |
| `/etc/init.d/rcS` | 主启动脚本 |
| `/usr/imi/start.sh` | 米家业务主启动脚本 |
| `/usr/imi/imiApp` | 米家主程序（独占 /dev/iav 硬件编码器） |
| `/usr/local/bin/rtsp_server` | 安霸原厂 RTSP 测试程序 |

### 密码审计

```bash
cat /mnt/camera_root/etc/shadow
# root::10933:0:99999:7:::
# 第二字段为空 = root 无密码
```

### 启动链审计

```bash
cat /mnt/camera_root/etc/inittab
# 查看哪些服务被官方屏蔽

grep -rn "telnet\|dropbear\|sdcard\|mmcblk" /mnt/camera_root/etc/init.d/
# 排查是否存在 SD 卡触发后门
```

### OTA 更新流程审计（致命漏洞）

```bash
cat /mnt/camera_root/usr/imi/proc_updatepackage.sh
```

该脚本存在以下设计缺陷：

- 对 `upgrade_imi.tar.gz` 包内脚本**无签名校验**
- 以 **Root 权限**直接执行包内的 `copy_imi.sh`
- 利用此漏洞可通过虚假 OTA 劫持实现无串口 Root（详见 Root 指南）

### SD 卡工厂后门审计

```bash
grep -rn "manufacture" /mnt/camera_root/usr/imi/
```

触发文件名为 `manufacture.txt`，放入 SD 卡根目录可激活生产线测试模式。

**注意**：此模式会**断开 Wi-Fi 与米家网络**，不能用于持久化控制。

---

## 五、RTSP 服务分析

### rtsp_server 正确 URL 格式

从 `liblwmedia.so` 依赖库中提取的报错字符串显示，URL 路径格式有严格要求：

```
BAD rtsp url: %s, should be like '%s10.0.0.2/stream_0'
```

实测结果：

| URL | 结果 | 原因 |
|---|---|---|
| `rtsp://IP:554/` | 404 | 无路径 |
| `rtsp://IP:554/stream_0` | 404 | 参数错误（EECode_BadParam） |
| `rtsp://IP:554/stream_1` | 404 | 同上 |
| `rtsp://IP:554/stream1` | **500** | 路径正确，但被 imiApp 抢占资源 |

500 错误的根本原因：imiApp 独占 `/dev/iav` 硬件编码器，rtsp_server 无法获取视频数据，内部报 `EECode_BadState`。

### 冲突原理

```
imiApp（米家主程序）
  └── 独占 /dev/iav（安霸硬件编码器）
  └── 独占位流缓存

rtsp_server（安霸测试程序）
  └── 尝试访问 /dev/iav → 冲突 → 返回 500
```

---

## 六、芯片平台说明

网络上大多数公开资料（fang-hacks 等）记录的是全志安凯（Allwinner Anyka）版本，但通过 UART 跑码确认，**固件 3.5.8 对应的后期批次已更换为安霸（Ambarella）芯片**。

### amboot 控制台能力边界

进入 `amboot>` 后可用的命令及其限制：

| 命令 | 功能 | 限制 |
|---|---|---|
| `show ptb` | 查看分区表 | 只读 |
| `show poc` | 查看启动配置 | 只读 |
| `setenv sn` | 修改序列号 | 有限 |
| `setenv ip/mask` | 修改网络参数 | 有限 |
| `setenv pri_file/lnx_file` | 修改升级文件名 | 有限 |
| `setenv bootargs` | **不支持** | 已被阉割 |
| `boot sd` / 外置内核引导 | **不支持** | 编译时移除 |

Linux 启动参数硬编码在只读固件内，无法通过控制台修改。

### 分区表（来自 `show ptb`）

| 分区 | 编译时间 | 说明 |
|---|---|---|
| bst / bld | 2018 年 | 底层 Bootloader，基本不变 |
| pri / lnx | 2020/03/13 | 主系统与内核，与固件版本锁定 |

---

## 七、硬件破解边界（不建议尝试）

在不放弃 Root 权限的纯硬件路线上，理论上可以：

1. 热风枪拆下主板 8 脚 Flash 芯片
2. 用 CH341A 编程器 + `flashrom` 导出完整 hex 固件
3. 修改后用电烙铁焊回

**风险评估**：Flash 上已知存在物理坏块（block 18），强行擦写极易导致分区地址错位，变砖概率极高，**不建议尝试**。
