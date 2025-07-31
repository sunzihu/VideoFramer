# FrameGrab

FrameGrab 是一款专业的视频帧提取工具，专为需要从视频中提取静态图像的用户设计。无论是内容创作者、视频编辑师还是普通用户，都能通过简单的操作，每秒提取一帧高质量图像。

## 项目描述

FrameGrab 是一款基于 uni-app 开发的视频帧提取应用，允许用户拍摄照片、录制视频，并以1秒为间隔从视频中提取帧。

### 主要功能：
- **视频帧提取**：自动每秒提取一帧，捕捉视频中的关键瞬间
- **多种视频来源**：支持拍摄新视频、上传本地视频或选择相册中的视频
- **高质量输出**：提取的帧保持原视频分辨率，确保图像清晰度
- **便捷保存**：一键保存单帧或批量保存所有帧到相册
- **实时预览**：直观展示提取的所有帧，方便浏览和选择

### 技术特点：
- 基于 uni-app 开发，支持多平台部署
- 采用 renderjs 技术实现纯前端视频处理
- 优化的视频帧提取算法，确保提取质量和效率
- 完善的错误处理机制，提供稳定可靠的用户体验

## 功能特点

- 使用设备相机拍摄照片
- 使用设备相机录制视频
- 从设备相册上传视频
- 以1秒为间隔从视频中提取帧
- 以网格布局显示提取的帧
- 一键保存单个帧或所有帧到相册
- 实时显示提取进度

## 如何运行此项目

本项目基于UniApp构建，需要HBuilderX才能正常运行。

### 前提条件

1. 下载并安装 [HBuilderX](https://www.dcloud.io/hbuilderx.html)

### 运行项目

1. 打开HBuilderX
2. 点击"文件" > "打开文件夹"并选择此项目文件夹
3. 在项目资源管理器中，右键点击项目并选择"运行" > "运行到浏览器"进行H5预览
4. 对于移动功能（相机、视频录制），请使用：
   - "运行" > "运行到手机模拟器"用于Android/iOS模拟器
   - "运行" > "运行到自定义手机"用于实体设备测试

### 重要说明

- 相机和视频录制功能需要真实设备或模拟器
- 帧提取功能在实际设备上效果最佳，因为需要较高的性能
- 为获得最佳效果，请使用HBuilderX应用在真实移动设备上进行实时预览测试
- 使用renderjs技术实现的帧提取在不同平台上可能有差异，App和H5平台支持最佳

## 项目结构

- `pages/index.vue` - 包含UI和功能的主页面
- `static/image/` - 应用图标
- `manifest.json` - 应用配置和权限
- `pages.json` - 页面路由配置
- `main.js` - 入口点
- `App.vue` - 全局应用配置

## 技术实现

本项目使用了以下关键技术：
- **renderjs** - 用于在前端直接处理视频帧提取
- **plus API** - 用于处理base64图像数据和保存到相册
- **uni-app组件** - 提供跨平台UI和功能

## 故障排除

如果提取后图像不显示：
1. 确保您在真实设备或模拟器上进行测试
2. 检查控制台日志是否有错误
3. 确保应用具有相机和存储访问的适当权限
4. 如果图像为空白，可能是由于跨域安全限制，请检查renderjs实现



---

# FrameGrab (English Version)

FrameGrab is a professional video frame extraction tool designed for users who need to extract static images from videos. Whether you're a content creator, video editor, or regular user, you can easily extract one high-quality frame per second with simple operations.

## Project Description

FrameGrab is a video frame extraction application developed based on uni-app, allowing users to take photos, record videos, and extract frames from videos at 1-second intervals.

### Main Features:
- **Video Frame Extraction**: Automatically extract one frame per second, capturing key moments in videos
- **Multiple Video Sources**: Support for shooting new videos, uploading local videos, or selecting videos from the gallery
- **High-Quality Output**: Extracted frames maintain the original video resolution, ensuring image clarity
- **Convenient Saving**: One-click saving of individual frames or batch saving of all frames to the album
- **Real-time Preview**: Intuitive display of all extracted frames for easy browsing and selection

### Technical Features:
- Developed based on uni-app, supporting multi-platform deployment
- Uses renderjs technology for pure front-end video processing
- Optimized video frame extraction algorithm, ensuring extraction quality and efficiency
- Comprehensive error handling mechanism, providing stable and reliable user experience

## Features

- Take photos using the device camera
- Record videos using the device camera
- Upload videos from the device gallery
- Extract frames from videos at 1-second intervals
- Display extracted frames in a grid layout
- One-click saving of individual or all frames to the album
- Real-time display of extraction progress

## How to Run This Project

This project is built with UniApp, which requires HBuilderX to run properly.

### Prerequisites

1. Download and install [HBuilderX](https://www.dcloud.io/hbuilderx.html)

### Running the Project

1. Open HBuilderX
2. Click "File" > "Open Folder" and select this project folder
3. In the project explorer, right-click on the project and select "Run" > "Run to Browser" for H5 preview
4. For mobile functionality (camera, video recording), use:
   - "Run" > "Run to Phone Emulator" for Android/iOS emulator
   - "Run" > "Run to Custom Phone" for physical device testing

### Important Notes

- Camera and video recording features require a real device or emulator
- The frame extraction feature works best on actual devices due to performance requirements
- For best results, test on a real mobile device using the HBuilderX app for real-time preview
- Frame extraction implemented with renderjs technology may vary across platforms, with best support on App and H5 platforms

## Project Structure

- `pages/index.vue` - Main page with UI and functionality
- `static/image/` - Icons for the application
- `manifest.json` - App configuration and permissions
- `pages.json` - Page routing configuration
- `main.js` - Entry point
- `App.vue` - Global app configuration

## Technical Implementation

This project uses the following key technologies:
- **renderjs** - For direct front-end processing of video frame extraction
- **plus API** - For handling base64 image data and saving to the album
- **uni-app components** - Providing cross-platform UI and functionality

## Troubleshooting

If images are not displaying after extraction:
1. Make sure you're testing on a real device or emulator
2. Check console logs for any errors
3. Ensure the app has proper permissions for camera and storage access
4. If images are blank, it may be due to cross-origin security restrictions; check the renderjs implementation

<img width="611" height="1089" alt="a2bf9645e0a4c79f6b32ffed8857c644" src="https://github.com/user-attachments/assets/a1179245-b768-4a9d-b3a1-d7ca15609617" />
