# AIGlasses-local

## 🎯 项目背景：为什么要在手机上跑大模型

- **隐私**，没有之一。用户图像上传云端，合规风险高，即使用本地服务器，不可避免还是需要经过运营商。
- **成本**，云端的推理，即使上了 **code plan** 也压不住视频采集级别的token消耗；而在外旅游时，本地服务器也需要消耗大额的流量。
- **延时**，在一些需要高速追踪的场合，“外挂”大脑的延时还是太高了。
- **算力**，AI眼镜作为第一人称视角，有很强的沉浸感，但续航和算力实在太紧张了。随身设备中，算力最强的只有手机，而且人手一个。
- **离线**，地铁、地下停车场、电梯照样工作


## 🏗️ 技术架构总览

```
┌─────────────────────────────────────────────────────┐
│                   小米眼镜 AI 解说助手                │
├─────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌─────────────┐    ┌─────────┐ │
│  │ USB 摄像头   │───▶│ 图像采集     │───▶│ Qwen3.5 │ │
│  │ (小米眼镜)   │    │ (OpenCV)    │    │ 推理    │ │
│  └─────────────┘    └─────────────┘    └────┬────┘ │
│                                              │      │
│  ┌─────────────┐    ┌─────────────┐         │      │
│  │ TTS 语音    │◀───│ 文本生成     │◀────────┘      │
│  │ (Android)   │    │ (llama.cpp) │                │
│  └─────────────┘    └─────────────┘                 │
└─────────────────────────────────────────────────────┘
                        ▲
                        │
              ┌─────────┴─────────┐
              │   OnePlus 12      │
              │  Snapdragon 8 Gen3│
              │   24GB LPDDR5X    │
              │   Adreno 750 GPU  │
              └───────────────────┘
```

| 指标 | 速度 |
|------|------|
| Prompt 处理 | **33.3 t/s** |
| 生成速度 | **16.9 t/s** |
| 内存占用 | **~1.2GB** |


### 安装 Termux

别从 Google Play 下，去 **F-Droid** 或 **GitHub Releases**：

```bash
# 推荐源
https://github.com/termux/termux-app/releases
```

## 基础环境

```bash
# 更新源
pkg update && pkg upgrade -y

# 安装依赖
pkg install -y python cmake clang git wget vulkan-tools
pip install huggingface_hub hf_transfer
pkg install -y git
```

### 编译llama.cpp
```bash
git clone https://github.com/ggml-org/llama.cpp
cd ~/llama.cpp
rm -rf build
cmake -B build -DGGML_LLAMAFILE=OFF -DLLAMA_BUILD_TESTS=OFF -DLLAMA_BUILD_SERVER=ON
cmake --build build --config Release -j$(nproc)

```

### 模型量化
```bash
cd ~/aiLearn/llama.cpp/
python3 -m venv llama_cpp
source llama_cpp/bin/activate
pip install --upgrade pip

huggingface-cli download Qwen/Qwen3.5-0.8B \
  --local-dir ./models

pip3 install sentencepiece protobuf torch transformers
cmake -B build -DLLAMA_BUILD_TESTS=OFF
cmake --build build --config Release -j$(nproc)

python3 convert_hf_to_gguf_update.py ~/models/Qwen3.5-0.8B/   --outfile ~/models/qwen3.5-0.8b-f16.gguf   --outtype f16

./build/bin/llama-quantize \
  ~/models/qwen3.5-0.8b-f16.gguf \
  ~/models/qwen3.5-0.8b-q4_k_m.gguf \
  Q4_K_M
```

### 视觉模型
```bash
aria2c -x 16 -s 16 -k 1M https://huggingface.co/unsloth/Qwen3.5-35B-A3B-GGUF/resolve/main/mmproj-F16.gguf
```


## 运行推理
```bash
cd ~/llama.cpp/build/bin
export LD_LIBRARY_PATH=.

termux-setup-storage

./llama-cli -m ~/storage/shared/Download/qwen3.5-0.8b-f16.gguf \
  -c 2048 \
  -p "你好，请介绍一下自己"
```

> [End thinking]                                                                          你好！我是Qwen3.5，一款基于深度学习模型开发 的超大规模语言模型。我具备强大的语言理解、逻辑推理、代码生成、数学计算、文本创作等多种能力，能够处理复杂任务。例如，我可以协助你完成编程、数据分析、内容创作、逻辑论证等，无论是解决实际问题还是激发创意，我都提供支持。                                                如果你有任何具体问题，或者想了解我的功能、应用场景，欢迎随时提问！ 😊                                                               [ Prompt: 33.3 t/s | Generation: 16.9 t/s ] 


## 交叉编译
### OpenCL
```bash
# 安装 OpenCL 头文件和库
cd ~/android
mkdir -p opencl

# 下载 OpenCL Headers
git clone https://github.com/KhronosGroup/OpenCL-Headers.git
cp -r OpenCL-Headers/CL $ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include/

# 下载 OpenCL ICD Loader
git clone https://github.com/KhronosGroup/OpenCL-ICD-Loader.git
cd OpenCL-ICD-Loader
mkdir build && cd build
cmake .. -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
  -DOPENCL_ICD_LOADER_HEADERS_DIR=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-28 \
  -DANDROID_STL=c++_shared
ninja
cp libOpenCL.so $ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/

# 编译 llama.cpp 带 OpenCL 支持
cd ~/aiLearn/llama.cpp
rm -rf build-android-opencl

cmake -B build-android-opencl \
  -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-28 \
  -DCMAKE_C_FLAGS="-march=armv8.7a" \
  -DCMAKE_CXX_FLAGS="-march=armv8.7a" \
  -DGGML_OPENCL=ON \
  -DGGML_OPENMP=OFF \
  -DGGML_LLAMAFILE=OFF \
  -DLLAMA_BUILD_TESTS=OFF \
  -DLLAMA_BUILD_SERVER=ON \
  -DCMAKE_BUILD_TYPE=Release

cmake --build build-android-opencl --config Release -j$(nproc)

```




## 运行推理服务器
```bash
./llama-server -m ~/storage/shared/Download/qwen3.5-0.8b-f16.gguf \
  -c 32768 \
  -b 1024 \
  -ub 512 \
  -t 8 \
  --mmproj ~/storage/shared/Download/qwen3.5-0.8b-mmproj-F16.gguf \
  --host 127.0.0.1 \
  --port 8080
```

| 参数   | 默认值 | 推荐值      | 说明                |
| ---- | --- | -------- | ----------------- |
| -c   | 512 | 4096     | 上下文长度             |
| -b   | 512 | 512-1024 | 批处理大小，越大吞吐越高      |
| -ub  | 512 | 512      | 解码批处理大小           |
| -t   | 自动  | 6-8      | 线程数（8核大核）         |
| -ngl | 0   | 0        | GPU 层数（CPU 推理设 0） |


### 调试
```bash
# 健康检查
curl http://localhost:8080/health

# 查看模型列表
curl http://localhost:8080/v1/models

# 简单对话测试
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "q4_k_m.gguf",
    "messages": [{"role": "user", "content": "你好"}],
    "max_tokens": 100
  }'

# 流式输出测试
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "q4_k_m.gguf",
    "messages": [{"role": "user", "content": "写一首关于春天的诗"}],
    "stream": true,
    "max_tokens": 200
  }'
```

