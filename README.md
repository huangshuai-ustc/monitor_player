# Monitor Merge - 监控视频多画幅拼接工具

一个用于将多个监控摄像头的视频合并为多画幅拼接视频的 Python 工具。

## 功能特性

- **多画幅拼接**：支持 1-9 路监控视频同时显示
- **智能时间对齐**：自动识别各路视频的时间，实现精确同步
- **多种时间提取方式**：OCR 识别画面时间 / 文件名解析 / 视频 metadata
- **自动布局**：根据监控数量自动选择最佳布局（单屏/双屏/田字格/2x3/3x3）
- **容错处理**：某路视频缺失时自动显示黑屏
- **时间戳水印**：输出视频带有时间戳显示

## 环境要求

- Python 3.8+
- FFmpeg + ffprobe
- OpenCV (`opencv-python`)
- Tesseract OCR (可选，用于 OCR 时间提取)

## 安装

### 1. 安装 Python 依赖

```bash
pip install -r requirements.txt
```

### 2. 安装 FFmpeg

**macOS:**
```bash
brew install ffmpeg
```

**Ubuntu/Debian:**
```bash
sudo apt install ffmpeg
```

**Windows:**
从 [ffmpeg.org](https://ffmpeg.org/download.html) 下载并配置

### 3. 安装 Tesseract OCR (可选)

如需使用 OCR 时间提取功能：

**macOS:**
```bash
brew install tesseract
```

**Ubuntu/Debian:**
```bash
sudo apt install tesseract-ocr
```

**Windows:**
从 [UB-Mannheim](https://github.com/UB-Mannheim/tesseract/wiki) 下载

## 项目结构

```
monitor_merge/
├── main.py           # 主入口程序
├── time_extract.py   # 时间信息提取模块
├── timeline.py       # 时间轴构建模块
├── merge.py          # 视频合并模块
├── config.yaml       # 配置文件
├── requirements.txt # Python 依赖
└── LICENSE           # MIT 许可证
```

## 快速开始

### 1. 安装依赖

```bash
# 系统依赖
# Ubuntu/Debian:
sudo apt install ffmpeg tesseract-ocr

# macOS:
brew install ffmpeg tesseract

# Python 依赖
pip install opencv-python pytesseract pyyaml
```

### 2. 运行

```bash
# 最简用法：指定文件夹即可
python main.py --folders ./cam1 ./cam2 ./cam3 ./cam4

# 用文件名提取时间（如文件名包含 20240101_080000）
python main.py --folders ./cam1 ./cam2 --method filename

# 自定义输出
python main.py --folders ./cam1 ./cam2 ./cam3 \
    --output result.mp4 \
    --resolution 3840x2160 \
    --fps 30

# 仅检查不合并
python main.py --folders ./cam1 ./cam2 --check
```

### 3. 布局自动判断

```
1个文件夹 → 全屏
┌──────┐
│  1   │
└──────┘

2个文件夹 → 左右分屏
┌───┬───┐
│ 1 │ 2 │
└───┴───┘

3~4个文件夹 → 田字格
┌───┬───┐
│ 1 │ 2 │
├───┼───┤
│ 3 │ 4 │  (3个时右下黑屏)
└───┴───┘

5~6个文件夹 → 2×3
┌───┬───┬───┐
│ 1 │ 2 │ 3 │
├───┼───┼───┤
│ 4 │ 5 │ 6 │
└───┴───┴───┘
```

## 配置说明

### config.yaml 完整配置项

```yaml
# 输入文件夹列表
input_folders:
  - ./cam1
  - ./cam2
  - ./cam3
  - ./cam4

# 输出文件路径
output_file: ./output/merged.mp4

# 输出分辨率
output_width: 1920
output_height: 1080

# 输出帧率
fps: 25

# 时间提取方式: ocr | filename | metadata | auto
time_extract_method: ocr

# OCR 区域设置（相对于视频尺寸的比例）
ocr_region:
  x: 0.0    # 起始 X 坐标 (0-1)
  y: 0.0    # 起始 Y 坐标 (0-1)
  w: 0.35   # 宽度比例
  h: 0.06   # 高度比例

# 时间格式（用于 OCR 解析）
time_format: "%Y-%m-%d %H:%M:%S"

# 文件名时间正则（用于 filename 方式）
filename_pattern: "(\\d{4})(\\d{2})(\\d{2})_(\\d{2})(\\d{2})(\\d{2})"
```

### 命令行参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--folders`, `-f` | 监控视频文件夹路径 | config.yaml 中的值 |
| `--output`, `-o` | 输出文件路径 | ./output/merged.mp4 |
| `--config`, `-c` | 配置文件路径 | config.yaml |
| `--method`, `-m` | 时间提取方式 | ocr |
| `--resolution`, `-r` | 输出分辨率 | 1920x1080 |
| `--fps` | 输出帧率 | 25 |
| `--time-format` | OCR 时间格式 | %Y-%m-%d %H:%M:%S |
| `--filename-pattern` | 文件名时间正则 | (\\d{4})(\\d{2})(\\d{2})_(\\d{2})(\\d{2})(\\d{2}) |
| `--check` | 仅检查依赖和扫描文件 | False |
| `--temp-dir` | 临时文件目录 | 系统临时目录 |

### 时间提取方式说明

| 方式 | 说明 | 适用场景 |
|------|------|----------|
| `ocr` | 从视频画面左上角 OCR 识别时间 | 监控画面有清晰时间戳 |
| `filename` | 从文件名解析时间 | 文件名包含时间信息如 `20240101_120000.mp4` |
| `metadata` | 读取视频文件的创建时间 | 视频 metadata 包含正确时间 |
| `auto` | 自动尝试上述方式 | 不确定时间来源时 |

## 使用示例

### 示例 1：基本用法

```bash
# 合并 4 路监控
python main.py --folders ./cam1 ./cam2 ./cam3 ./cam4
```

### 示例 2：指定输出

```bash
python main.py --folders ./cam1 ./cam2 --output ./merged.mp4
```

### 示例 3：从文件名提取时间

```bash
python main.py --folders ./cam1 ./cam2 --method filename
```

### 示例 4：自定义分辨率和帧率

```bash
python main.py --folders ./cam1 ./cam2 ./cam3 --resolution 3840x2160 --fps 30
```

### 示例 5：仅检查文件（不合并）

```bash
python main.py --folders ./cam1 ./cam2 ./cam3 ./cam4 --check
```

## 布局说明

工具会根据监控数量自动选择最佳布局：

| 监控数量 | 布局 |
|----------|------|
| 1 | 1x1 全屏 |
| 2 | 2x1 左右分屏 |
| 3-4 | 2x2 田字格 |
| 5-6 | 3x2 |
| 7-9 | 3x3 |

## 工作原理

1. **扫描视频**：遍历每个输入文件夹，提取视频信息
2. **时间对齐**：根据时间信息将所有视频同步到统一时间轴
3. **分段处理**：按时间切分成多个片段，每段包含各监控的视频源信息
4. **视频拼接**：使用 FFmpeg filter_complex 实现多画幅拼接
5. **合并输出**：将所有片段合并为最终视频

## 注意事项

- 输入视频支持格式：MP4, AVI, MKV, MOV, FLV, TS
- 各路视频的时间不需要完全对齐，工具会自动处理时间重叠和空缺
- 某路视频在某个时间段缺失时，该位置会显示黑屏
- 建议各路视频的分辨率和帧率保持一致，以获得最佳效果

## 常见问题

### Q: OCR 识别失败怎么办？

A: 尝试以下方法：
1. 调整 `config.yaml` 中的 `ocr_region` 参数，匹配你视频中时间戳的位置
2. 使用 `--method filename` 从文件名提取时间
3. 使用 `--method metadata` 从视频元数据提取时间

### Q: 合并速度慢怎么办？

A: 
1. 降低输出分辨率：`--resolution 1280x720`
2. 降低帧率：`--fps 15`
3. 使用更快的编码预设（在 merge.py 中调整 `-preset`）

### Q: 如何清理临时文件？

A: 合并完成后会提示是否清理临时文件，输入 `y` 确认清理

## 依赖

- `opencv-python>=4.8` - 视频处理和 OCR
- `pytesseract>=0.3.10` - OCR 文字识别
- `pyyaml>=6.0` - 配置文件解析

## 许可证

MIT License - 详见 [LICENSE](LICENSE) 文件

## 作者

huangshuai - 2026
