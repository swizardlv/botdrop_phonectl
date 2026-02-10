# openclaw 手机控制工具 — `phonectl`

在 BotDrop 终端里，确保环境变量已设置（`.bashrc` 里已配好），直接使用 `phonectl` 命令：

## 触屏操作
```bash
phonectl tap 668 1423        # 点击坐标
phonectl longpress 540 1170  # 长按（默认1秒）
phonectl swipe 540 1600 540 800 400  # 滑动（起点→终点, 持续ms）
phonectl scroll_up           # 向上翻页
phonectl scroll_down         # 向下翻页
```

## 按键
```bash
phonectl back                # 返回
phonectl home                # 回主屏
phonectl key power           # 电源键
phonectl key volup           # 音量+
phonectl key voldown         # 音量-
phonectl key enter           # 回车
```

## 看屏幕
```bash
phonectl screenshot          # 截图到 /sdcard/screen.png
phonectl uidump_text         # 读取屏幕上所有文字
phonectl uidump              # 完整 UI 树（含坐标 bounds）
phonectl find_text "WLAN"    # 查找文字元素的坐标
phonectl tap_text "WLAN"     # 找到文字并点击它
phonectl current_app         # 当前前台应用
```

## 启动 App
```bash
phonectl launch_pkg com.android.camera  # 按包名启动
phonectl open_url "https://example.com" # 打开网址
```

## 拍照流程
相机快门按钮没有文字，只能用坐标点击。
```bash
# 1. 打开相机
phonectl launch_pkg com.android.camera
sleep 3

# 2. 点击快门（白色圆形按钮，坐标 540,2000）
phonectl tap 540 2000
sleep 2

# 3. 查看最新照片文件名
phonectl shell "ls -t /sdcard/DCIM/Camera/ | head -1"

# 4. 复制照片到 BotDrop home 目录（方便读取）
phonectl shell "cat /sdcard/DCIM/Camera/IMG_xxxx.jpg" >~/photo.jpg
```

### 相机界面常用坐标
- **快门按钮**：`540 2000`（resource-id: `shutter_button_horizontal`，content-desc: `拍摄`）
- **前后置切换**：`850 2000`（resource-id: `v9_camera_picker_horizontal`）
- **缩略图/相册**：`230 2000`（resource-id: `v9_thumbnail_layout_horizontal`）
- **0.6x 变焦**：`426 1602`
- **1x 变焦**：`540 1602`
- **2x 变焦**：`654 1602`
- **模式切换**（底部文字栏 y≈1767）：专业 `97 1767` / 录像 `237 1767` / 拍照 `375 1767` / 人像 `520 1767` / 更多 `650 1767`

## 文件访问
BotDrop 可以直接读写 `/sdcard/` 下的文件，但需注意：
- `/sdcard` 是 `/storage/self/primary` 的软链接
- BotDrop 的 home 目录是 `/data/data/app.botdrop/files/home`
- BotDrop 的临时目录是 `$TMPDIR`（即 `/data/data/app.botdrop/files/usr/tmp`），**不是** `/tmp`（不存在）
- 照片保存路径：`/sdcard/DCIM/Camera/IMG_yyyyMMdd_HHmmss.jpg`
- 截图保存路径：`/sdcard/screen.png`（或 `phonectl screenshot` 指定的文件名）
- 读取照片文件用绝对路径，二进制模式：`open('/sdcard/DCIM/Camera/xxx.jpg', 'rb')`
- 也可以先复制到 home 目录：`cp /sdcard/DCIM/Camera/xxx.jpg ~/photo.jpg`

## 技术要点
- 屏幕分辨率：**1080 x 2340**
- 核心原理：通过 `/dev/uinput` 创建虚拟触摸屏设备（`INPUT_PROP_DIRECT` + `BUS_VIRTUAL`），绕过了 MIUI 对 `input` 命令和 `INJECT_EVENTS` 权限的限制
- 每次 tap/swipe 有约 2 秒设备注册延迟（uinput 设备创建开销）
- 工作链路：BotDrop SSH → ADB shell (`localhost:5555`) → `/data/local/tmp/uinput_touch` → 内核触摸事件
- `phonectl` 脚本内已内置 `LD_LIBRARY_PATH`，无需依赖 `.bashrc` 环境变量
- `phonectl` 位于 `/data/data/app.botdrop/files/usr/bin/phonectl`，在 PATH 中可直接调用
