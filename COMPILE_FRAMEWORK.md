# MetaRTC 编译框架详细指南

## 编译架构设计

### 1. 分层设计

```
┌─────────────────────────────────────────────┐
│        应用层 (Application Layer)           │
│  Demo / 用户应用 (libmetartc8 + 业务逻辑)   │
└────────────────┬────────────────────────────┘
                 │ 依赖
┌────────────────▼────────────────────────────┐
│        接口层 (Interface Layer)             │
│  libmetartc8 (C++ 高级接口)                 │
│  - 采集/编解码/播放                        │
│  - 平台适配                                 │
│  - 应用集成                                │
└────────────────┬────────────────────────────┘
                 │ 依赖
┌────────────────▼────────────────────────────┐
│        核心协议层 (Core Protocol Layer)     │
│  ┌──────────────────┬──────────────────┐   │
│  │ libmetartccore8  │   libyangwhip8   │   │
│  │ (WebRTC 栈)      │  (推拉流协议)    │   │
│  └──────────┬───────┴──────────────────┘   │
│             │ 依赖                         │
└─────────────┼────────────────────────────────┘
              │
┌─────────────▼────────────────────────────────┐
│        基础工具层 (Utility Layer)             │
│  libyangutil8 (Pure C 工具库)               │
│  - 音视频处理                               │
│  - 编解码基础                              │
│  - JSON/数据结构                            │
└──────────────────────────────────────────────┘
```

### 2. 编译参数体系

#### 架构参数 (Architecture)
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 值  | 架构名称      | 特点           | 常见应用
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 0   | x86/x64      | PC 架构        | 桌面开发、测试
 1   | MIPS         | 网络设备架构   | 路由器、网关
 2   | ARM          | 移动设备架构   | 手机、平板
 3   | RISC-V       | 开源架构       | 教育、科研
 4   | Loongson     | 龙芯架构       | 国产 CPU
 5   | WRT          | 嵌入式 Linux   | 网络设备
 6   | macOS        | 苹果 PC 架构   | Mac 开发
 7   | iOS          | 苹果移动架构   | iPhone、iPad
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 位数参数 (Bitness)
```
32位编译：適用于低功耗设备、内存受限环境
64位编译：適用于现代 PC、服务器、高端移动设备
```

#### PIC 参数 (Position Independent Code)
```
PIC = 0 : 静态编译模式
     - 适用于：可执行文件、静态库链接
     - 优点：执行速度快、不依赖重定位表
     - 缺点：无法用于动态库

PIC = 1 : 动态编译模式
     - 适用于：共享库、动态库
     - 优点：支持动态加载、ASLR 安全特性
     - 缺点：有轻微性能开销
```

### 3. 编译脚本层次

#### 第一层：顶级编译脚本
```bash
# cmake_lib_x64.sh (用于 x86_64 Linux/Windows)
# cmake_lib_android.sh (用于 Android)
# cmake_lib_ios.sh (用于 iOS)

特点：
✓ 一键式编译
✓ 自动调用所有模块
✓ 固定参数配置
✓ 适合快速编译
```

#### 第二层：模块编译脚本
```bash
# 每个模块目录下的 cmake_build.sh

调用方式：
./cmake_build.sh <架构> <位数> <PIC>

特点：
✓ 单个模块编译
✓ 支持参数化
✓ 可独立调用
✓ 适合开发调试
```

#### 第三层：CMake 配置文件
```bash
# CMakeLists.txt (每个模块)

特点：
✓ 详细的编译规则
✓ 依赖管理
✓ 平台适配
✓ 编译选项控制
```

---

## 编译流程详解

### X86_64 编译流程

```
执行: ./cmake_lib_x64.sh
     ↓
[参数设置]
 yang_main_os=0      (Linux/Windows)
 yang_main_bit=64    (64位)
 yang_main_pic=0     (不使用PIC)
     ↓
[编译 libyangutil8]
 进入 libyangutil8 目录
 执行 ./cmake_build.sh 0 64 0
 生成: build/libyangtil8.a (静态库)
     ↓
[编译 libmetartccore8] 
 进入 libmetartccore8 目录
 执行 ./cmake_build.sh 0 64 0
 依赖: libyangutil8
 生成: build/libmetartccore8.a
     ↓
[编译 libyangwhip8]
 进入 libyangwhip8 目录
 执行 ./cmake_build.sh 0 64 0
 依赖: libmetartccore8
 生成: build/libyangwhip8.a
     ↓
[编译 libmetartc8]
 进入 libmetartc8 目录
 执行 ./cmake_build.sh 0 64 0
 依赖: 上述所有库
 生成: build/libmetartc8.a
     ↓
[编译完成]
 输出目录: bin/lib_x64/
 包含库文件: *.a (静态库)
```

### Android 编译流程

```
执行: ./cmake_lib_android.sh
     ↓
[进入各模块目录]
 依次执行: ./cmake_android.sh
     ↓
[NDK 工具链配置]
 检测 ANDROID_NDK_ROOT
 配置 C/C++ 编译器
 设置 ABI (armeabi-v7a, arm64-v8a, x86, x86_64)
     ↓
[增量编译各模块]
 libyangutil8 → libmetartccore8 → libyangwhip8 → libmetartc8
     ↓
[编译输出]
 bin/lib_android/<ABI>/lib*.a
```

---

## 各模块编译细节

### libyangutil8 编译配置

```cmake
# CMakeLists.txt 片段
project(libyangutil8)
set(CMAKE_BUILD_TYPE "RELEASE")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2 -std=gnu11")

# 源文件收集
aux_source_directory(./src/yangaudio DIR_SRCS)
aux_source_directory(./src/yangvideo DIR_SRCS)
aux_source_directory(./src/yangutil DIR_SRCS)
aux_source_directory(./src/yangjson DIR_SRCS)

# 生成库文件
add_library(yangutil8 ${DIR_SRCS})
```

**头文件依赖：**
```
${HOME_BASE}/include           # 公共接口
${HOME_BASE}/thirdparty/include # 第三方库
${HOME_BASE}/libyangutil8/src   # 内部实现
```

**编译标志：**
- `-O2` : 中等优化
- `-std=gnu11` : GNU C11 标准
- 启用所有机器相关优化

### libmetartccore8 编译配置

```cmake
project(libmetartccore8)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2 -std=gnu11")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -std=gnu++11")

# 多个源目录
aux_source_directory(./src/yangutil/sys DIR_SRCS)
aux_source_directory(./src/yangutil/buffer2 DIR_SRCS)
aux_source_directory(./src/yangice DIR_SRCS)
aux_source_directory(./src/yangssl DIR_SRCS)
aux_source_directory(./src/yangsdp DIR_SRCS)
aux_source_directory(./src/yangrtp DIR_SRCS)
aux_source_directory(./src/yangrtc DIR_SRCS)
# ... 更多源目录

# 包含第三方库头文件
include_directories(${HOME_BASE}/thirdparty/user_include/openssl)
include_directories(${HOME_BASE}/thirdparty/user_include/mbedtls3)

# 生成库文件
add_library(metartccore8 ${DIR_SRCS})
```

**关键依赖：**
- OpenSSL 或 Mbedtls（DTLS/SRTP）
- libsrtp（SRTP 实现）
- usrsctp（DataChannel）

**平台特定配置：**
```cmake
if (CMAKE_SYSTEM_NAME MATCHES "Windows")
    add_definitions(-D_TIMESPEC_DEFINED)
    include_directories(${HOME_BASE}/thirdparty/include/win/include)
endif()
```

### libmetartc8 编译配置

```cmake
project(libmetartc8)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -std=gnu++11")

# 条件编译 - 可选功能开关
if (NOT NoCapture)
    aux_source_directory(./src/yangcapture DIR_SRCS)
    aux_source_directory(./src/yangencoder DIR_SRCS)
    aux_source_directory(./src/yangdecoder DIR_SRCS)
endif()

if (NOT NoPlayer)
    aux_source_directory(./src/yangplayer DIR_SRCS)
endif()

# 平台特定模块
if (CMAKE_SYSTEM_NAME MATCHES "Windows")
    aux_source_directory(./src/yangcapture/win DIR_SRCS)
    aux_source_directory(./src/yangdecoder/pc DIR_SRCS)
    aux_source_directory(./src/yangrecord DIR_SRCS)
elseif(CMAKE_SYSTEM_NAME MATCHES "Linux")
    # Linux 特定代码
elseif(APPLE)
    aux_source_directory(./src/yangmac DIR_SRCS)
else()  # Android
    aux_source_directory(./src/yangandroid DIR_SRCS)
endif()

add_library(metartc8 ${DIR_SRCS})
```

**特点：**
- 支持模块化编译（通过 NoCapture、NoPlayer 标志）
- 平台自适应代码包含
- 依赖 libyangutil8 和 libmetartccore8

---

## 编译优化与配置

### 编译优化选项

#### 速度优化
```cmake
-O2              # 平衡的优化（推荐）
-O3              # 激进优化（可能增加代码大小）
-march=native    # 针对本地 CPU 优化
```

#### 大小优化
```cmake
-Os              # 优化大小
-flto            # 链接时优化
-fvisibility=hidden  # 隐藏符号（减小库大小）
```

#### 调试支持
```cmake
-g               # 生成调试符号
-DDEBUG          # 启用调试日志
```

### 条件编译

```cmake
# 禁用采集功能（嵌入式环境）
./cmake_build.sh 2 64 0 -DNoCapture=ON

# 禁用播放器功能
./cmake_build.sh 2 64 0 -DNoPlayer=ON

# 同时禁用采集和播放
./cmake_build.sh 2 64 0 -DNoCapture=ON -DNoPlayer=ON
```

---

## 输出结构

### 编译输出目录结构

```
bin/
├── lib_x64/                    # x64 输出
│   ├── libyangtil8.a
│   ├── libmetartccore8.a
│   ├── libyangwhip8.a
│   └── libmetartc8.a
├── lib_android/
│   ├── armeabi-v7a/
│   │   ├── libyangtil8.a
│   │   ├── libmetartccore8.a
│   │   ├── libyangwhip8.a
│   │   └── libmetartc8.a
│   └── arm64-v8a/
│       └── ...
└── app_debug/                  # 演示应用输出
    ├── metapushstream8
    └── metaplayer8
```

### 库文件输出

**库文件类型：**
- `.a` : 静态库（链接时合并）
- `.so` : 共享库（动态链接）
- `.dll` : Windows 动态库
- `.dylib` : macOS 动态库

---

## 编译故障排查

### 常见问题

#### 1. 找不到头文件
```
错误: fatal error: openssl/ssl.h: No such file or directory

解决:
- 检查 thirdparty/user_include/openssl 目录
- 确保 OpenSSL 开发包已安装
- 在 CMakeLists.txt 中验证 include_directories 路径
```

#### 2. 链接错误
```
错误: undefined reference to 'CRYPTO_malloc'

解决:
- 确保 libcrypto.a 在 thirdparty/lib 中
- 检查库依赖顺序
- 验证架构匹配（32位 vs 64位）
```

#### 3. 架构不匹配
```
错误: cannot find -lsrtp
       /usr/lib/libsrtp.a(error object): 
       compiled for x86_64, not compatible with x86

解决:
- 编译所有依赖库的正确架构版本
- 使用 file 命令检查库文件类型: file libsrtp.a
- 确保交叉编译器配置正确
```

#### 4. C++ 标准不兼容
```
错误: 'nullptr' was not declared in this scope

解决:
- 确保使用 -std=gnu++11 或更新标准
- 检查 CMakeLists.txt 中的 CMAKE_CXX_FLAGS
- 更新编译器到较新版本
```

---

## 编译最佳实践

### 1. 清洁编译
```bash
# 删除所有构建产物
find . -name "build" -type d -exec rm -rf {} +
find . -name "CMakeFiles" -type d -exec rm -rf {} +

# 重新开始编译
./cmake_lib_x64.sh
```

### 2. 增量编译（开发时）
```bash
# 只编译特定模块
cd libmetartc8
./cmake_build.sh 0 64 0
```

### 3. 调试编译
```bash
# 编译时保留调试符号
./cmake_build.sh 0 64 0 -DCMAKE_BUILD_TYPE=Debug

# 启用详细输出
./cmake_build.sh 0 64 0 --verbose
```

### 4. 交叉编译
```bash
# 指定 ARM 交叉编译工具链
./cmake_build.sh 2 64 0 \
  -DCMAKE_C_COMPILER=arm-linux-gnueabihf-gcc \
  -DCMAKE_CXX_COMPILER=arm-linux-gnueabihf-g++
```

---

## 性能调优

### 编译时优化

```bash
# 启用 LTO（链接时优化）
./cmake_build.sh 0 64 0 -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON

# 针对特定 CPU 优化
./cmake_build.sh 0 64 0 -DCMAKE_C_FLAGS="-O3 -march=native"

# 删除不必要的符号
./cmake_build.sh 0 64 0 -DCMAKE_C_FLAGS="-O2 -fvisibility=hidden"
```

### 库大小优化

```bash
# 优化输出大小
./cmake_build.sh 0 64 0 -DCMAKE_C_FLAGS="-Os"

# 剥离调试符号
strip bin/lib_x64/libmetartc8.a
```

---

## 持续集成配置示例

```bash
#!/bin/bash
# CI 编译脚本

# 编译所有架构
for arch in x64 android ios; do
    echo "编译 $arch..."
    ./cmake_lib_${arch}.sh || exit 1
done

# 运行测试
cd demo/metapushstream8
cmake . -DCMAKE_BUILD_TYPE=Debug
make
./test_pushstream

# 验证输出
ls -lh bin/lib_*/lib*.a
```

---

*最后更新：2026年4月27日*
