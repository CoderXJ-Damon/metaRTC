# MetaRTC 项目文档

## 目录
1. [项目概述](#项目概述)
2. [编译框架](#编译框架)
3. [核心模块](#核心模块)
4. [模块功能详解](#模块功能详解)
5. [编译指南](#编译指南)
6. [项目结构](#项目结构)

---

## 项目概述

### MetaRTC 简介
MetaRTC 是一个为嵌入式/IoT/机器人设备设计的完整 WebRTC 协议栈实现，采用纯 C 语言编写，支持多个平台和架构。

**关键特性：**
- ✓ 完整的 WebRTC 协议栈实现
- ✓ 支持 ICE/STUN/TURN/DTLS/SRTP
- ✓ 完整的 RTP/RTCP 支持
- ✓ 自适应码率和拥塞控制
- ✓ 灵活的媒体处理管道
- ✓ P2P 连接逻辑
- ✓ UDP/TCP 双通道支持
- ✓ 多流、多轨道支持

### 支持平台与架构
**操作系统：**
- Linux / RTOS / Windows / macOS / Android / iOS

**处理器架构：**
- x86 / ARM / MIPS / Loongson / RISC-V

### 核心能力
**音视频支持：**
- 视频编解码：H.264, H.265(HEVC)
- 音频编解码：Opus, G.711A/U, AAC, MP3, Speex
- 音频处理：AEC/ANS/AGC/VAD/CNG

**其他功能：**
- DataChannels 数据通道
- NACK/PLI/FIR/FEC/TWCC 反馈机制
- 支持 WHIP/WHEP 协议
- IPv4/IPv6 双协议栈
- MP4/FLV 格式录制

---

## 编译框架

### 编译系统架构

#### 1. 多架构支持系统
```
arch/
├── yang_build.sh          # 架构选择和参数处理脚本
├── x86_64.cmake           # x86 64位架构配置
├── aarch64.cmake          # ARM 64位架构配置
├── arm32.cmake            # ARM 32位架构配置
├── mips32.cmake           # MIPS 32位架构配置
├── mips64.cmake           # MIPS 64位架构配置
├── wrt32.cmake            # WRT 32位架构配置
├── mac_arm64.cmake        # macOS ARM 64位架构配置
└── ios64.cmake            # iOS 64位架构配置
```

#### 2. 顶级编译脚本
```
cmake_lib_x64.sh          # x64 Linux/Windows 编译脚本
cmake_lib_android.sh      # Android 编译脚本
cmake_lib_ios.sh          # iOS 编译脚本
```

**编译脚本工作流程：**
1. 依次编译 4 个核心库：
   - libyangutil8（音视频处理库）
   - libmetartccore8（WebRTC 协议栈）
   - libyangwhip8（WHIP/WHEP 实现）
   - libmetartc8（高级接口）

2. 每个库通过 `cmake_build.sh` 脚本进行编译
3. 支持参数化：操作系统、位数（32/64）、位置无关代码（PIC）

#### 3. CMake 配置
所有库遵循统一的 CMake 编译流程：
- 最小版本：CMake 2.8+
- 编译标志：`-O2 -std=gnu11(C)` 和 `-std=gnu++11(C++)`
- 构建类型：RELEASE
- 头文件包含路径统一管理

---

## 核心模块

### 模块依赖关系

```
┌─────────────────────────────────────┐
│        libmetartc8 (C++)            │ ← 高级应用接口
│     音视频采集/编解码/传输/渲染     │
└────────────────┬────────────────────┘
                 │
        ┌────────┴─────────┐
        │                  │
┌───────▼────────┐  ┌──────▼────────────┐
│ libmetartccore8│  │  libyangwhip8      │ ← WHIP/WHEP协议
│  WebRTC核心    │  │  推拉流协议        │
│  协议栈        │  └────────────────────┘
└───────┬────────┘
        │
┌───────▼───────────────┐
│   libyangutil8        │ ← 基础工具库
│ 音视频处理/编解码库   │
└───────────────────────┘
```

---

## 模块功能详解

### 1. libyangutil8 - 基础工具库（Pure C）

**位置：** `libyangutil8/src/`

**主要子模块：**

#### yangaudio - 音频处理
- 音频编解码基础
- 音频采样率转换
- 音频数据处理工具

#### yangvideo - 视频处理
- 视频格式转换
- 视频数据处理
- 视频编码优化

#### yangutil - 通用工具
- 内存管理
- 数据结构（链表、队列、缓冲）
- 日志系统
- 配置管理

#### yangjson - JSON 处理
- JSON 解析
- JSON 序列化
- 配置文件处理

**编译特性：**
- 纯 C 实现
- 高度可移植
- 支持交叉编译
- 最小化依赖

---

### 2. libmetartccore8 - WebRTC 协议栈（Pure C）

**位置：** `libmetartccore8/src/`

**主要子模块：**

#### yangice - ICE 协议实现
- ICE Lite 支持
- 候选地址收集
- 连接性检查
- 媒体路径选择

#### yangssl - 加密与安全
- DTLS (Datagram TLS) 实现
- SRTP (Secure RTP) 支持
- 密钥交换
- 证书管理

#### yangsdp - 会话描述协议
- SDP 解析
- 媒体描述处理
- 能力协商
- 会话配置

#### yangrtp - RTP 协议
- RTP 数据包处理
- 时间戳管理
- 序列号处理
- 有效载荷处理

#### yangrtc - WebRTC 核心
- PeerConnection 管理
- RTCSession 控制
- 媒体轨道管理
- 连接状态管理

#### yangstream - 媒体流处理
- 流创建/销毁
- 流状态管理
- 轨道绑定
- 流统计信息

#### yangpush - 推送功能
- 媒体推送接口
- 推流管理
- 推流配置

#### yangpacer - 码率控制
- 发送码率估算
- 拥塞检测
- 码率平滑
- 丢包处理

#### yangutil/sys - 系统工具
- 系统接口
- 线程操作
- 互斥锁
- 事件处理

#### yangavutil - 音视频通用工具
- 音视频格式定义
- 通用数据结构
- 工具函数

**协议支持：**
- ICE / ICE-Lite
- STUN / TURN
- DTLS 1.2+
- SRTP
- RTP/RTCP
- 完整的 WebRTC 媒体处理

**编译特性：**
- 纯 C 实现
- 支持 OpenSSL 或 Mbedtls
- 依赖：libsrtp、usrsctp
- 多平台兼容

---

### 3. libyangwhip8 - WHIP/WHEP 协议（Pure C）

**位置：** `libyangwhip8/src/`

**主要功能：**

#### yangwhip - WHIP/WHEP 实现
- WHIP (WebRTC-HTTP Ingestion Protocol)
  - WebRTC 媒体推入协议
  - HTTP/HTTPS 传输
  - 媒体推流到 SFU/MCU
  
- WHEP (WebRTC-HTTP Egestion Protocol)
  - WebRTC 媒体拉取协议
  - HTTP/HTTPS 传输
  - 媒体拉流 SFU/MCU

**应用场景：**
- 直播推流
- 流媒体拉取
- SFU (Selective Forwarding Unit) 集成
- MCU (Multipoint Control Unit) 集成
- CDN 集成

**编译特性：**
- 纯 C 实现
- 与 libmetartccore8 协作
- HTTP/HTTPS 协议支持

---

### 4. libmetartc8 - 高级接口库（C++）

**位置：** `libmetartc8/src/`

**主要子模块：**

#### yangaudiodev - 音频设备接口
**平台特定实现：**
- Windows：DirectSound/WASAPI
- Linux：ALSA/PulseAudio
- macOS：CoreAudio
- Android：OpenSLES/AAudio

**功能：**
- 音频设备枚举
- 音频采集
- 音频播放
- 设备切换

#### yangcapture - 媒体采集
**视频采集：**
- Windows：DirectShow/DXGI 桌面捕获
- Linux：V4L2
- macOS：AVFoundation
- Android：Camera API

**功能：**
- 摄像头采集
- 屏幕共享
- 分辨率转换
- 帧率控制

#### yangencoder - 视频编码
- H.264 硬件编码（NVENC, QSV, VideoToolbox）
- H.265(HEVC) 硬件编码
- 软件编码支持
- 编码参数调优
- 比特率控制

#### yangdecoder - 视频解码
- H.264 硬件解码
- H.265(HEVC) 硬件解码
- 软件解码
- 解码优化

#### yangplayer - 媒体播放
- 音视频渲染
- 播放控制
- 同步处理
- 缓冲管理

#### yangp2p / yangp2p2 - P2P 连接管理
- P2P 连接建立
- NAT 穿透
- 连接优化
- 连接回落策略

#### yangrtc - RTC 应用接口
- RTCPeerConnection 包装
- RTCSession 接口
- 事件回调

#### yangstream - 流管理
- 流生命周期
- 轨道管理
- 统计信息收集

#### yangutil - C++ 工具
- 智能指针
- 线程管理
- 缓冲区管理
- 日志系统

#### yangavutil - 音视频工具
- 音视频格式转换
- 数据处理工具
- 统计计算

#### yangandroid - Android 特定模块
- Android 平台集成
- JNI 接口
- Android 特定优化

#### yangmac - macOS 特定模块
- macOS 平台集成
- Objective-C 接口
- Metal/OpenGL 支持

#### yangrecord - 媒体录制（Windows/Linux）
- MP4 录制
- FLV 录制
- H.264/H.265 录制
- AAC 音频录制

#### yangsrt - SRT 协议支持（Windows/Linux）
- SRT 推流
- SRT 拉流
- 低延迟传输

**编译特性：**
- C++ 实现
- 依赖 libmetartccore8 和 libyangutil8
- 可选模块（通过 NoCapture、NoPlayer 等标志）
- 平台自适应编译

---

## 编译指南

### 编译参数说明

#### 架构选择 (yang_moc)
```
0 = x86/x64
1 = MIPS
2 = ARM (默认)
3 = RISC-V
4 = Loongson
5 = WRT
6 = macOS
7 = iOS
```

#### 位数选择 (yang_bit)
```
32 = 32位编译
64 = 64位编译 (默认)
```

#### PIC 选择 (yang_pic)
```
0 = 不使用位置无关代码
1 = 使用位置无关代码 (默认，用于共享库)
```

### 编译步骤

#### 1. x64 Linux/Windows 编译
```bash
./cmake_lib_x64.sh
# 等价于: yang_main_os=0, yang_main_bit=64, yang_main_pic=0
```

#### 2. Android 编译
```bash
./cmake_lib_android.sh
```

#### 3. iOS 编译
```bash
./cmake_lib_ios.sh
```

#### 4. 自定义架构编译
```bash
cd arch
# 执行 yang_build.sh 进行参数化编译
./yang_build.sh [架构] [位数] [PIC]
```

### 编译依赖

**必需库：**
- OpenSSL 或 Mbedtls (安全加密)
- libsrtp (SRTP 实现)
- usrsctp (SCTP/DataChannel)

**可选库：**
- FFmpeg (音视频处理)
- libyuv (视频转换)
- opus (音频编码)
- x264/x265 (视频编码)

---

## 项目结构

```
metaRTC/
├── README.md                    # 项目说明
├── LICENSE                      # 开源协议
│
├── arch/                        # 架构配置目录
│   ├── yang_build.sh           # 架构选择脚本
│   ├── x86_64.cmake
│   ├── aarch64.cmake
│   ├── arm32.cmake
│   ├── mips32.cmake
│   ├── mips64.cmake
│   ├── mac_arm64.cmake
│   └── ios64.cmake
│
├── cmake_lib_x64.sh            # x64 编译脚本
├── cmake_lib_android.sh        # Android 编译脚本
├── cmake_lib_ios.sh            # iOS 编译脚本
│
├── include/                     # 公共头文件目录
│   ├── yang_config.h           # 配置头文件
│   ├── yang_config_os.h        # OS 配置
│   ├── yangaudio/              # 音频接口
│   ├── yangaudiodev/           # 音频设备接口
│   ├── yangavutil/             # 音视频工具接口
│   ├── yangcapture/            # 采集接口
│   ├── yangcodec/              # 编解码接口
│   ├── yangdecoder/            # 解码器接口
│   ├── yangencoder/            # 编码器接口
│   ├── yangjson/               # JSON 接口
│   ├── yangplayer/             # 播放器接口
│   ├── yangpush/               # 推送接口
│   ├── yangrtc/                # RTC 接口
│   ├── yangssl/                # SSL/DTLS 接口
│   └── yangstream/             # 流接口
│
├── bin/                         # 编译输出目录
│   └── app_debug/
│       └── yang_config.ini     # 应用配置
│
├── libyangutil8/               # 基础工具库 (Pure C)
│   ├── CMakeLists.txt
│   ├── cmake_build.sh
│   ├── src/
│   │   ├── yangaudio/          # 音频处理
│   │   ├── yangvideo/          # 视频处理
│   │   ├── yangutil/           # 通用工具
│   │   └── yangjson/           # JSON 处理
│   └── metartccore8.pro
│
├── libmetartccore8/            # WebRTC 核心库 (Pure C)
│   ├── CMakeLists.txt
│   ├── cmake_build.sh
│   ├── src/
│   │   ├── yangice/            # ICE 协议
│   │   ├── yangssl/            # DTLS/SRTP
│   │   ├── yangsdp/            # SDP 处理
│   │   ├── yangrtp/            # RTP 处理
│   │   ├── yangrtc/            # RTC 核心
│   │   ├── yangstream/         # 流处理
│   │   ├── yangpush/           # 推送
│   │   ├── yangpacer/          # 码率控制
│   │   ├── yangutil/           # 系统工具
│   │   └── yangavutil/         # 通用工具
│   └── metartccore8.pro
│
├── libyangwhip8/               # WHIP/WHEP 库 (Pure C)
│   ├── CMakeLists.txt
│   ├── cmake_build.sh
│   └── src/
│       └── yangwhip/           # WHIP/WHEP 实现
│
├── libmetartc8/                # 高级接口库 (C++)
│   ├── CMakeLists.txt
│   ├── cmake_build.sh
│   ├── src/
│   │   ├── yangaudiodev/       # 音频设备 (平台特定)
│   │   │   ├── win/            # Windows 实现
│   │   │   ├── linux/          # Linux 实现
│   │   │   ├── mac/            # macOS 实现
│   │   │   └── android/        # Android 实现
│   │   ├── yangcapture/        # 视频采集 (平台特定)
│   │   │   ├── win/
│   │   │   ├── linux/
│   │   │   ├── mac/
│   │   │   └── android/
│   │   ├── yangencoder/        # 视频编码
│   │   ├── yangdecoder/        # 视频解码
│   │   ├── yangplayer/         # 媒体播放
│   │   ├── yangp2p/            # P2P 管理
│   │   ├── yangp2p2/           # P2P 管理(改进版)
│   │   ├── yangrtc/            # RTC 应用接口
│   │   ├── yangstream/         # 流管理
│   │   ├── yangutil/           # C++ 工具
│   │   ├── yangavutil/         # 音视频工具
│   │   ├── yangandroid/        # Android 模块
│   │   ├── yangmac/            # macOS 模块
│   │   ├── yangrecord/         # 媒体录制 (PC)
│   │   └── yangsrt/            # SRT 协议 (PC)
│   └── metartc8.pro
│
├── demo/                        # 演示项目
│   ├── metapushstream8/        # 推流演示
│   │   └── CMakeLists.txt
│   └── metaplayer8/            # 播放演示
│
├── codec/                       # 编码器实现
│   └── yangwincodec8/          # Windows 编码器
│       ├── YangEncoderGpu.cpp  # GPU 编码
│       ├── codec/
│       │   ├── NvCodec/        # NVIDIA NVENC
│       │   ├── QsvCodec/       # Intel QuickSync
│       │   └── nvenc.cpp
│       └── metartc_codec.pro
│
└── thirdparty/                 # 第三方库
    ├── include/                # 头文件
    ├── user_include/           # 用户包含
    │   ├── openssl/
    │   ├── mbedtls3/
    │   └── ffmpeg/
    └── lib/                    # 库文件
```

---

## 编译流程概览

### 顺序编译关系

```
libyangutil8 (基础工具库)
    ↓
libmetartccore8 (WebRTC 协议栈)  ←依赖 libyangutil8
    ↓
libyangwhip8 (WHIP/WHEP 协议)   ←依赖 libmetartccore8
    ↓
libmetartc8 (高级接口库)         ←依赖以上所有库
    ↓
demo/ (演示应用)                  ←依赖 libmetartc8
```

### 编译特性

**编译优化：**
- 编译优化级别：O2
- 并行编译支持
- 增量编译支持
- 交叉编译支持

**目标产出：**
- 静态库 (.a)
- 共享库 (.so/.dll/.dylib)
- 演示应用 (可执行文件)

---

## 主要配置文件

### yang_config.h
基础配置宏定义：
- 功能开关
- 调试选项
- 性能参数
- 平台定义

### CMakeLists.txt
- 编译规则定义
- 依赖关系配置
- 编译标志设置
- 源文件收集

### cmake_build.sh
每个模块的编译脚本：
- 参数化编译
- 架构选择
- 输出目录管理

---

## 技术栈概览

| 层级 | 库 | 语言 | 功能 |
|-----|-----|------|------|
| 应用 | libmetartc8 | C++ | 高级接口、采集编解码 |
| 协议 | libmetartccore8 | C | WebRTC 协议栈 |
| 协议 | libyangwhip8 | C | WHIP/WHEP 协议 |
| 基础 | libyangutil8 | C | 音视频处理、工具库 |

## 关键依赖

- **安全加密：** OpenSSL / Mbedtls
- **媒体传输：** libsrtp
- **数据通道：** usrsctp
- **视频处理：** FFmpeg (可选)
- **硬件编码：** NVIDIA NVENC / Intel QSV / Apple VideoToolbox

---

*本文档基于 MetaRTC 项目结构生成，2026年4月27日*
