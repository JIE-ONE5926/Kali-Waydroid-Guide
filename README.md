# Kali Linux + Waydroid 完整安装与配置指南

本指南专为 Kali Linux 环境定制，涵盖虚拟机配置、环境检查、镜像部署、渲染修复及启动流程，旨在解决常见的闪退、黑屏及 `Failed to get service` 报错问题。

## 0. **所需文件清单**

| 文件名                                                      | 类型       | 说明                                                         | 官方来源                                                     |
| ----------------------------------------------------------- | ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `kali-linux-2026.2-installer-amd64.iso`                     | 🖥️ 系统镜像 | Kali Linux 2026.2 版本的官方 64位安装镜像。                  | [Kali Linux 官网下载页](https://www.kali.org/get-kali/#kali-installer-images) |
| `VMware虚拟机文件.rar`                                      | 📦 压缩包   | 已配置好环境的 Kali Linux 虚拟机完整备份包。                 | 自行打包备份                                                 |
| `lineage-20.0-20260403-VANILLA-waydroid_x86_64-system.zip`  | 🤖 安卓系统 | Waydroid 的 LineageOS 20.0 纯净版 (Vanilla) x86_64 系统镜像。 | [Waydroid 官方项目页](https://sourceforge.net/projects/waydroid/) |
| `lineage-20.0-20260403-MAINLINE-waydroid_x86_64-vendor.zip` | ⚙️ 核心组件 | Waydroid 的 LineageOS 20.0 主线版 (Mainline) x86_64 厂商/底层组件包。 | [Waydroid 官方项目页](https://sourceforge.net/projects/waydroid/) |
| `lineage-20.0-20260403-GAPPS-waydroid_x86_64-system.zip`    | 📱 安卓系统 | Waydroid 的 LineageOS 20.0 预装版 (GApps) x86_64 系统镜像。  | [Waydroid 官方项目页](https://sourceforge.net/projects/waydroid/) |

## 1. 虚拟机配置建议

若使用 VMware 或 VirtualBox 运行 Kali，请务必按以下标准配置，否则 Waydroid 无法正常工作。

### 处理器设置

- **处理器数量**：建议设置为 **2**。
- **每个处理器的内核数量**：建议设置为 **2** 或 **4**（总核心数 4-8 核）。
- **虚拟化引擎**：**必须勾选** `虚拟化 Intel VT-x/EPT 或 AMD-V/RVI(V)`。Waydroid 依赖 LXC 容器技术，强依赖硬件虚拟化指令集。

### 内存与存储

- **内存**：至少分配 **8GB**，推荐 **12GB**。
- **存储**：使用 SSD 虚拟磁盘，分配至少 **30GB** 空间。

### 显卡设置

- **3D 图形加速**：必须在虚拟机设置的“显示器”选项卡中勾选 **加速 3D 图形**。
- **显存大小**：建议至少分配 **2GB** 以上。

------

## 2. 前置环境检查

Waydroid 强依赖 Wayland 图形协议，X11 环境下极不稳定。

1. **检查当前会话类型**：

   ```bash
   echo $XDG_SESSION_TYPE
   ```

2. **判断**：

   - 必须输出 `wayland`。
   - 如果输出 `x11`，请在登录界面右下角选择 `Plasma (Wayland)` 或 `GNOME on Wayland` 重新登录。

------

## 3. 添加源并安装核心组件

Kali 基于 Debian Testing/Sid，直接使用官方脚本可能报错，建议手动指定为 `bookworm` (Debian 12) 源。

1. **清理旧配置（如有）**：

   ```bash
   sudo rm -f /etc/apt/sources.list.d/waydroid.list
   ```

2. **添加官方源并安装**：

   ```bash
   curl https://repo.waydro.id | sudo bash -s bookworm
   sudo apt update
   sudo apt install waydroid -y
   ```

3. **安装解压工具**：

   ```bash
   sudo apt install lzip unzip wget -y
   ```

------

## 4. 镜像下载、解压与移动

手动部署镜像是解决 Kali 下自动下载失败、超时及初始化报错的最稳妥方案。

1. **创建必要的目录结构**：

   ```bash
   sudo mkdir -p /usr/share/waydroid-extra/images
   ```

2. **下载镜像**：
   前往 GitHub Waydroid Images 页面，下载对应架构（通常是 x86_64）的 `system.zip` 和 `vendor.zip`，并拖入虚拟机的 `~/Downloads` 目录。

3. **解压并移动文件**：
   在 `~/Downloads` 目录打开终端执行：

   ```bash
   # 1. 解压文件
   unzip lineage-20.0-20260403-GAPPS-waydroid_x86_64-system.zip
   unzip lineage-20.0-20260403-MAINLINE-waydroid_x86_64-vendor.zip
   
   # 2. 将镜像文件移动到系统指定目录
   sudo mv system.img /usr/share/waydroid-extra/images/
   sudo mv vendor.img /usr/share/waydroid-extra/images/
   ```

------

## 5. 初始化与配置

### 执行初始化

```bash
sudo waydroid init
```

> **提示**：由于已手动放置镜像，此命令会直接读取本地文件，速度极快且不会报错。

### 强制软件渲染（关键修复）

为防止虚拟机或特定显卡导致的闪退/黑屏，需修改底层属性配置。

1. **编辑配置文件**：

   ```bash
   sudo nano /var/lib/waydroid/waydroid_base.prop
   ```

2. **添加参数**：
   在文件末尾添加以下两行（确保没有多余空格）：

   ```ini
   ro.hardware.gralloc=default
   ro.hardware.egl=swiftshader
   ```

3. **保存退出**：按 `Ctrl+O` 回车保存，`Ctrl+X` 退出。

------

## 6. 启动容器和桌面

Waydroid 的启动逻辑是“先开引擎，再点火，最后挂挡”，缺一不可。

1. **重启容器服务**：

   ```bash
   sudo systemctl restart waydroid-container
   ```

   > **为什么用 restart 而不是 start？**
   > 虽然首次安装用 `start` 也可以，但 `restart` 能确保刚才修改的渲染配置（SwiftShader）强制生效，并清理可能存在的残留旧进程，是更安全的“防呆”操作。

2. **启动 Android 会话（⚠️ 核心步骤）**：

   ```bash
   waydroid session start
   ```

   > **绝对重点**：执行后终端会滚动日志或停住。**千万不要关闭这个终端窗口，也不要按 `Ctrl+C`！** 让它保持运行，这是 Android 系统的“心跳”。

3. **打开图形界面**：
   **新建一个终端标签页**（快捷键 `Ctrl+Shift+T`），然后输入：

   ```bash
   waydroid show-full-ui
   ```

------

## 7. 应用安装与管理

Waydroid 默认无应用商店，推荐通过命令行管理。

### 安装本地 APK

将 APK 拖入虚拟机后，在终端执行：

```bash
waydroid app install /home/你的用户名/Downloads/应用名.apk
```

### 查看与启动应用

- **查看已安装列表**：

  ```bash
  waydroid app list
  ```

- **通过包名启动**：

  ```bash
  waydroid app launch com.tencent.mm
  ```

> **💡 进阶小贴士**：建议先下载一个“酷安”或“应用宝”的 APK 安装进 Waydroid，之后即可直接在安卓桌面内下载软件，无需反复使用命令行。
