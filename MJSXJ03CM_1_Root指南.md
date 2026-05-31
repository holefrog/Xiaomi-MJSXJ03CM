#  小米米家小白智能摄像机增强版 [MJSXJ03CM]  — Root 提权与 Telnet 后门指南

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

## 一、设备基本信息

| 项目 | 详情 |
|---|---|
| 型号 | MJSXJ03CM |
| 固件版本 | 3.5.8_20031301_c |
| 芯片平台 | 安霸（Ambarella）— 非全志安凯 |
| Bootloader | `amboot>`（非标准 U-Boot，无通用 bootargs 修改入口） |
| Root 密码 | 空（`/etc/shadow` 第二字段为空） |
| Telnet 端口 | 23 |
| 串口波特率 | 115200 |
| MAC 前缀 | `04:CF:8C` |

---

## 二、Root 提权方式对比

本设备存在两条独立的 Root 获取路径，根据你的情况选择其一：

| 方式 | 前提条件 | 是否一次性 | 风险 |
|---|---|---|---|
| **A. UART 串口注入**（推荐） | 有串口线，能物理接触主板 | 一次性，后续用 Telnet | 低 |
| **B. 虚假 OTA 劫持** | 能控制局域网 DNS/ARP | 一次性，后续用 Telnet | 中 |

---

## 三、方式 A：UART 串口注入（推荐）

### 准备

- USB 转 TTL 串口线，接摄像机主板 UART 焊点
- 运行 Ubuntu/Linux 的电脑

### 步骤 1：建立串口连接

```bash
picocom -b 115200 /dev/ttyUSB0
```

### 步骤 2：进入 amboot 控制台

- 在 picocom 界面中**按住回车键不放**
- 同时给摄像机主板接上 Micro-USB 电源
- 出现 `amboot>` 提示符后松开回车

### 步骤 3：注入启动参数，获取临时 Root Shell

```
boot lnx console=ttyS0,115200 root=ubi0:rootfs ubi.mtd=4 rootfstype=ubifs init=/bin/sh
```

等待出现 `/ #` 提示符。

### 步骤 4：挂载 /proc 并重挂根目录为可写

```bash
mount -t proc proc /proc
mount -o remount,rw /
```

无报错即成功。

### 步骤 5：植入永久 Telnet 后门

```bash
sed -i 's/#ttyS0/ttyS0/g' /etc/inittab
sed -i 's/null::once:\/usr\/sbin\/telnetd/null::once:\/bin\/busybox telnetd -l \/bin\/sh/g' /etc/inittab
sync
```

### 步骤 6：重启验证

拔掉电源重新上电，等蓝灯常亮后：

```bash
telnet <摄像机IP>
# 出现 / # 即成功，后续无需再使用串口线
```

---

## 四、方式 B：虚假 OTA 劫持（无需串口线）

利用系统更新流程（`proc_updatepackage.sh`）的设计缺陷：**对升级包内脚本无签名校验，且以 Root 权限执行**。

### 步骤 1：构造恶意升级包

在 Ubuntu 上创建以下目录结构，然后打包：

```bash
mkdir ota_payload
cat > ota_payload/copy_imi.sh << 'EOF'
#!/bin/sh
# 植入 Telnet 后门
sed -i 's/#ttyS0/ttyS0/g' /etc/inittab
sed -i 's/null::once:\/usr\/sbin\/telnetd/null::once:\/bin\/busybox telnetd -l \/bin\/sh/g' /etc/inittab
sync
EOF
chmod +x ota_payload/copy_imi.sh
tar czf upgrade_imi.tar.gz -C ota_payload .
```

### 步骤 2：在本地起 HTTP 服务

```bash
python3 -m http.server 8080
# upgrade_imi.tar.gz 放在当前目录
```

### 步骤 3：劫持摄像机的升级请求

在路由器或本地用 DNS/ARP 劫持，将摄像机的固件升级域名解析到你的电脑 IP。

摄像机触发更新检查后，会下载并执行你的 `copy_imi.sh`，以 Root 权限完成后门植入。

### 步骤 4：重启后用 Telnet 连接（同方式 A 步骤 6）

---

## 五、日常 Telnet 登录

```bash
telnet 192.168.50.xxx   # IP 从路由器后台查，MAC 前缀 04:CF:8C
# 用户：root
# 密码：直接回车（无密码）
```

---

## 六、RTSP 视频流配置

### 现状说明

摄像机内置 `/usr/local/bin/rtsp_server`（安霸原厂测试程序），但**无法与米家 imiApp 共存**：imiApp 独占 `/dev/iav` 硬件编码器，导致 rtsp_server 拉不到视频数据（返回 500）。

米家 App 使用的是 TUTK P2P 私有协议，不走 RTSP。

### 选项 A：保留米家 App + RTSP（推荐）

目前官方固件下无法直接实现，需等待或移植开源方案（如 `v4l2rtspserver` 内存 Hook 方式）。

### 选项 B：放弃米家 App，纯本地 RTSP

通过 Telnet 登录后修改启动脚本，在开机时关闭 imiApp，将硬件资源完全交给 rtsp_server：

```bash
# 1. 重挂根目录为可写
mount -o remount,rw /

# 2. 修改启动脚本，注释掉 imiApp 启动行，追加 rtsp_server
# 编辑 /usr/imi/start.sh，找到启动 imiApp 的行并注释
# 在文件末尾追加：
echo "/usr/local/bin/rtsp_server &" >> /usr/imi/start.sh
sync
```

重启后测试：

```bash
ffplay -rtsp_transport tcp rtsp://192.168.50.136:554/stream1
```

> 注意路径是 `stream1`，不是 `stream_0` 或 `stream_1`（后者返回 404）。

---

## 七、官方救砖（固件重写）

当设备变砖时使用。**注意：只能同版本重写，无法降级。**

固件下载：[创米官方软件升级中心](https://www.imilab.com/wechat/software_upgrade.html)

### SD 卡准备

- 格式：FAT32，容量 32GB 或以下
- 将 `tfk.bin` 和 `tfr.bin` **直接放入根目录，不改名**

### 刷机步骤

1. 摄像机开机，卡针按住 RESET 键，听到复位提示音 → 黄灯常亮
2. 黄灯常亮后立刻拔掉电源
3. 插入 SD 卡，重新接上电源
4. 等待黄灯常亮（不转头自检）→ 刷写完成 → 红灯闪烁
5. 重新绑定米家 App

---

## 八、已知限制

- **无法降级固件**：3.5.8 使用 x509 非对称签名校验，软件层面无法绕过
- **amboot 封闭**：`setenv` 仅允许修改 SN/IP 等，无法注入 `bootargs`，无法从 SD 卡外部引导
- **Flash 坏块警告**：block 18 存在物理坏块，硬件编程风险极高
- **工厂模式不可用**：`manufacture.txt` 触发的盲测模式会断开 Wi-Fi，不能作为持久化方案
