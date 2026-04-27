# libyuv 库获取和编译指南

本文档为 MetaRTC 项目配置 libyuv 库的完整步骤。

## 概述

metapushstream8 应用依赖 libyuv 库进行视频格式转换和处理。本指南提供针对不同平台的详细步骤。

---

## 一、Windows 平台编译指南

### 前置要求
- CMake 3.16 或以上
- Visual Studio 2019/2022 或 GCC/Clang
- Git

### 步骤 1：克隆 libyuv 源码

```bash
# 进入临时目录（或自定义位置）
cd x:/master/rtc/metaRTC/thirdparty

# 克隆 libyuv 库
git clone https://chromium.googlesource.com/libyuv/libyuv.git

cd libyuv
```

### 步骤 2：编译 libyuv

#### 方案 A：使用 CMake（推荐）

```bash
# 创建编译目录
mkdir build_x64
cd build_x64

# 生成 Visual Studio 项目（64 位）
cmake .. -G "Visual Studio 16 2019" -A x64 -DCMAKE_BUILD_TYPE=Release

# 编译
cmake --build . --config Release

# 编译成功后，库文件位置：
# Release\yuv.lib 或 Release\yuv_static.lib
```

#### 方案 B：使用 GCC/MinGW

```bash
mkdir build_x64
cd build_x64

cmake .. -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++
make -j4
```

### 步骤 3：复制库文件到项目

```bash
# 从编译目录复制库文件
copy x:/master/rtc/metaRTC/thirdparty/libyuv/build_x64/Release/yuv.lib x:/master/rtc/metaRTC/thirdparty/lib/win/

# 或如果是静态库
copy x:/master/rtc/metaRTC/thirdparty/libyuv/build_x64/yuv_static.lib x:/master/rtc/metaRTC/thirdparty/lib/win/
```

### 步骤 4：编译 metapushstream8

```bash
cd x:/master/rtc/metaRTC/demo/metapushstream8

# 方案 A：使用 CMake
mkdir build
cd build
cmake .. -G "Visual Studio 16 2019" -A x64
cmake --build . --config Release

# 方案 B：使用 qmake（Qt）
qmake metapushstream8.pro
nmake release  # 或 make release
```

---

## 二、Linux/Mac 平台编译指南

### 前置要求
- CMake 3.16 或以上
- GCC/Clang 编译器
- make 或 ninja
- Git

### 步骤 1：克隆 libyuv 源码

```bash
cd /home/user/metaRTC/thirdparty  # 替换为实际路径
git clone https://chromium.googlesource.com/libyuv/libyuv.git
cd libyuv
```

### 步骤 2：编译 libyuv（Linux x86_64）

```bash
# 创建编译目录
mkdir build_linux_x64
cd build_linux_x64

# 生成构建文件
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON

# 编译
make -j$(nproc)

# 库文件位置：libyuv.a
ls -lh libyuv.a
```

### 步骤 3：复制库文件到项目

```bash
# Linux 库文件
cp /path/to/libyuv/build_linux_x64/libyuv.a \
   /home/user/metaRTC/thirdparty/lib/libyuv.a

# 或者放到版本特定目录
mkdir -p /home/user/metaRTC/bin/lib_release
cp /path/to/libyuv/build_linux_x64/libyuv.a \
   /home/user/metaRTC/bin/lib_release/
```

### 步骤 4：使用项目编译脚本编译 metapushstream8

```bash
cd /home/user/metaRTC

# 使用项目提供的编译脚本
bash cmake_lib_x64.sh

# 编译 metapushstream8 应用
cd demo/metapushstream8
mkdir build
cd build
cmake ..
make -j4

# 输出应用
# 位置：../../bin/app_debug/metapushstream8 （调试版）
# 或：../../bin/app_release/metapushstream8 （发布版）
```

---

## 三、macOS 平台编译指南

### 步骤 1：克隆 libyuv 源码

```bash
cd ~/metaRTC/thirdparty
git clone https://chromium.googlesource.com/libyuv/libyuv.git
cd libyuv
```

### 步骤 2：编译 libyuv

#### Intel Mac

```bash
mkdir build_mac_x64
cd build_mac_x64

cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON

make -j$(sysctl -n hw.ncpu)
```

#### Apple Silicon（M1/M2）

```bash
mkdir build_mac_arm64
cd build_mac_arm64

cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON

make -j$(sysctl -n hw.ncpu)
```

### 步骤 3：复制库文件

```bash
# Intel Mac
cp build_mac_x64/libyuv.a ~/metaRTC/thirdparty/lib/mac/

# Apple Silicon
cp build_mac_arm64/libyuv.a ~/metaRTC/thirdparty/lib/mac/libyuv_arm64.a
```

### 步骤 4：编译项目应用

```bash
cd ~/metaRTC/demo/metapushstream8
mkdir build
cd build
cmake ..
make -j$(sysctl -n hw.ncpu)
```

---

## 四、Android 交叉编译指南

### 前置要求
- Android NDK r21 或以上
- CMake 3.16 或以上

### 步骤 1：克隆 libyuv 源码

```bash
cd ~/metaRTC/thirdparty
git clone https://chromium.googlesource.com/libyuv/libyuv.git
cd libyuv
```

### 步骤 2：编译 libyuv（arm64-v8a）

```bash
export ANDROID_NDK=/path/to/android-ndk-r21

mkdir build_android
cd build_android

cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-21 \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON

make -j$(nproc)

# 库文件位置：libyuv.a
```

### 步骤 3：复制库文件

```bash
# 为不同 ABI 编译和复制
mkdir -p ~/metaRTC/bin/lib_android/arm64-v8a
cp build_android/libyuv.a ~/metaRTC/bin/lib_android/arm64-v8a/

# 如需支持其他 ABI，重复上述步骤：
# - armeabi-v7a
# - x86
# - x86_64
```

### 步骤 4：编译应用

```bash
cd ~/metaRTC
bash cmake_lib_android.sh
```

---

## 五、快速验证

### 验证库文件是否存在

```bash
# Windows
dir x:/master/rtc/metaRTC/thirdparty/lib/win/yuv.lib

# Linux/Mac
ls -lh /path/to/metaRTC/thirdparty/lib/libyuv.a
或
ls -lh /path/to/metaRTC/bin/lib_release/libyuv.a

# Android
ls -lh /path/to/metaRTC/bin/lib_android/arm64-v8a/libyuv.a
```

### 验证头文件

```bash
# 头文件应该已经存在
ls -la /path/to/metaRTC/thirdparty/include/libyuv/

# 应该包含以下文件：
# - basic_types.h
# - convert.h
# - scale.h
# - rotate.h
# 等等
```

### 编译测试

```bash
cd /path/to/metaRTC/demo/metapushstream8

# 清理旧编译
rm -rf build CMakeFiles CMakeCache.txt

# 重新编译
mkdir build
cd build
cmake ..
make

# 检查是否成功
make VERBOSE=1  # 看详细输出
```

---

## 六、常见问题排查

### 问题 1：找不到 yuv 库

```
error: cannot find -lyuv
```

**解决方案：**
```bash
# 检查库文件是否存在
find /path/to/metaRTC -name "*yuv*" -type f

# 确保库文件在正确的目录
# Windows: thirdparty/lib/win/yuv.lib
# Linux:   thirdparty/lib/libyuv.a 或 bin/lib_debug/libyuv.a
# Mac:     thirdparty/lib/mac/libyuv.a
```

### 问题 2：头文件找不到

```
fatal error: libyuv.h: No such file or directory
```

**解决方案：**
```bash
# 检查头文件目录
ls -la /path/to/metaRTC/thirdparty/include/libyuv/

# 如果目录为空，重新克隆 libyuv
cd /path/to/metaRTC/thirdparty
rm -rf libyuv
git clone https://chromium.googlesource.com/libyuv/libyuv.git
```

### 问题 3：架构不匹配

```
undefined reference to `libyuv::...'
```

**解决方案：**
```bash
# 确保使用的库文件与编译目标架构一致
file /path/to/lib/libyuv.a

# 应该输出类似：
# /path/to/lib/libyuv.a: current ar archive, 64-bit ...
```

### 问题 4：CMake 找不到库

```
CMake Error: Target "metapushstream8" links to target "yuv" but the target was not found
```

**解决方案：**
```bash
# 检查 CMakeLists.txt 中的库路径设置
cat /path/to/metaRTC/demo/metapushstream8/CMakeLists.txt | grep -A 5 "target_link_libraries"

# 确保 link_directories 设置正确
cat /path/to/metaRTC/demo/metapushstream8/CMakeLists.txt | grep -A 2 "target_link_directories"
```

---

## 七、一键编译脚本（可选）

创建 `build_libyuv.sh` 脚本，简化编译流程：

### Linux/Mac 版本

```bash
#!/bin/bash
# build_libyuv.sh

set -e

METARTC_ROOT="$(cd "$(dirname "$0")" && pwd)"
LIBYUV_SOURCE="${METARTC_ROOT}/thirdparty/libyuv"
LIBYUV_BUILD="${LIBYUV_SOURCE}/build_linux_x64"

echo "=== 开始编译 libyuv ==="

# 1. 克隆源码
if [ ! -d "$LIBYUV_SOURCE" ]; then
    echo "正在克隆 libyuv 源码..."
    git clone https://chromium.googlesource.com/libyuv/libyuv.git "${LIBYUV_SOURCE}"
fi

# 2. 编译
echo "正在编译 libyuv..."
mkdir -p "${LIBYUV_BUILD}"
cd "${LIBYUV_BUILD}"

cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_POSITION_INDEPENDENT_CODE=ON

make -j$(nproc)

# 3. 复制库文件
echo "正在复制库文件..."
mkdir -p "${METARTC_ROOT}/bin/lib_release"
cp libyuv.a "${METARTC_ROOT}/bin/lib_release/"

echo "=== libyuv 编译完成 ==="
echo "库文件位置: ${METARTC_ROOT}/bin/lib_release/libyuv.a"

# 4. 编译 metapushstream8
echo ""
echo "=== 开始编译 metapushstream8 ==="
cd "${METARTC_ROOT}/demo/metapushstream8"
rm -rf build
mkdir build
cd build
cmake ..
make -j$(nproc)

echo "=== 编译完成 ==="
echo "应用位置: ${METARTC_ROOT}/bin/app_debug/metapushstream8"
```

### 使用脚本

```bash
# 授予执行权限
chmod +x build_libyuv.sh

# 运行脚本
./build_libyuv.sh
```

---

## 八、项目集成验证

编译完成后，验证项目是否正确集成 libyuv：

```bash
# 检查应用是否链接了 yuv 库
ldd /path/to/metaRTC/bin/app_debug/metapushstream8 | grep yuv

# 或使用 nm 检查符号
nm -C /path/to/metaRTC/bin/app_debug/metapushstream8 | grep -i yuv
```

---

**最后更新：2026-04-27**
