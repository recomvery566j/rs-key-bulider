

# RS-Key 跨库自动化构建环境

## 仓库用途说明

本仓库专用于自动化构建 RS-Key 开源安全密钥的定制化固件。通过配置 GitHub Actions 跨库拉取上游源码，在 Nix 隔离环境中注入特定的硬件标识环境变量，从而输出能够被官方 Yubico 管理工具（如 Yubico Authenticator 和 YubiKey Manager）原生识别的 `.uf2` 镜像文件。

## 核心编译特征

本次构建锁定的具体目标分支及固件特性如下：

* **上游源**: `TheMaxMur/RS-Key` 主分支。
* **构建目标**: `firmware-strong-pin-pqc`。
* **硬件基线**: 适用于默认 4MB Flash 容量且无物理显示屏的 RP2350 安全硬件。
* **安全策略**: 强制开启强 PIN 码验证策略。
* **密码学支持**: 启用后量子密码学 (Post-Quantum Cryptography, PQC) 模块集成。
* **协议层注入**: 编译期强制注入 `VIDPID=Yubikey5` 环境变量，激活专有 APDU 协议栈与完整硬件描述符模拟。

## 工作流架构

本仓库的 CI/CD 工作流脱离了与上游源码的物理绑定，采用以下机制运行：

1. **源码隔离**: 工作流文件独立维护，规避常规代码同步操作导致的文件覆盖丢失问题。
2. **沙盒穿透**: Nix 构建系统默认启用严格沙盒，本流程通过修改编译指令显式开放外部环境读取权限，保障底层构建脚本成功捕获应用层协议切换变量。
3. **闭环输出**: 编译生成的未签名镜像文件将作为 GitHub Artifact 自动打包暂存，保留周期为 7 天。

## 本地签名与部署指南

为防范供应链投毒，本流水线仅执行编译，输出产物为未签名状态。最终的安全信任根必须由部署者在本地离线建立。

**1. 固件签名封装**
下载编译产物后，使用本地冷备份的私钥 (`private_key.pem`) 对镜像执行密码学签名及防降级版本号 (SVN) 配置：

```bash
picotool.exe seal --verbose --sign --major 1 --minor 0 rs-key-yubikey5-strong-pin-pqc.uf2 rs-key-signed.uf2 private_key.pem

```

**2. 底层硬件刷写**
确保 RP2350 设备处于 BOOTSEL 物理引导模式，且系统已配置合规的通信驱动（WinUSB），执行固件加载：

```bash
picotool.exe load -v -x rs-key-signed.uf2

```

**3. 调试通道熔断**
固件刷写并验证运行正常后，通过 OTP 指令在硅片层面物理切断 SWD 调试引脚，完成最高安全级别的硬件锁定：

```bash
picotool.exe otp set OTP_DATA_CRIT1.DEBUG_DISABLE 1 -f

```
