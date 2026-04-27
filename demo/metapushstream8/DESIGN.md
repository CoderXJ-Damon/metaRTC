# metapushstream8 应用设计文档

**版本**: 1.0  
**最后更新**: 2026-04-27  
**应用**: Qt 音视频推流客户端

---

## 目录

- [1. 应用概述](#1-应用概述)
- [2. 整体架构](#2-整体架构)
- [3. 初始化流程](#3-初始化流程)
- [4. 消息驱动架构](#4-消息驱动架构)
- [5. 推流核心流程](#5-推流核心流程)
- [6. 数据流向](#6-数据流向)
- [7. 关键组件详解](#7-关键组件详解)
- [8. 用户交互流程](#8-用户交互流程)
- [9. 编码和网络配置](#9-编码和网络配置)

---

## 1. 应用概述

**metapushstream8** 是一个基于Qt框架的实时音视频推流应用，实现了以下核心功能：

- ✅ **多源视频捕获**：支持摄像头、屏幕、外部接口
- ✅ **实时编码**：H.264/H.265 编码，支持GPU硬件加速
- ✅ **WebRTC推流**：推送到支持 WHIP/WHEP 协议的媒体服务器
- ✅ **多协议支持**：兼容 SRS、ZLM 等流媒体服务器
- ✅ **本地预览**：实时显示采集内容
- ✅ **音频支持**：Opus编码，48kHz采样，支持立体声

---

## 2. 整体架构

### 2.1 系统组件图

```
┌──────────────────────────────────────────────────────────────────┐
│                        RecordMainWindow (Qt GUI)                 │
│                         主窗口 + 业务逻辑                         │
└────────────┬─────────────────────────────────┬───────────────────┘
             │                                  │
             ▼                                  ▼
    ┌──────────────────┐           ┌──────────────────────┐
    │YangRecordThread  │           │YangPushMessageHandle │
    │  (渲染线程)       │           │  (消息处理线程)      │
    │ 20ms刷新本地预览 │           │  系统事件处理        │
    └────────┬─────────┘           └──────────┬───────────┘
             │                                 │
             │                    ┌────────────▼──────────┐
             │                    │YangPushHandleImpl      │
             │                    │  (推流核心实现)       │
             │                    └────────────┬──────────┘
             │                                 │
             │          ┌──────────────────────┼──────────┐
             │          │                      │          │
             ▼          ▼                      ▼          ▼
        ┌─────────┐ ┌─────────┐          ┌─────────┐ ┌─────────┐
        │  Play   │ │  Video  │          │ Audio   │ │ WebRTC  │
        │ Widget  │ │ Capture │          │ Capture │ │ Publish │
        │ (预览)  │ │& Encoder│          │& Encoder│ │ (推流)  │
        └─────────┘ └─────────┘          └─────────┘ └────┬────┘
             ▲                                            │
             │                                    ┌───────▼────────┐
             │                                    │YangRtcPublish  │
             │                                    │(WebRTC Protocol)
             │                                    └────────┬───────┘
             └────────────────────────────────────────────┼────────┐
                                                          │        │
                                        ┌─────────────────▼─┐     │
                                        │  媒体服务器网络   │     │
                                        │ WHIP/SRS/ZLM等   │     │
                                        └───────────────────┘     │
                                                                  │
                                        ┌─────────────────────┐   │
                                        │  本地视频缓冲区     │   │
                                        │YangVideoBuffer      │───┘
                                        └─────────────────────┘
```

### 2.2 核心类关系

```
RecordMainWindow
  ├─ YangContext (配置和参数)
  ├─ YangPushFactory (工厂类)
  ├─ YangSysMessageHandle* m_message (消息线程)
  │   └─ YangPushMessageHandle
  │       └─ YangPushHandleImpl
  │           ├─ YangPushPublish (捕获和编码)
  │           │   ├─ YangPushCapture (捕获)
  │           │   └─ YangPushEncoder (编码)
  │           └─ YangRtcPublish (WebRTC推送)
  │               └─ YangPeerConnection8 (网络连接)
  └─ YangRecordThread (渲染线程)
      └─ YangPlayWidget (本地预览窗口)
```

---

## 3. 初始化流程

### 3.1 应用启动流程（main.cpp）

```
int main(int argc, char *argv[])
├─ 创建 QApplication
├─ 设置 Qt 字符编码 (UTF-8)
├─ 创建 RecordMainWindow
│  ├─ 初始化 YangContext (加载 yang_config.ini)
│  ├─ 设置编码器格式 (YangI420)
│  ├─ 设置 GPU 编码标志 (NVIDIA/Intel/AMD)
│  ├─ 初始化 UI 组件
│  ├─ 创建本地预览窗口 (YangPlayWidget)
│  └─ 获取本地 IP 地址
│
├─ 创建 YangPushFactory
├─ 创建 YangSysMessageHandle* (消息处理线程)
├─ 启动消息线程 sysmessage->start()
│
├─ 创建 YangRecordThread (视频渲染线程)
├─ 初始化渲染线程参数
├─ 启动渲染线程 videoThread.start()
│
├─ 显示主窗口 w.show()
├─ 等待 200ms
├─ 初始化预览 w.initPreview()
│
└─ 进入 Qt 事件循环 a.exec()
```

### 3.2 主窗口初始化（recordmainwindow.cpp:20-90）

#### 步骤 1: YangContext 配置

```cpp
YangContext *m_context = new YangContext();
m_context->init("yang_config.ini");

// 编码格式设置
m_context->avinfo.video.videoEncoderFormat = YangI420;

// GPU 编码配置 (可选)
#if Yang_Enable_GPU_Encoding
    m_context->avinfo.video.videoEncHwType = YangV_Hw_Nvdia;
#endif
```

#### 步骤 2: 音频配置

```cpp
m_context->avinfo.audio.enableMono = yangfalse;           // 立体声
m_context->avinfo.audio.sample = 48000;                  // 48kHz 采样
m_context->avinfo.audio.channel = 2;                     // 双声道
m_context->avinfo.audio.enableAec = yangfalse;           // 无回声抑制
m_context->avinfo.audio.audioEncoderType = Yang_AED_OPUS; // Opus 编码
```

#### 步骤 3: 缓冲区配置

```cpp
// 音频缓冲
m_context->avinfo.audio.audioCacheNum = 8;
m_context->avinfo.audio.audioPlayCacheNum = 8;

// 视频缓冲
m_context->avinfo.video.videoCacheNum = 10;
m_context->avinfo.video.evideoCacheNum = 10;
m_context->avinfo.video.videoPlayCacheNum = 10;
```

#### 步骤 4: 视频参数配置

```cpp
m_videoType = Yang_VideoSrc_Camera;  // 默认摄像头
m_context->avinfo.video.videoEncoderType = Yang_VED_H264; // H264 编码
m_context->avinfo.enc.enc_threads = 4;  // 4 线程编码
```

#### 步骤 5: 网络配置

```cpp
m_context->avinfo.rtc.rtcLocalPort = 10000 + yang_random()%15000;  // 随机端口
m_context->avinfo.rtc.iceCandidateType = YangIceHost;
m_context->avinfo.audio.enableAudioFec = yangfalse;  // SRS 不使用 Audio FEC
```

### 3.3 初始化预览（initPreview）

```cpp
void RecordMainWindow::initPreview() {
    if(m_videoType == Yang_VideoSrc_Screen) {
        yang_post_message(YangM_Push_StartScreenCapture, 0, NULL);
    } else if(m_videoType == Yang_VideoSrc_Camera) {
        yang_post_message(YangM_Push_StartVideoCapture, 0, NULL);
    } else if(m_videoType == Yang_VideoSrc_OutInterface) {
        yang_post_message(YangM_Push_StartOutCapture, 0, NULL);
    }
}
```

---

## 4. 消息驱动架构

### 4.1 消息系统概述

应用使用**异步消息队列**解耦 UI 层和业务逻辑：

```
用户操作 → 发送消息 → 消息队列 → YangSysMessageHandle 处理 → 业务逻辑执行 → 回调结果
```

### 4.2 核心消息类型

| 消息ID | 名称 | 用途 | 处理类 |
|-------|------|------|-------|
| `YangM_Push_StartVideoCapture` | 启动视频捕获 | 初始化摄像头采集 | YangPushHandleImpl |
| `YangM_Push_StartScreenCapture` | 启动屏幕捕获 | 初始化屏幕采集 | YangPushHandleImpl |
| `YangM_Push_StartOutCapture` | 启动外部接口 | 初始化外部数据源 | YangPushHandleImpl |
| `YangM_Push_Connect` | WebRTC 推流 | 推流到非WHIP服务器(SRS/ZLM) | YangPushHandleImpl::publish() |
| `YangM_Push_Connect_Whip` | WHIP 推流 | 推流到WHIP服务器 | YangPushHandleImpl::publish() |
| `YangM_Push_Disconnect` | 断开连接 | 停止推流 | YangPushHandleImpl::disconnect() |
| `YangM_Push_SwitchToCamera` | 切换摄像头 | 从屏幕切换到摄像头 | YangPushHandleImpl |
| `YangM_Push_SwitchToScreen` | 切换屏幕 | 从摄像头切换到屏幕 | YangPushHandleImpl |

### 4.3 YangPushMessageHandle 消息处理

**文件**: `demo/metapushstream8/yangpush/YangPushMessageHandle.cpp`

```cpp
class YangPushMessageHandle : public YangSysMessageHandle {
public:
    YangPushHandleImpl* m_push;  // 核心推流实现
    
    void handleMessage(YangSysMessage *mss) {
        switch (mss->messageId) {
        case YangM_Push_StartVideoCapture:
            if(m_push) m_push->changeSrc(Yang_VideoSrc_Camera, false);
            break;
            
        case YangM_Push_StartScreenCapture:
            if(m_push) m_push->changeSrc(Yang_VideoSrc_Screen, false);
            break;
            
        case YangM_Push_Connect:
            if(mss->user && m_push) 
                ret = pushPublish((char*)mss->user, yangfalse);
            break;
            
        case YangM_Push_Connect_Whip:
            if(mss->user && m_push) 
                ret = pushPublish((char*)mss->user, yangtrue);
            break;
            
        case YangM_Push_Disconnect:
            if(m_push) m_push->disconnect();
            break;
        }
        
        // 回调结果到 UI
        if (mss->handle) {
            if (ret) mss->handle->failure(ret);
            else mss->handle->success();
        }
    }
};
```

### 4.4 消息回调机制

```cpp
void RecordMainWindow::receiveSysMessage(YangSysMessage *mss, int32_t err) {
    switch (mss->messageId) {
    case YangM_Push_Connect:
        if(err) {
            ui->m_b_rec->setText("开始");
            ui->m_l_err->setText("push error(" + QString::number(err) + ")!");
        }
        break;
        
    case YangM_Push_StartVideoCapture:
    case YangM_Push_StartScreenCapture:
        // 获取视频预处理缓冲区用于本地预览
        m_rt->m_videoBuffer = m_pushfactory.getPreVideoBuffer(m_message);
        break;
    }
}
```

---

## 5. 推流核心流程

### 5.1 推流启动（YangPushHandleImpl::publish）

**文件**: `demo/metapushstream8/yangpush/YangPushHandleImpl.cpp:110-171`

这是应用的**心脏**，负责完整的推流初始化。执行流程如下：

```
┌──────────────────────────────────────┐
│ 步骤 1: URL 解析                      │
│ yang_url_parse() 提取服务器和资源信息 │
└────────────┬─────────────────────────┘
             ▼
┌──────────────────────────────────────┐
│ 步骤 2: 停止之前的推流                │
│ stopPublish()                        │
│ - 断开媒体服务器                     │
│ - 删除 WebRTC 连接                   │
│ - 停止视频编码                       │
└────────────┬─────────────────────────┘
             ▼
┌──────────────────────────────────────┐
│ 步骤 3: 创建 YangRtcPublish 对象       │
│ 用于 WebRTC 协议处理和网络传输        │
└────────────┬─────────────────────────┘
             ▼
┌──────────────────────────────────────┐
│ 步骤 4: 启动音频处理（可选）         │
│ - startAudioCapture()                │
│ - initAudioEncoding()                │
│ - startAudioEncoding()               │
└────────────┬─────────────────────────┘
             ▼
┌──────────────────────────────────────┐
│ 步骤 5: 启动视频处理                 │
│ - initVideoEncoding()                │
│ - m_cap->setRtcNetBuffer()           │
│ - startVideoEncoding()               │
│ - startVideoCaptureState()           │
└────────────┬─────────────────────────┘
             ▼
┌──────────────────────────────────────┐
│ 步骤 6: 初始化 WebRTC 连接            │
│ - m_rtcPub->init(url, isWhip)       │
│ - m_rtcPub->start()                 │
│ 连接媒体服务器，开始推流             │
└──────────────────────────────────────┘
```

### 5.2 伪代码实现

```cpp
int YangPushHandleImpl::publish(char* url, yangbool isWhip) {
    // 1. URL 解析
    memset(&m_url, 0, sizeof(m_url));
    if(!isWhip) {
        if(yang_url_parse(m_context->avinfo.sys.familyType, url, &m_url) != Yang_Ok) {
            return 1;  // URL 解析失败
        }
    }
    
    // 2. 停止之前的推流
    stopPublish();
    
    // 3. 创建 WebRTC 推流对象
    if (m_rtcPub == NULL) {
        m_rtcPub = new YangRtcPublish(m_context);
    }
    
    // 4. 启动音频处理
    if(m_hasAudio) {
        m_hasAudio = bool(m_cap->startAudioCapture() == Yang_Ok);
    }
    if (m_hasAudio) {
        m_cap->initAudioEncoding();
        m_cap->startAudioEncoding();
    }
    
    // 5. 启动视频处理
    m_cap->initVideoEncoding();
    m_cap->setRtcNetBuffer(m_rtcPub);
    m_cap->startVideoEncoding();
    
    // 6. 初始化 WebRTC 连接
    int err;
    if(isWhip) {
        err = m_rtcPub->init(url, yangtrue);  // WHIP 协议
    } else {
        err = m_rtcPub->init(url, yangfalse); // 其他协议
    }
    
    if (err != Yang_Ok) {
        return yang_error_wrap(err, "connect server failure!");
    }
    
    // 7. 启动推流线程
    m_rtcPub->start();
    
    // 8. 启动采集
    if (m_hasAudio) {
        m_cap->startAudioCaptureState();
    }
    if (m_videoState == Yang_VideoSrc_Camera) {
        m_cap->startVideoCaptureState();
    }
    
    return err;
}
```

### 5.3 断开连接（disconnect）

```cpp
void YangPushHandleImpl::disconnect() {
    if (m_cap) {
        if(m_hasAudio) m_cap->stopAudioCaptureState();
        m_cap->stopVideoCaptureState();
        m_cap->stopScreenCaptureState();
    }
    stopPublish();
}
```

---

## 6. 数据流向

### 6.1 视频数据流（推流路径）

```
┌─────────────────────────────────────────────────────────────────┐
│                      视频源                                      │
│  ├─ 摄像头 (Yang_VideoSrc_Camera)                              │
│  ├─ 屏幕   (Yang_VideoSrc_Screen)                              │
│  └─ 外部   (Yang_VideoSrc_OutInterface)                        │
└────────────────────┬────────────────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              YangPushCapture (视频捕获)                          │
│  - 从摄像头/屏幕捕获原始视频帧                                   │
│  - 支持多种视频格式 (NV12, I420, ARGB 等)                      │
└────────────────────┬────────────────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              YangVideoBuffer (原始视频缓冲)                      │
│  - 存储原始视频帧                                                │
│  - 供本地预览和编码使用                                         │
│  - 双缓冲机制，避免丢帧                                          │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ├─────────────────────────┐
                     │                         │
                     ▼                         ▼
        ┌──────────────────────┐  ┌──────────────────────┐
        │   本地预览            │  │   编码器              │
        │ YangPlayWidget       │  │ YangPushEncoder     │
        │ (本地显示)            │  │ H.264/H.265 编码    │
        └──────────────────────┘  └──────────────────────┘
                     ▲                         │
                     │                         ▼
                     │              ┌──────────────────────┐
                     │              │YangVideoEncoderBuffer│
                     │              │  (编码视频缓冲)      │
                     │              │  RTP 数据包          │
                     │              └──────────────────────┘
                     │                         │
                     │                         ▼
                     │              ┌──────────────────────┐
                     │              │  YangRtcPublish     │
                     │              │ (WebRTC 协议处理)    │
                     │              │ - ICE 候选地址       │
                     │              │ - SDP 交换           │
                     │              │ - 包装 RTP/RTCP      │
                     │              └──────────────────────┘
                     │                         │
                     │                         ▼
                     │              ┌──────────────────────┐
                     │              │ YangPeerConnection8 │
                     │              │  (网络发送)          │
                     │              │  UDP/DTLS/SRTP       │
                     │              └──────────────────────┘
                     │                         │
                     └─────────────────────────┼────────────────┐
                                              │                │
                                    ┌─────────▼────────┐     │
                                    │ 媒体服务器       │     │
                                    │ (WHIP/SRS/ZLM)  │     │
                                    └──────────────────┘     │
                                              │              │
                                              ▼              │
                                     ┌──────────────────┐   │
                                     │  流媒体处理      │   │
                                     │  转码/转发/录制  │   │
                                     └──────────────────┘   │
                                                            │
                                        ┌───────────────────┘
                                        │
                                        ▼
                              ┌──────────────────────┐
                              │    其他观众/应用      │
                              │   (拉取媒体流)       │
                              └──────────────────────┘
```

### 6.2 音频数据流

```
┌──────────────────────────────────────────────┐
│        音频源                                │
│  - 系统音频设备 (麦克风)                     │
└────────────┬─────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────┐
│    YangPushCapture                          │
│    startAudioCapture()                      │
│    获取原始 PCM 音频                        │
└────────────┬─────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────┐
│    YangAudioBuffer                          │
│    原始音频缓冲 (PCM 48kHz, 2ch)            │
└────────────┬─────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────┐
│    Opus 编码器                              │
│    initAudioEncoding()                      │
│    startAudioEncoding()                     │
│    编码成 Opus 比特流                       │
└────────────┬─────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────┐
│    YangAudioEncoderBuffer                   │
│    编码音频缓冲 (Opus RTP)                  │
└────────────┬─────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────┐
│    YangRtcPublish                           │
│    打包 RTP 数据包                          │
└────────────┬─────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────┐
│    网络传输到媒体服务器                      │
└──────────────────────────────────────────────┘
```

### 6.3 本地预览流

```
原始视频缓冲 (YangVideoBuffer)
          ▼
┌─────────────────────────────────────────────┐
│      YangRecordThread (渲染线程)            │
│      每 20ms 执行一次 render()              │
│      - 检查缓冲区是否有新帧                 │
│      - getVideoRef() 获取帧数据             │
│      - 检查 width > 0 确保有效               │
└─────────────────────────────────────────────┘
          ▼
┌─────────────────────────────────────────────┐
│      YangPlayWidget                         │
│      playVideo(buffer, width, height)       │
│      使用 OpenGL/Direct3D 渲染显示          │
└─────────────────────────────────────────────┘
          ▼
┌─────────────────────────────────────────────┐
│      本地窗口显示                            │
│      用户可实时看到采集内容                  │
└─────────────────────────────────────────────┘
```

---

## 7. 关键组件详解

### 7.1 YangPushPublish（捕获和编码）

**文件**: `include/yangpush/YangPushPublish.h`

```cpp
class YangPushPublish {
public:
    // 捕获控制
    void setCaptureType(int captureType);        // 设置视频源
    void startCamera();                          // 启动摄像头
    void stopCamera();                           // 停止摄像头
    void setScreenInterval(int32_t interval);    // 屏幕采样间隔
    void setDrawmouse(bool isDraw);              // 屏幕是否绘制鼠标
    
    // 编码控制
    void initAudioEncoding();                    // 初始化音频编码
    void initVideoEncoding();                    // 初始化视频编码
    void startAudioEncoding();                   // 启动音频编码
    void startVideoEncoding();                   // 启动视频编码
    void deleteVideoEncoding();                  // 停止视频编码
    
    // 采集控制
    int32_t startAudioCapture();                 // 启动音频采集
    int32_t startVideoCapture();                 // 启动视频采集
    void startAudioCaptureState();               // 激活音频采集
    void startVideoCaptureState();               // 激活视频采集
    void stopAudioCaptureState();                // 停止音频采集
    void stopVideoCaptureState();                // 停止视频采集
    void stopScreenCaptureState();               // 停止屏幕采集
    
    // 缓冲区管理
    YangVideoBuffer* getPreVideoBuffer();        // 获取预处理视频缓冲
    YangVideoBuffer* getOutPreVideoBuffer();     // 获取外部预处理视频缓冲
    YangVideoBuffer* getOutVideoBuffer();        // 获取外部视频缓冲
    
    // 其他
    void setVideoInfo(YangVideoInfo *pvideo);    // 设置视频参数
    void setRtcNetBuffer(YangRtcPublish *prr);   // 设置网络缓冲区
    YangPushCapture* getPushCapture();           // 获取采集器
};
```

### 7.2 YangRtcPublish（WebRTC 推流）

**文件**: `include/yangpush/YangRtcPublish.h`

```cpp
class YangRtcPublish : public YangThread, public YangCallbackRtc {
public:
    // 初始化和连接
    int32_t init(char* url, yangbool isWhip);
    int32_t connectMediaServer();
    int32_t disConnectMediaServer();
    
    // 数据流设置
    void setInVideoMetaData(YangVideoMeta *pvmd);
    void setInAudioList(YangAudioEncoderBuffer *pbuf);
    void setInVideoList(YangVideoEncoderBuffer *pbuf);
    
    // 配置和控制
    void setMediaConfig(int32_t uid, YangAudioParam* audio, YangVideoParam* video);
    void sendRequest(int32_t uid, uint32_t ssrc, YangRequestType req);
    
    // 流传输
    int32_t stopPublishAudioData();
    int32_t stopPublishVideoData();
    int32_t publishMsg(YangFrame* msgFrame);
    
    // 线程控制
    void start();     // 继承自 YangThread
    void stop();
    void run();       // 线程主函数
    
private:
    std::vector<YangPeerConnection8*> m_pushs;  // WebRTC 连接对象
    YangVideoEncoderBuffer *m_in_videoBuffer;   // 输入视频缓冲
    YangAudioEncoderBuffer *m_in_audioBuffer;   // 输入音频缓冲
    YangVideoMeta *m_vmd;                       // 视频元数据
};
```

### 7.3 YangRecordThread（本地预览）

**文件**: `demo/metapushstream8/video/yangrecordthread.cpp`

```cpp
class YangRecordThread : public QThread {
public:
    YangVideoBuffer *m_videoBuffer;  // 视频缓冲区指针
    YangPlayWidget *m_video;         // 预览窗口指针
    
    void initPara(YangContext *pcontext);  // 初始化参数
    void stopAll();                        // 停止线程
    
protected:
    void run() {                           // 线程主循环
        m_isLoop = 1;
        m_isStart = 1;
        
        while(m_isLoop) {
            QThread::msleep(20);           // 20ms 周期（50FPS）
            render();
        }
        
        m_isStart = 0;
    }
    
    void render() {
        if(m_videoBuffer && m_videoBuffer->size() > 0) {
            uint8_t* frame = m_videoBuffer->getVideoRef(&m_frame);
            
            if(frame && m_video && m_videoBuffer->m_width > 0) {
                m_video->playVideo(frame, 
                                 m_videoBuffer->m_width, 
                                 m_videoBuffer->m_height);
            }
        }
    }
};
```

---

## 8. 用户交互流程

### 8.1 启动推流的完整流程

```
用户点击 [开始] 按钮
    ▼
on_m_b_rec_clicked() 回调
    ▼
判断是否已启动推流
    ├─ 未启动 (m_isStartpush == 0)
    │   ├─ 读取 URL 文本框内容
    │   ├─ 检查是否选中 WHIP 协议
    │   ├─ 发送消息：YangM_Push_Connect 或 YangM_Push_Connect_Whip
    │   ├─ 设置按钮文本为 "停止"
    │   └─ 设置 m_isStartpush = 1
    │
    └─ 已启动 (m_isStartpush == 1)
        ├─ 发送消息：YangM_Push_Disconnect
        ├─ 设置按钮文本为 "开始"
        └─ 设置 m_isStartpush = 0
```

### 8.2 协议选择流程

**WHIP 协议选择** (`on_m_c_whip_clicked`)

```
用户勾选 WHIP 复选框
    ▼
设置媒体服务器类型：Yang_Server_Whip_Whep
    ▼
组织 WHIP URL
    ▼
URL 格式：http://<local_ip>:1985/rtc/v1/whip/?app=live&stream=livestream

发送推流请求时使用 isWhip=true
```

**其他协议选择** (未勾选 WHIP)

```
选项 1: SRS 协议
    URL 格式：webrtc://<local_ip>:1985/live/livestream
    媒体服务器：Yang_Server_Srs

选项 2: ZLM 协议 (默认)
    URL 格式：webrtc://<local_ip>:80/live/test
    媒体服务器：Yang_Server_Zlm
```

### 8.3 错误处理

```
推流错误回调 receiveSysMessage(YangM_Push_Connect, errCode)
    ▼
判断是否有错误
    ├─ 有错误 (err != 0)
    │   ├─ 重置按钮为 "开始"
    │   ├─ 显示错误信息标签："push error(errCode)!"
    │   └─ 重置 m_isStartpush = 0
    │
    └─ 无错误
        └─ 推流成功，继续传输数据
```

---

## 9. 编码和网络配置

### 9.1 视频编码配置

```cpp
// 编码器选择
m_context->avinfo.video.videoEncoderType = Yang_VED_H264;  // H.264
// 或
m_context->avinfo.video.videoEncoderType = Yang_VED_H265;  // H.265

// 编码格式
m_context->avinfo.video.videoEncoderFormat = YangI420;     // 原始格式
m_context->avinfo.video.videoCaptureFormat = YangNv12;     // 采集格式

// GPU 硬件加速
#if Yang_Enable_GPU_Encoding
    m_context->avinfo.video.videoEncHwType = YangV_Hw_Nvdia;  // NVIDIA
    // 或
    m_context->avinfo.video.videoEncHwType = YangV_Hw_Intel;  // Intel QSV
    // 或
    m_context->avinfo.video.videoEncHwType = YangV_Hw_Amd;    // AMD
#endif

// 编码线程数
m_context->avinfo.enc.enc_threads = 4;
```

### 9.2 音频编码配置

```cpp
// 编码器
m_context->avinfo.audio.audioEncoderType = Yang_AED_OPUS;

// 采样率和声道
m_context->avinfo.audio.sample = 48000;      // 48 kHz
m_context->avinfo.audio.channel = 2;         // 立体声

// FEC (前向纠错)
m_context->avinfo.audio.enableAudioFec = yangfalse;  // SRS 不使用

// 声学处理
m_context->avinfo.audio.enableMono = yangfalse;  // 不转为单声道
m_context->avinfo.audio.enableAec = yangfalse;   // 无回声消除
```

### 9.3 网络配置

```cpp
// RTC 本地端口
m_context->avinfo.rtc.rtcLocalPort = 10000 + yang_random()%15000;

// ICE 候选地址类型
m_context->avinfo.rtc.iceCandidateType = YangIceHost;  // 仅本机地址
// 或
m_context->avinfo.rtc.iceCandidateType = YangIceAll;   // 所有候选地址

// 媒体服务器类型
m_context->avinfo.sys.mediaServer = Yang_Server_Whip_Whep;  // WHIP
// 或
m_context->avinfo.sys.mediaServer = Yang_Server_Srs;       // SRS
// 或
m_context->avinfo.sys.mediaServer = Yang_Server_Zlm;       // ZLM

// 网络传输类型 (由 URL 自动设置)
m_context->avinfo.sys.transType = m_url.netType;
m_context->avinfo.sys.familyType = AF_INET;  // IPv4 / AF_INET6
```

### 9.4 缓冲区配置

```cpp
// 音频缓冲
m_context->avinfo.audio.audioCacheNum = 8;           // 采集缓冲数
m_context->avinfo.audio.audioPlayCacheNum = 8;       // 播放缓冲数
m_context->avinfo.audio.audioCacheSize = 8;          // 缓冲大小

// 视频缓冲
m_context->avinfo.video.videoCacheNum = 10;         // 原始视频缓冲
m_context->avinfo.video.evideoCacheNum = 10;        // 编码视频缓冲
m_context->avinfo.video.videoPlayCacheNum = 10;     // 预览缓冲
```

---

## 10. 状态管理

### 10.1 推流状态

```cpp
int32_t m_isStartpush = 0;   // 推流状态标志
    // 0: 未推流 → 1: 推流中

bool m_isStartRecord = false;  // 录制状态标志
    // false: 未录制 → true: 录制中
```

### 10.2 视频源状态

```cpp
int m_videoState;   // 当前视频源类型
    // Yang_VideoSrc_Camera        - 摄像头
    // Yang_VideoSrc_Screen        - 屏幕
    // Yang_VideoSrc_OutInterface  - 外部接口
```

---

## 附录 A: 快速参考

### 快速启动步骤

1. **配置文件**: `bin/app_debug/yang_config.ini`
2. **编译**: 使用 CMake 或 Visual Studio 项目文件
3. **运行**: 执行 metapushstream8 应用
4. **输入 URL**: 填写媒体服务器地址
5. **选择协议**: 勾选 WHIP 或选择 SRS/ZLM
6. **点击开始**: 开始推流

### 常见错误代码

| 错误码 | 含义 | 解决方案 |
|------|------|--------|
| 1 | URL 解析失败 | 检查 URL 格式是否正确 |
| 2 | 网络连接失败 | 检查网络连接和防火墙 |
| 3 | 媒体服务器不响应 | 检查服务器是否在线 |
| 4 | 编码初始化失败 | 检查硬件编码器是否可用 |

---

**文档完成** ✓