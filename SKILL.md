---
name: windows-region-exact-record
display_name: Windows 区域录屏 精确时长 录屏10秒 gdigrab ffmpeg 高DPI坐标
display_name_en: Windows Exact-Duration Region Recording
description: 在 Windows 上按精确时长录制屏幕指定矩形区域。含已实测的 ffmpeg gdigrab 参数（物理像素偏移）、200% 缩放下的坐标陷阱、免安装的截图工具路径，以及一套开箱即用的取坐标+录制脚本。
description_zh: Windows 区域录屏精确时长（区域录屏/录屏10秒/指定区域录制/gdigrab/ffmpeg 抓屏/offset_x/video_size/高DPI 缩放坐标不对/录屏尺寸减半/show_region 校验/无声音/mp4）：物理像素坐标陷阱 + 脚本 + 验收口径。
description_en: Record an exact-length screen region on Windows with ffmpeg gdigrab, including the high-DPI physical-pixel coordinate trap and a picker script.
category: 媒体工具
version: 1.0.0
author: WorkBuddy
agent_created: true
---

# Windows 区域录屏（精确时长）

## 何时用

需要「录制屏幕上某一块矩形、时长卡死在 N 秒」的场景：桌宠/动效 demo、UI 演示、meme 素材片段、给客户看某个局部的行为。

## 工具位置（已装好，含内置 ffmpeg，无需 PATH）

```
C:\Users\mercy\Tools\screen-rec\
  region_rec_gui.py # GUI（推荐）：拖拽框选 → 确认 → 开始 → 结束，录制中有红框+状态条
  rec_gui.bat       # 双击开 GUI（pythonw，无黑框）
  pick_region.py    # 点两下取矩形（左上 + 右下），输出物理像素 X Y W H
  rec_region.py     # 按精确秒数录制该区域；--pick 先点选、--check 先画框校验
  rec10s.bat        # 命令行入口（见下表）
  ffmpeg\bin\ffmpeg.exe  # 便携版，脚本优先用它，其次 PATH
  last_region.txt   # pick 后的区域缓存，rec 默认读它
  last_run.txt      # GUI 每次启动写的自检：DPI API 结果 / 缩放 / 坐标口径
```

Python 用托管解释器：`C:\Users\mercy\.workbuddy\binaries\python\versions\3.13.12\python.exe`

## GUI（首选，2026-09-25 加）

双击 `rec_gui.bat`。面板默认在**右上角**（避开 WorkBuddy 侧栏），可拖标题栏挪。

| 步骤 | 操作 |
|---|---|
| ① 框选区域 | 点按钮 → 全屏变暗 → **按住左键拖出矩形**（实时显示尺寸）→ 松开。右键 / ESC 取消 |
| ② 确认选区 | 点「确认选区」→ 暗屏消失，选区周围出现**绿框**。不满意点「重画」 |
| ③ 开始录制 | 点「开始录制」→ 面板隐藏，选区外出现**红框**，下方浮出状态条「● REC 0:05.3 (手动结束)」 |
| ④ 结束录制 | 点状态条上的「结束录制」（或面板上的同名按钮）→ 面板回来并显示文件名与大小时长 |

- **「限时(秒)」** 填 0 = 手动结束；填 10 = 到 10 秒自动停。两种模式都实测过。
- 红框和状态条**自动摆在录制矩形之外**（红框向外 3px、状态条优先放选区正下方），所以不会被录进画面；红框本身用「外框减内框」的窗口区域实现，内部点击可穿透。
- 关闭面板会先收尾封装视频（ffmpeg 收 `q`），不会留半截损坏文件。
- 自检模式（不碰鼠标、自动跑完就退出，QA 用）：
  `python region_rec_gui.py --selftest "1000 700 1200 800" --selftest-sec 5 --out D:\tmp`

`rec10s.bat` 四个命令行入口：

| 命令 | 行为 |
|---|---|
| 双击（无参数） | 点两下框选 → 3 秒倒计时 → 精确录 10 秒 |
| `rec10s.bat last` | 复用 `last_region.txt` 里的区域，倒计时 3 秒后录 10 秒 |
| `rec10s.bat pick` | 只取区域、缓存坐标，不录 |
| `rec10s.bat check` | 画 3 秒方框校验区域，不落盘 |

## 三步走

1. 双击 `rec10s.bat` → 点左上角，再点右下角（ESC 中止，120 秒无点击自动退出）。坐标写入 `last_region.txt`。
2. **偏移有疑问先 `rec10s.bat check`** → `-show_region 1` 在屏幕上闪 3 秒方框，肉眼确认框住的是想要的那块。
3. 倒计时结束后自动开录，精确 10 秒，输出到 `~/Videos/Screen Recordings/region_<时间戳>_10s.mp4`。

命令行等价：

```
python rec_region.py --sec 10 --pick --countdown 3     # 点选 + 录
python rec_region.py --sec 10 --region "X Y W H" --fps 30 --countdown 3
python rec_region.py --sec 10 --countdown 0 --out D:\out.mp4
```

## 硬规则与坑（2026-09-25 实测于 4K@200% 机器）

1. **坐标必须是物理像素。** 取坐标的进程必须 DPI-aware，否则在 200% 缩放下拿到的是逻辑像素、只有实际值的一半，录出来会录到错误位置。优先用 `SetProcessDpiAwarenessContext(-4)`（per-monitor v2），失败再退 `shcore.SetProcessDpiAwareness(2)`。`pick_region.py` 与 `region_rec_gui.py` 都已处理。
2. **gdigrab 看到的是物理桌面。** 实测默认抓桌面 `3840x2160`（不是缩放后的 1920x1080），`offset_x/offset_y` 也按物理像素算、从主屏左上角起算。所以「DPI-aware 取坐标 + gdigrab 用坐标」是同一套口径，可直接互用。
3. **宽高必须偶数。** `yuv420p` 不允许奇数，脚本自动向下取偶；手写命令要自己减。
4. **时长不是帧精确，别用帧数验收。** `-t 10` 实测：容器 `duration=10.000000`，但只有 298 帧 @30fps（末帧 9.90s）——gdigrab 会补帧/丢帧。验收看 `duration`，不看 `nb_frames`。
5. **gdigrab 不录音。** 需要系统声/麦克风就换系统截图工具（见下），或另加 `-f dshow -i audio="..."` 混流。
6. **别用 `-show_region 1` 正式录制**，那个方框会被一起录进去（校验完记得去掉）。
7. **取坐标进程要有控制台才能用 ESC 中止。** 从脚本/管道里后台拉起时没有控制台，`msvcrt.kbhit()` 会抛异常——已用 try/except 兜住，此时靠 120 秒无点击自动退出。

## 用 ctypes 写 Win32 小工具时的坑（写 GUI 踩到的，2026-09-25）

8. **必须声明 argtypes**，否则 Python int 会被当 `c_int` 传，64 位句柄和特殊值全部出错。
   真实翻车现场：`SetProcessDpiAwarenessContext(-4)` 不写 argtypes 直接返回 0（失败），必须
   `f.argtypes=[ctypes.c_void_p]` 且传 `ctypes.c_void_p(-4)`；`CreateWindowExW` 的 hInstance 句柄过长会报
   `OverflowError: int too long to convert`。
9. **函数在哪个 DLL 要查清楚**：`GetStockObject`/`CreateFontW`/`CreatePen` 在 gdi32，而 `FillRect`/`DrawTextW` 在 user32。
   写错会得到 `AttributeError: function 'X' not found`（不是崩溃，是加载时就炸）。
10. **DPI 感知必须在创建任何窗口/取任何 DC 之前设置。** 没设成的表现：`GetDeviceCaps(88)` 返回 96、
    `GetSystemMetrics(0)` 返回逻辑尺寸、界面被位图放大（发虚）、鼠标坐标是逻辑像素（差 2 倍）。
    自检写法：`dmPelsWidth / GetSystemMetrics(0)`，比值 1.0 = 物理坐标可直接用，>1 = 逻辑坐标要乘回去。
11. **WNDPROC 里一定要 try/except 并落盘日志。** pythonw 没有 stdout，一处 ctypes 参数错误（例如把坐标元组
    当窗口句柄传）只会让「开始录制」静默半途失败，看不出来。日志文件比猜快十倍。
12. **验收指示器有没有被录进画面**：取视频边缘 6px 条带求平均色，等于内容色而不是指示色即为合格。
    实测 `crop=6:800:0:0,scale=1:1` → (248,243,249)，上边缘 (255,255,255)，而红框是 (234,31,33)。
13. **自动化验证的时序陷阱**：`sleep` 和截图必须在**同一次** shell 调用里，否则两次工具调用之间的延迟会让
    录制早已结束；优先用程序自带的 `--selftest` 模式，而不是合成鼠标点击（合成点击会跟用户抢鼠标）。
14. **从对话里帮用户启动这个 GUI，必须用常驻后台方式**（`run_in_background`）。用 `nohup` 或 `cmd /c start`
    拉起的进程会在该次工具调用结束时被回收（沙箱会收整个进程树），表现为「启动后几秒窗口就消失了」。
    启动后建议截图确认面板真的出现，别只看进程列表。

## 验收口径

```
ffprobe -v error -select_streams v:0 -show_entries stream=width,height,r_frame_rate,codec_name,pix_fmt -show_entries format=duration,size -of default=noprint_wrappers=1 out.mp4
```
pass 条件：`width/height` = 请求值、`r_frame_rate` = 请求帧率、`duration` = 请求秒数、`pix_fmt=yuv420p`、`codec_name=h264`。

## 零安装替代（够用就别装东西）

Windows 11 自带截图工具就能区域录制：`Win + Shift + R` → 拖出区域 → 「开始」（有 3 秒倒计时）→ 到 10 秒点「停止」。
- 没有自动停：**手动卡 10 秒**，或在编辑器里把首尾裁到 10 秒。
- 默认存 `C:\Users\<用户>\Videos\Screen Recordings`，MP4、无水印，支持系统声 + 麦克风。
- 上限：无硬限制，但超过 15–20 分钟容易保存失败；只录单一显示器。
- 它是「软」10 秒，要可复现的精确时长就走上面的 ffmpeg 路线。
