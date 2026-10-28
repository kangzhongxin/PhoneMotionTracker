# 手机移动轨迹实时追踪（PhoneMotionTracker）

使用 **C# + WinForms + YoloDotNet + OpenCvSharp** 调用电脑摄像头，实时检测画面中的手机，并绘制手机的当前位置、检测框和最近 6 秒的移动轨迹。

## 功能

- 调用本机摄像头（支持切换摄像头编号，优先使用 Media Foundation 以获得满帧率）
- YOLOv8n（COCO 数据集）实时检测 `cell phone` 类别
- One Euro 自适应平滑：同时平滑中心点和检测框宽高，静止时抑制抖动、移动时保持跟手
- 短暂漏检时保持最后位置 300 ms，避免检测框和中心点闪烁
- 用 Catmull-Rom 插值绘制平滑轨迹；旧点偏蓝、新点偏黄，保留最近 6 秒
- 支持调整置信度阈值、显示/隐藏轨迹、清除轨迹
- 显示实际显示 FPS、单帧推理耗时、轨迹点数量
- 可替换为自己的任意 YOLO ONNX 检测模型

## 环境要求

- Windows 10/11
- .NET 8 SDK（或 Visual Studio 2022 / 2026）
- 可用的摄像头

主要 NuGet 包：

| 包 | 版本 | 用途 |
| --- | --- | --- |
| OpenCvSharp4 | 4.13.0.20260627 | 摄像头采集、图像绘制 |
| OpenCvSharp4.runtime.win | 4.13.0.20260627 | OpenCV Windows 原生库 |
| OpenCvSharp4.Extensions | 4.13.0.20260627 | Mat 与 Bitmap 转换 |
| YoloDotNet | 4.2.0 | YOLO ONNX 推理 |
| YoloDotNet.ExecutionProvider.Cpu | 1.1.0 | ONNX Runtime CPU 执行提供程序 |

## 运行

用 Visual Studio 打开 `PhoneMotionTracker.sln`，直接 F5；或使用命令行：

```bash
dotnet run --project PhoneMotionTracker/PhoneMotionTracker.csproj
```

操作步骤：

1. 选择摄像头编号（默认 `0`）。
2. 设置置信度阈值（默认 `0.35`）。
3. 点击「开始检测」。
4. 手机进入画面后，会显示橙色检测框、绿色中心标记和移动轨迹。
5. 点击「清除轨迹」可清空历史轨迹；点击「停止」结束采集并释放摄像头。

## 实现流程

1. `OpenCvSharp.VideoCapture` 依次尝试 **Media Foundation（MSMF）**、DirectShow 和系统默认后端，默认请求 1280×720 / 30 FPS。部分摄像头在 DirectShow 下只有约 17 FPS，MSMF 下可达到 30 FPS。
2. 采集线程把 BGR 帧转换为 BGRA，通过固定（pin）的字节缓冲区以零拷贝方式构造 `SKImage`，交给 YoloDotNet 推理。
3. 过滤标签为 `cell phone` / `phone` / `mobile phone` 等的结果；同屏有多部手机时，选择离上一帧中心最近的目标，保证跟踪连续。
4. 使用 One Euro 滤波器分别平滑中心点 X/Y 和检测框宽/高，静止时强力去抖、快速移动时自动降低平滑度；检测框和中心标记都使用平滑后的结果。
5. 漏检不超过 300 ms 时保留上一帧的框和位置，避免闪烁；超过 800 ms 自动断开轨迹，防止把断点连成长直线。
6. 在采集线程中按显示区尺寸预缩放，UI 通过 `VideoSurface` 做 1:1 位图绘制，避免 `PictureBox` 每帧进行 GDI+ 缩放。
7. 轨迹使用 Catmull-Rom 插值平滑，避免 30 FPS 离散点形成明显折线；UI 用 `BeginInvoke` 更新，界面来不及显示时丢弃旧帧。

## 模型

仓库中的 `PhoneMotionTracker/models/yolov8n.onnx` 为 **Ultralytics YOLOv8n（COCO）** 模型，opset 17，输入 640×640，包含 COCO 的 80 个类别，其中手机对应类别 `cell phone`（索引 67）。

生成方式：

```bash
yolo export model=yolov8n.pt format=onnx opset=17 imgsz=640
```

> **模型许可**：Ultralytics YOLO 预训练模型通常采用 **AGPL-3.0** 或需要商业授权，商用前请自行确认许可要求。YoloDotNet 本身为 MIT 许可。也可以点击界面中的「浏览...」替换为您自己训练/授权的 ONNX 模型。

模型文件会在构建时自动复制到输出目录的 `models/` 文件夹。

## 性能参考

在本机（i9-13900HX，CPU 推理，1280×720 采集）实测：

- 单帧推理约 **20–26 ms**
- DirectShow 摄像头出帧仅 **16.9 FPS**，改用 Media Foundation 后为 **30.3 FPS**
- 应用实际显示 **25–31 FPS**，UI 掉帧数为 0
- 缩放合成约 2 ms，Mat → Bitmap 约 1 ms

如需 GPU 加速，可把 `YoloDotNet.ExecutionProvider.Cpu` 替换为 `YoloDotNet.ExecutionProvider.Cuda`，并在代码中使用 CUDA 执行提供程序（注意：YoloDotNet 要求只能引用一个执行提供程序包）。

## 目录结构

```
PhoneMotionTracker.sln
README.md
PhoneMotionTracker/
├── Form1.cs                 # 摄像头采集、推理、平滑跟踪、轨迹绘制
├── Form1.Designer.cs        # WinForms 界面
├── VideoSurface.cs          # 1:1 快速绘制控件，避免每帧 GDI+ 缩放
├── Program.cs
├── PhoneMotionTracker.csproj
└── models/yolov8n.onnx      # 默认检测模型
```

## 常见问题

- **打不开摄像头**：检查摄像头是否被微信、腾讯会议等程序占用；在「设置 → 隐私和安全性 → 摄像头」中允许桌面应用访问摄像头。
- **检测不到手机**：手机在画面中太小或太暗时置信度会偏低，可适当降低置信度阈值；也可换用更大的模型（如 YOLOv8s/m）提高精度。
- **首次运行较慢**：启动时需要加载 ONNX 模型，通常 1 秒左右；之后每帧推理保持在毫秒级。
- **移动画面不够流畅**：优先确认摄像头是否运行在 30 FPS。程序已优先使用 MSMF；若摄像头只支持 DirectShow 且帧率只有 15–20 FPS，画面本身就会发卡，与跟踪算法无关。可尝试降低采集分辨率或在显卡上使用 CUDA 执行提供程序。
