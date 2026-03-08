# Hydra Redroid Image Builder

从 AOSP 源码编译自定义 Redroid Docker 镜像，一次性内置所有组件，容器创建即开箱可用。

## 镜像内置组件

| 组件 | 说明 |
|------|------|
| Android 14 (x86_64) | 基于 AOSP `android-14.0.0_r1` + Redroid 补丁 |
| GApps | MindTheGapps，提供 Google Play Store / Play Services |
| libndk_translation | ARM → x86 指令转译层，支持运行 ARM 应用 |
| Mihomo | Clash Meta 代理内核，开机自启服务 |
| WhatsApp | 系统应用预装（XAPK split APKs） |
| TikTok | 系统应用预装（XAPK split APKs） |
| 设备伪装 | Pixel 7 指纹，降低模拟器检测风险 |

## 环境要求

- **磁盘**：≥ 250GB 可用空间（源码 ~100GB + 编译产物 ~100GB + 余量）
- **内存**：≥ 32GB RAM（编译时推荐 64GB+）
- **CPU**：核心数越多越好，直接影响编译时间
- **Docker**：已安装 Docker Engine（需要 BuildKit 支持）
- **网络**：需要访问 AOSP 仓库和 GitHub

## 目录结构

```
redroid-builder/
├── Makefile                         # 编译入口，所有操作通过 make 执行
├── Dockerfile.builder               # AOSP 编译环境镜像（Ubuntu 20.04 + 清华源）
├── scripts/
│   ├── 01-sync-source.sh            # 下载 AOSP 源码 + Redroid 补丁
│   ├── 02-setup-gapps.sh            # 配置 MindTheGapps
│   ├── 03-apply-customizations.sh   # 注入全部自定义组件
│   ├── 04-build-aosp.sh             # 编译 AOSP
│   ├── 05-package-image.sh          # 打包为 Docker 镜像
│   └── 06-verify.sh                 # 自动化验证
├── overlay/
│   ├── system/etc/init/mihomo.rc    # Mihomo init 服务定义
│   ├── system/etc/mihomo/config.yaml# Mihomo 最小默认配置
│   └── build.prop.append            # 属性追加（libndk + 设备伪装）
├── manifests/
│   └── mindthegapps.xml             # GApps 仓库 manifest
└── configs/
    └── redroid_device.mk.patch      # device.mk 修改说明文档
```

XAPK 文件放在项目根目录 `hydra/apks/`：

```
hydra/apks/
├── WhatsApp_xxx.xapk
└── TikTok_xxx.xapk
```

## 快速开始

### 1. 准备 XAPK

从 APKPure 等渠道下载 WhatsApp 和 TikTok 的 XAPK 文件，放到 `hydra/apks/` 目录：

```bash
ls ../apks/
# WhatsApp_2.26.8.72_APKPure.xapk
# TikTok_44.2.3_APKPure.xapk
```

### 2. 一键编译

```bash
cd redroid-builder
make all
```

这将依次执行：`setup` → `sync` → `gapps` → `customize` → `build` → `package`

产出镜像：`hydra/redroid:14.0.0-custom`

### 3. 验证

```bash
make verify
```

### 4. 运行

```bash
docker run -itd --name phone1 --privileged \
  -v /dev/binderfs:/dev/binderfs \
  -v phone1-data:/data \
  -p 5555:5555 \
  hydra/redroid:14.0.0-custom
```

## 分步执行

如果网络不稳定或想逐步调试，可以分步执行每个阶段：

```bash
# 检查磁盘空间
make check-space

# 检查 XAPK 文件
make check-apks

# Step 1: 构建编译环境 Docker 镜像（约 5 分钟）
make setup

# Step 2: 下载 AOSP 源码（约 100GB，几小时）
make sync

# Step 3: 配置 GApps
make gapps

# Step 4: 注入自定义组件（libndk、Mihomo、APK、设备伪装）
make customize

# Step 5: 编译 AOSP（2-4 小时）
make build

# Step 6: 打包 Docker 镜像
make package
```

## Make Targets 一览

| Target | 说明 |
|--------|------|
| `make help` | 显示所有可用 target |
| `make setup` | 构建 AOSP 编译环境 Docker 镜像 |
| `make sync` | 下载 AOSP 源码 + Redroid 补丁 |
| `make gapps` | 配置 MindTheGapps 集成 |
| `make customize` | 注入 libndk、Mihomo、预装应用、设备伪装 |
| `make build` | 编译 AOSP（耗时最长） |
| `make package` | 将编译产物打包为 Docker 镜像 |
| `make all` | 完整编译流水线（上述全部） |
| `make verify` | 自动化验证镜像功能 |
| `make clean` | 清理编译产物（保留源码） |
| `make clean-all` | 删除所有数据（包括源码，需确认） |
| `make check-space` | 检查磁盘空间 |
| `make check-apks` | 检查 XAPK 文件是否就位 |

## 编译流水线详解

### Step 1: setup — 编译环境

构建一个基于 Ubuntu 20.04 的 Docker 镜像，包含 AOSP 编译所需的全部依赖：

- 使用清华 TUNA 镜像源加速下载
- 使用 `--network=host` 解决 Docker BuildKit 的 DNS 问题
- 映射宿主机 UID/GID，编译产物权限正确
- 内置 `repo` 工具、OpenJDK 11、ccache

### Step 2: sync — 下载源码

在 Docker 容器内执行 `repo init` + `repo sync`：

- AOSP 基线：`android-14.0.0_r1`
- Redroid 补丁：`remote-android/local_manifests` 分支 `14.0.0`
- 使用 `--depth=1` 浅克隆，减少下载量
- 源码存放在 `~/redroid-src`（宿主机），通过 volume 挂载到容器 `/src`
- 支持断点续传：再次运行 `make sync` 会跳过已完成的部分

### Step 3: gapps — Google 服务

- 将 `manifests/mindthegapps.xml` 注入 `.repo/local_manifests/`
- 执行 `repo sync` 拉取 MindTheGapps vendor 仓库
- 修改 `device/redroid/redroid/device.mk`，添加 GApps inherit

### Step 4: customize — 自定义组件

在一个脚本中完成 6 项修改：

**1) libndk_translation**
- 下载 Android 14 预编译包
- 创建 vendor makefile 配置 ARM ABI 列表和 native bridge 属性
- 修改 device.mk inherit

**2) Mihomo 代理内核**
- 下载 `mihomo-linux-amd64` 二进制
- 创建 `Android.mk`，通过 AOSP 构建系统安装到 `/system/bin/mihomo`
- 复制 init service 和默认配置

**3) 预装应用（XAPK 处理）**
- 从 `hydra/apks/` 读取 XAPK 文件（ZIP 格式）
- 解压出所有 split APK（base + config.arm64_v8a + config.xxhdpi + 语言包等）
- 暂存到 `$SRC_DIR/.hydra-staged-apps/`
- 在 Step 6 打包时注入到 `/system/priv-app/` 目录

> **为什么不用 AOSP BUILD_PREBUILT？**
> XAPK 包含 split APKs（一个 base APK + 多个 config split APK），AOSP 的 `BUILD_PREBUILT` 只支持单个 APK 文件。
> 我们改为在打包阶段直接将 split APKs 放入 `/system/priv-app/AppName/`，Android PackageManager 启动时会自动扫描和识别。

**4) 设备指纹伪装**
- 伪装为 Google Pixel 7 (panther)
- 包括 build fingerprint、brand、model、security patch 等

**5) 默认显示属性**
- LCD 密度 320dpi

**6) SELinux 策略**
- 为 Mihomo 添加网络权限（tun socket、net_admin、net_raw）
- 使用 userdebug 自带的 `su` context，不依赖 Magisk

### Step 5: build — 编译

```bash
source build/envsetup.sh
lunch redroid_x86_64-userdebug
m -j$(nproc)
```

- 使用 ccache 加速重复编译（缓存上限 50GB）
- 编译日志保存到 `$SRC_DIR/build.log`
- 产出 `system.img` + `vendor.img`

### Step 6: package — 打包镜像

此脚本在宿主机上运行（需要 sudo）：

1. 挂载 `system.img` 和 `vendor.img`
2. 合并为 rootfs
3. 将暂存的 XAPK split APKs 注入 `/system/priv-app/`
4. 创建 tar 归档（保留 xattr）
5. `docker import` 为 Docker 镜像，设置 entrypoint 为 `/init`

## 自定义组件详解

### Mihomo 代理服务

**init 服务** (`overlay/system/etc/init/mihomo.rc`)：

- 开机完成后（`sys.boot_completed=1`）检查配置文件是否存在
- 仅在 `/data/mihomo/config.yaml` 存在时启动
- 以 root 身份运行，拥有 `net_admin` / `net_raw` 权限
- SELinux context: `u:r:su:s0`

**配置下发**：

配置文件由 Hydra 后端通过 ADB push 下发：

```bash
adb push config.yaml /data/mihomo/config.yaml
adb shell setprop mihomo.restart 1    # 触发重启
```

**默认配置** (`overlay/system/etc/mihomo/config.yaml`)：

作为参考模板存在于 `/system/etc/mihomo/`，实际运行使用 `/data/mihomo/config.yaml`。

### 预装应用

WhatsApp 和 TikTok 以 XAPK 格式（split APKs）预装到 `/system/priv-app/`：

```
/system/priv-app/
├── WhatsApp/
│   ├── WhatsApp.apk          # base APK
│   ├── config.arm64_v8a.apk  # native 库
│   └── config.xxhdpi.apk     # 资源
└── TikTok/
    ├── TikTok.apk            # base APK
    ├── config.arm64_v8a.apk  # native 库
    ├── config.xxhdpi.apk     # 资源
    ├── config.en.apk         # 语言包
    └── ...                   # 其他 split APKs
```

更新应用版本：替换 `hydra/apks/` 中的 XAPK 文件，重新执行 `make customize && make build && make package`。

### 设备伪装

| 属性 | 值 |
|------|------|
| `ro.product.model` | Pixel 7 |
| `ro.product.brand` | google |
| `ro.product.device` | panther |
| `ro.build.fingerprint` | `google/panther/panther:14/AP2A.240805.005/...` |
| `ro.build.version.security_patch` | 2024-08-05 |

## 验证清单

`make verify` 自动检查以下项目：

| 检查项 | 预期结果 |
|--------|----------|
| 容器启动 | `sys.boot_completed = 1` |
| Google Play Store | `com.android.vending` 已安装 |
| Google Play Services | `com.google.android.gms` 已安装 |
| Native bridge | `ro.dalvik.vm.native.bridge = libndk_translation.so` |
| ARM ABI 支持 | `arm64-v8a` 在 abilist 中 |
| Native bridge 启用 | `ro.enable.native.bridge.exec = 1` |
| WhatsApp | `com.whatsapp` 已安装 |
| TikTok | `com.zhiliaoapp.musically` 已安装 |
| Mihomo 二进制 | `/system/bin/mihomo` 存在 |
| Mihomo 进程 | 进程运行中（需有配置文件） |
| 设备型号 | `Pixel 7` |
| 设备指纹 | 包含 `google/panther` |

## 可调参数

在 Makefile 头部可以修改以下变量：

```makefile
ANDROID_VERSION    := 14.0.0          # Android 版本号
ANDROID_BRANCH     := android-14.0.0_r1  # AOSP 分支
REDROID_BRANCH     := 14.0.0          # Redroid local_manifests 分支
IMAGE_NAME         := hydra/redroid   # Docker 镜像名
IMAGE_TAG          := 14.0.0-custom   # Docker 镜像 tag
SRC_DIR            := $(HOME)/redroid-src  # 源码存放路径
MIHOMO_VERSION     := v1.19.0         # Mihomo 版本
```

## 常见问题

### 编译失败：磁盘空间不足

AOSP 编译需要 ~250GB。检查空间：

```bash
make check-space
```

### repo sync 中断

直接重新运行 `make sync`，会自动续传。

### 编译环境 Docker 镜像构建失败（DNS 问题）

Makefile 已配置 `--network=host` 解决 BuildKit DNS 隔离问题。如果仍失败，检查宿主机 DNS 是否正常：

```bash
ping mirrors.tuna.tsinghua.edu.cn
```

### 更换 XAPK 版本

替换 `hydra/apks/` 中的文件，然后：

```bash
make customize    # 重新解压 XAPK
make build        # 重新编译（ccache 加速，大部分跳过）
make package      # 重新打包
```

### 清理重来

```bash
make clean        # 只清编译产物，保留源码
make clean-all    # 全部删除（需确认）
```
