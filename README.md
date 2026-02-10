# botdrop_phonectl

让 BotDrop App 中的 AI（openclaw）突破沙盒限制，直接控制 Android 手机。

## 背景

BotDrop 是一个运行在 Android 上的 AI 应用（基于 Termux），但其 AI 被限制在沙盒内，无法操作手机界面。MIUI 进一步封锁了 ADB shell 的 `input` 命令（`INJECT_EVENTS` 权限被剥夺），使得常规触摸注入方案全部失效。

本项目通过 Linux 内核的 `/dev/uinput` 接口，创建一个虚拟触摸屏设备，绕过 Android Framework 层的权限检查，实现从 BotDrop 内部对手机的完全控制。

## 工作原理

```
BotDrop (终端) → adb shell (localhost:5555) → uinput_touch → /dev/uinput → 内核触摸事件
```

关键技术点：
- 使用 `INPUT_PROP_DIRECT` 让 Android 将虚拟设备识别为 **直接触摸屏**（`SOURCE_TOUCHSCREEN`），而非鼠标
- 使用 `BUS_VIRTUAL` 避免被标记为外部设备（EXTERNAL）
- 支持 MT protocol B 多点触控协议
- ADB 通过 TCP (`localhost:5555`) 回环连接，无需网络

## 文件结构

```
├── README.md           # 本文件
├── uinput_touch.c      # 虚拟触摸屏注入工具（C 源码）
├── phonectl.sh         # 手机控制脚本（部署到 BotDrop）
├── instruction.md      # openclaw 使用说明
└── setup.md            # 完整配置指南
```

## 快速开始

### 0. 准备
```bash
brew install hudochenkov/sshpass/sshpass
brew install scrcpy 
brew install --cask android-commandlinetools 
```

### 1. 编译 uinput_touch

```bash
# 需要 Android NDK
NDK_CC=/opt/homebrew/share/android-commandlinetools/ndk/27.2.12479018/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android30-clang

$NDK_CC -static -O2 -o uinput_touch uinput_touch.c
```

### 2. 部署到手机

```bash
# 推送触摸工具
adb push uinput_touch /data/local/tmp/uinput_touch
adb shell chmod 755 /data/local/tmp/uinput_touch

# 开启 ADB TCP（供 BotDrop 内部连接）
adb tcpip 5555
```

### 3. 安装到 BotDrop
先替换以下脚本中的 <username> (可以在botdrop终端通过whoami获取) <password> <yourip> 都可以在botdrop 页面获取。
```bash
# 通过 SSH 复制控制脚本
sshpass -p '<password>' scp -P 8022 phonectl.sh <username>@<phone_ip>:/data/data/app.botdrop/files/usr/bin/phonectl
```

> 详细步骤见 [setup.md](setup.md)

## 使用示例

```bash
# 点击屏幕坐标
phonectl tap 540 1170

# 滑动
phonectl swipe 540 1600 540 800 400

# 返回键
phonectl back

# 截图
phonectl screenshot

# 读取屏幕文字
phonectl uidump_text

# 找到文字并点击
phonectl tap_text "设置"

# 打开相机并拍照
phonectl launch_pkg com.android.camera
sleep 3
phonectl tap 540 2000
```

> 完整命令列表见 [instruction.md](instruction.md)

## uinput_touch 支持的操作

| 命令 | 说明 |
|------|------|
| `tap <x> <y>` | 点击 |
| `longpress <x> <y> [ms]` | 长按（默认 1000ms） |
| `swipe <x1> <y1> <x2> <y2> [ms]` | 滑动（默认 300ms） |
| `key <name>` | 按键（home/back/power/volup/voldown/enter/menu） |

## 适用环境

- **测试设备**：小米 MI CC 9
- **系统**：Android 11 / MIUI 12.5.5
- **要求**：`/dev/uinput` 对 shell 用户可写（属于 uhid 组）
- **不需要** root 权限
- **不需要** 开启"USB调试（安全设置）"

## 已知限制

- 每次触摸操作有约 **2 秒延迟**（uinput 设备注册开销）
- `home` 键被 MIUI 拦截，phonectl 改用 `am start` 实现
- `input keyevent` / `input tap` 等标准命令在 MIUI 下仍不可用
- 手机重启后需重新执行 `adb tcpip 5555`

## License

MIT
