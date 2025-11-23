## 数据集概述

Level1 模型数据集包含 17 个基于 NCNN 的示例模型，用于评估 AI Agent 在模型级场景下迁移 底层算子的能力（以 ncnn 框架的算子为目标）。

## 数据集构建目的

1. 测试模型级迁移能力：在完整示例中验证 Agent 实现并替换底层算子的能力。  
2. 确保算子覆盖：通过真实模型驱动迁移必要的算子集合，检验覆盖完整性。  
3. 评估实际效果：运行完整示例验证迁移后算子的正确性与性能。

## 模型选择原则
- 每个示例代表一个完整任务，Agent 在迁移时只能基于该示例获得算子信息。

### 选择标准

1. 算子互斥：示例之间尽量无算子重叠。    
2. 实战场景：示例来自真实的模型推理（分类、检测、OCR、分割等）。  
3. 服务器友好：优先不依赖摄像头/GUI，便于批量自动化测试。  
4. 默认参数：支持使用默认模型和输入以便自动化运行。

## 数据集组成

模型列表（共 17 个）：

| ID | 模型文件 (cpp)             | 主要算子（代表）                                                                 | 应用场景 |
|---:|:-------------------------|:----------------------------------------------------------------------------------|:--------|
| 1  | mobilenetssd.cpp       | from_pixels_resize, substract_mean_normalize, Convolution, DepthwiseConvolution, SSD 输出 | 实时目标检测 |
| 2  | squeezenet.cpp         | from_pixels_resize, substract_mean_normalize, Convolution, Fire, Softmax          | 图像分类 |
| 7  | nanodet.cpp            | from_pixels_resize, DepthwiseConvolution, Concat, Reshape                        | 轻量目标检测 |
| 8  | simplepose.cpp         | from_pixels_resize, channel, Convolution, Permute, Reshape                       | 姿态估计 |
| 9  | shufflenetv2.cpp       | from_pixels_resize, DepthwiseConvolution, Concat, Permute, Softmax               | 轻量分类 |
|13  | mobilenetv2ssdlite.cpp | from_pixels_resize, DepthwiseConvolution, Concat, SSD 输出                       | 轻量目标检测 |
|14  | mobilenetv3ssdlite.cpp | from_pixels_resize, SEModule (SE), create_extractor                              | 带注意力机制的检测 |
|21  | peelenet_ssd.cpp       | from_pixels_resize, Convolution, Concat, Reshape                                 | SSD 检测 |
|23  | rvm.cpp                | from_pixels_resize, Split, Concat, DepthwiseConvolution                          | 视频抠图（RVM） |
|24  | p2pnet.cpp             | from_pixels_resize, Convolution, Reshape                                         | 密度估计 / 人群计数 |
|25  | ppocr.cpp              | from_pixels_resize, Convolution, Pooling, Reshape                                | 文本检测与识别流水线 |
|26  | ppocr_v4.cpp           | from_pixels_resize, Convolution, Pooling, Reshape                                | 文本检测与识别（改进） |
|27  | piper.cpp              | from_pixels_resize, Convolution, Reshape                                         | 文本到语音（推理模块） |
|28  | yolov8_cls.cpp         | from_pixels_resize, SiLU, Convolution, Softmax                                   | YOLOv8 分类变体 |
|31  | yolov11_cls.cpp        | from_pixels_resize, SiLU, Convolution, Softmax                                   | YOLOv11 分类变体 |
|38  | squeezenetssd.cpp      | from_pixels_resize, Concat, Permute, Softmax                                     | SqueezeNet + SSD 检测 |
|39  | squeezenet_c_api.cpp   | C API 封装：ncnn_mat_from_pixels_resize, ncnn_extractor_create, ncnn_*_forward    | C 语言接口示例（分类） |

## 算子分布（主要类别）

- 输入与预处理：from_pixels_resize, substract_mean_normalize  
- 数据布局与选通：Permute, Reshape, channel, Split, Concat  
- 卷积类：Convolution, DepthwiseConvolution, Fire（模块化卷积）  
- 激活与归一化：ReLU, SiLU, BatchNorm, SEModule（注意力）  
- 池化与下采样：Pooling, Reshape  
- 推理框架接口：create_extractor, input, extract（ncnn::Extractor）  
- 输出层：Softmax

## 测试与运行依赖

- 必要依赖：ncnn 库。  
- C API 示例需同时编译 ncnn 的 C 接口（若存在）。  

## 测试流程（简要）

1. 为迁移后的底层算子实现或替换模块后，重新编译 ncnn/相关库。  
2. 进入示例代码目录，创建独立 build 目录：mkdir build && cd build  
3. 使用 cmake 指定 ncnn的安装路径并生成 Makefile：  
   - cmake -DNCNN_DIR=/your/ncnn/build -Dncnn_DIR=/your/ncnn/install ..  
1. make 编译并运行示例二进制以验证功能与性能。

---

此 README 便于快速了解 Level1 数据集的模型分布、算子覆盖与测试要点。