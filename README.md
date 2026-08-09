# RS-Key 跨库自动化构建环境

## 仓库用途说明

本仓库专用于自动化构建 RS-Key 开源安全密钥的定制化固件。通过配置 GitHub Actions 跨库拉取上游源码，在 Nix 隔离环境中注入特定的硬件标识环境变量，从而输出能够被官方 Yubico 管理工具原生识别的 `.uf2` 镜像文件。

## 核心编译特征

本次构建锁定的具体目标分支及固件特性如下：

* **上游源**: `TheMaxMur/RS-Key` 主分支。
* **构建目标**: `firmware-strong-pin-pqc`。
* **硬件基线**: 适用于默认 4MB Flash 容量且无物理显示屏的 RP2350 安全硬件。
* **安全策略**: 强制开启强 PIN 码验证策略。
* **密码学支持**: 启用后量子密码学 (PQC) 模块集成。
* **协议层注入**: 编译期强制注入 `VIDPID=Yubikey5` 环境变量，激活专有 APDU 协议栈与完整硬件描述符模拟。

## 工作流架构

本仓库的 CI/CD 工作流脱离了与上游源码的物理绑定，采用以下机制运行：

1. **源码隔离**: 工作流文件独立维护，规避常规代码同步操作导致的文件覆盖丢失问题。
2. **沙盒穿透**: Nix 构建系统默认启用严格沙盒，本流程通过修改编译指令显式开放外部环境读取权限，保障底层构建脚本成功捕获应用层协议切换变量。
3. **闭环输出**: 编译生成的未签名镜像文件将作为 GitHub Artifact 自动打包暂存，保留周期为 7 天。

## 首次部署与硬件锁定指南

全新硬件在首次刷入固件前，必须在本地离线建立安全信任根并物理锁定硬件防线。请确保设备处于 BOOTSEL 引导模式，并已通过 Zadig 替换为 WinUSB 驱动。

**1. 生成离线签名私钥**
使用 OpenSSL 生成用于硬件鉴权的 ECDSA 私钥。此私钥代表对硬件的最高控制权，请务必进行安全的离线冷备份：

```bash
openssl ecparam -name secp256k1 -genkey -noout -out private_key.pem

```

**2. 初始签名与元数据封装**
对编译所得的未签名固件进行密码学签名，并同步生成包含公钥哈希的 OTP 配置摘要文件 (`otp_config.json`)：

```bash
picotool.exe seal --verbose --sign --major 1 --minor 0 rs-key-yubikey5-strong-pin-pqc.uf2 rs-key-signed.uf2 private_key.pem otp_config.json

```

**3. 固化公钥与激活安全启动**
将公钥哈希物理写入芯片的 OTP 区域，并激活底层 BootROM 的安全启动校验机制：

```bash
picotool.exe otp load otp_config.json -f
picotool.exe otp set OTP_DATA_CRIT1.SECURE_BOOT_ENABLE 1 -f

```

**4. 熔断物理调试通道**
为防范侧信道攻击与内存提取，在物理层面彻底切断 SWD 调试引脚：

```bash
picotool.exe otp set OTP_DATA_CRIT1.DEBUG_DISABLE 1 -f

```

**5. 刷入受信任固件**
将完成签名的固件加载至主存储区，设备将自动重启并进入应用态：

```bash
picotool.exe load -v -x rs-key-signed.uf2

```

## 日常固件更新指南

硬件底层信任根固化后，后续的固件升级流程无需再次修改 OTP，仅需执行签名与刷入操作。

**1. 常规固件签名**
获取最新编译的未签名固件，使用备份的私钥进行签名。无需生成 OTP 配置文件：

```bash
picotool.exe seal --verbose --sign --major 1 --minor 0 rs-key-yubikey5-strong-pin-pqc.uf2 rs-key-signed.uf2 private_key.pem

```

*注：若需防范旧版固件降级攻击，请在签名时手动递增 `--major` 或 `--minor` 参数。*

**2. 刷入新版固件**
确保设备进入 BOOTSEL 模式，执行固件加载：

```bash
picotool.exe load -v -x rs-key-signed.uf2

```
