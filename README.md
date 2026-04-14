# 监控回放系统

一个基于 Web 的监控视频回放系统，支持多摄像头视频管理、H.265 视频播放、帧截图浏览等功能。

测试数据来源于小米的摄像机录像

## 功能特性

- **多摄像头支持**：可配置多个摄像头，每个摄像头包含多个视频片段
- **时间轴导航**：可视化时间轴，支持拖拽定位
- **两种播放模式**：
  - **帧截图模式**：每秒截取一帧 JPEG 图片，兼容性好
  - **源视频播放模式**：后端 FFmpeg 转码 H.265 → H.264，支持拖拽
- **播放控制**：播放/暂停/停止、变速（0.25x ~ 60x）、逐帧跳转
- **日期跳转**：快速跳转到指定日期

## 环境要求

- Python 3.8+
- FFmpeg（已安装本地）
- 现代浏览器（Chrome/Firefox/Edge/Safari），其中只有 safari 支持源视频播放模式。

## 安装部署

### 1. 安装依赖

```bash
pip install -r requirements.txt
```

### 2. 配置摄像头

编辑 `config.yaml`：

```yaml
cameras:
  - name: "摄像头名称"
    folder: "./视频目录路径"

# 文件名格式 - 用正则捕获开始和结束时间
filename_pattern: "_(\\d{14})_(\\d{14})\\.mp4$"
time_format: "%Y%m%d%H%M%S"

server:
  host: "0.0.0.0"
  port: 8080
```

### 3. 视频文件命名规范

视频文件名需包含开始和结束时间戳，格式示例：
```
video_0400_0_10_20250613113832_20250613115449.mp4
```
即：`..._<开始YYYYMMDDHHmmss>_<结束YYYYMMDDHHmmss>.mp4`

### 4. 启动服务

```bash
python server.py
```

然后在浏览器打开 http://localhost:8080

## 视频编码说明

### 问题背景

原始监控视频通常使用 H.265 (HEVC) 编码，但大多数桌面浏览器（Chrome、Firefox、Edge）原生不支持 H.265 解码。

### 解决方案

本系统提供两种播放模式：

| 模式 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| 帧截图 | 后端 FFmpeg 截取单帧 | 兼容性好 | 不连续 |
| 源视频播放 | safari 打开 | 支持拖拽、快进 | 直接可打开 |

### 使用源视频播放模式

1. 在播放模式选择器中选择 **"源视频播放模式"**
2. 支持拖拽进度条

## 项目结构

```
monitor_player/
├── server.py          # FastAPI 后端服务
├── scanner.py         # 视频文件扫描器
├── config.yaml        # 配置文件
├── requirements.txt  # Python 依赖
├── transcoded/       # 转码缓存目录（自动创建）
├── static/            # 前端静态文件
│   ├── index.html
│   ├── app.js
│   ├── style.css
│   └── timeline.js
└── 视频目录/          # 监控视频存放目录
    ├── 摄像头1/
    ├── 摄像头2/
    └── ...
```

## API 接口

| 接口 | 说明 |
|------|------|
| `GET /api/cameras` | 获取所有摄像头信息 |
| `GET /api/frame/{cam_id}` | 获取指定时间的帧截图 |
| `GET /api/clip_info/{cam_id}` | 查询某时刻对应的视频文件 |
| `GET /api/stream/{cam_id}` | 流式返回 mp4 片段（转码后） |
| `GET /api/video_data/{cam_id}/{filename}` | 获取视频原始字节（支持 Range，用于 ffmpeg.wasm） |
| `GET /stream/{camera_index}/{filename}` | FFmpeg 实时转码 H.265 → H.264 流式播放 |
| `GET /video_files/{cam_id}/{filename}` | 提供原始视频文件（支持 Range 请求） |

## 技术栈

- 后端：FastAPI + Python + FFmpeg
- 前端：原生 JavaScript + CSS
- 视频处理：FFmpeg

## 核心模块说明

### server.py

FastAPI 后端服务，主要功能：

- **帧截图 API** (`/api/frame/{cam_id}`): 使用 FFmpeg 从视频中截取指定时间的单帧 JPEG 图片
- **流媒体 API** (`/api/stream/{cam_id}`): 将视频转为 H.264 格式流式传输，支持播放
- **视频文件服务** (`/video_files/{cam_id}/{filename}`): 直接提供原始视频文件，支持 Range 请求，可拖拽播放
- **实时转码 API** (`/stream/{camera_index}/{filename}`): 实时将 H.265 转码为 H.264 流式输出
- **视频数据 API** (`/api/video_data/{cam_id}/{filename}`): 提供视频原始字节，支持 Range 请求，供 ffmpeg.wasm 解码使用

### scanner.py

视频文件扫描器，负责：

- 从配置的文件夹中递归扫描视频文件
- 使用正则表达式从文件名解析起止时间
- 按时间排序视频片段
- 提供二分查找快速定位某时刻对应的视频文件
- 合并连续/重叠片段，生成时间轴数据

### 前端功能

- **帧截图模式**: 每秒请求一帧 JPEG，适合不支持 H.265 的浏览器
- **源视频播放模式**: 直接播放原始视频（Safari）或使用 ffmpeg.wasm 解码
- **时间轴**: 可视化显示有视频的时间段，支持拖拽定位
- **播放控制**: 播放/暂停/停止、变速（0.25x ~ 60x）、逐帧跳转
- **日期跳转**: 选择日期快速定位到对应时间的视频

## 常见问题

### 1. 视频播放失败

确保系统已安装 FFmpeg：
```bash
ffmpeg -version
```

### 2. 视频文件不被识别

检查 `config.yaml` 中的 `filename_pattern` 是否与实际文件名匹配。

### 3. Safari 播放 H.265

Safari 原生支持 H.265 解码，可直接使用源视频播放模式获得最佳体验。

### 4. Chrome/Firefox/Edge 播放

这些浏览器不支持 H.265，建议使用帧截图模式。
