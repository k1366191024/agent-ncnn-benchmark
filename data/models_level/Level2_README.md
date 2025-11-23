## 数据集概述

Level2 模型数据集包含 24 个样例程序（ncnn / 推理示例），主要为目标检测、实例分割、人脸检测与姿态估计等视觉任务的示例代码。每个示例以一个独立的 C++ 示例文件（examples/*.cpp）描述模型输入、调用链路及所依赖的算子与算子接口。

## 数据集构建目的

1. 测试 agent 在完整模型场景下迁移 HAL / 底层算子的能力，而非孤立算子测试。  
2. 保证算子覆盖度：通过真实模型模型驱动迁移所需的算子集合验证完整性。  
3. 评估实际效果：运行完整示例验证迁移后的算子在真实推理流程中的正确性与性能。

## 模型选择原则

### 核心原则：多样性与算子覆盖

- 优先选择覆盖多类模型架构（YOLO 系列、R-CNN、SCRFD、YOLACT、PeleeNet 等）以最大化算子集合覆盖。  
- 在保证代表性的同时尽量减少不必要的算子冗余，便于按模型拆分迁移任务。  
- 偏好服务端友好示例（可使用静态图片/预置输入，尽量避免依赖摄像头或 GUI）。

### 选择标准

1. 算子覆盖：示例集合应覆盖卷积、归一化、激活、重排/变形、池化、分支/拼接等常见推理算子。  
2. 实战性：示例源自项目 demo 与 examples，代表真实推理流程。  
3. 自动化友好：支持默认输入与批处理，便于自动验证。

## 数据集组成

该 JSON 列表中每个条目包含：id / cpp_filename / source_path / 简要描述 / model_path / 主要算子数组 / 算子接口签名数组。

模型一览（节选，按 id 升序）：

| ID  | 示例文件                   | 主要算子（示例）                                 | 场景说明 |
| --- | -------------------------- | ------------------------------------------------ | -------- |
| 3   | yolov5.cpp                 | from_pixels_resize, substract_mean_normalize, Convolution, BatchNorm, SiLU, Concat, SPP | YOLOv5 实时目标检测，多尺度预测 |
| 4   | yolov8.cpp                 | from_pixels_resize, substract_mean_normalize, Convolution, SiLU, DepthwiseConvolution, AreaAttention | YOLOv8 动态分辨率检测 |
| 5   | scrfd.cpp                  | Convolution, BatchNorm, ReLU, DepthwiseConvolution, Permute, Sigmoid | SCRFD 人脸检测，多尺度特征融合 |
| 6   | retinaface.cpp             | Convolution, BatchNorm, ReLU, Pooling, Concat    | RetinaFace 人脸检测与 5 点关键点 |
| 10  | fasterrcnn.cpp             | Convolution, ROIPooling, Pooling, Softmax        | Faster R-CNN 两阶段检测 |
| 11  | yolov3.cpp                 | LeakyReLU, Convolution, SPP, Concat              | YOLOv3 三尺度检测 |
| 12  | yolov4.cpp                 | Mish, SPP, Split, Concat                          | YOLOv4 (CSPDarknet + SPP) |
| 15  | yolox.cpp                  | SiLU, Split, Concat, Softmax                      | YOLOX 解耦头与无锚设计 |
| 16  | yolov2.cpp                 | LeakyReLU, Convolution, Softmax                   | YOLOv2 经典实现 |
| 17  | yolov7.cpp                 | SiLU, DepthwiseConvolution, Split, Concat        | YOLOv7 E-ELAN 聚合 |
| 18  | yolov7_pnnx.cpp            | split, concat                                     | PNNX 导出的动态输入模型 |
| 19  | rfcn.cpp                   | roi_pooling                                       | R-FCN 区域卷积检测 |
| 20  | yolact.cpp                 | Split, Concat, ReLU                               | YOLACT 实时实例分割（原型掩码） |
| 22  | peelenet_ssd_seg.cpp       | Split, Concat, Softmax                            | PeleeNet SSD + 语义分割分支 |
| 29  | yolov8_pose.cpp            | split                                             | YOLOv8 姿态估计变体 |
| 30  | yolov8_seg.cpp             | Split, Concat, SiLU, DepthwiseConvolution        | YOLOv8 实例分割变体 |
| 32  | yolov11_obb.cpp            | DepthwiseConvolution, Split, Concat              | YOLOv11 旋转框检测 |
| 33  | yolov11_pose.cpp           | DepthwiseConvolution, Split, Concat              | YOLOv11 姿态估计变体 |
| 34  | yolov11_seg.cpp            | DepthwiseConvolution, Split, Concat              | YOLOv11 实例分割变体 |
| 35  | yolov11.cpp                | DepthwiseConvolution, Split, Concat              | YOLOv11 常规模型 |
| 36  | yolov8_world.cpp           | SiLU, DepthwiseConvolution, Conc at, Softmax     | YOLOv8-World 开放词汇检测 |
| 37  | yolov8_obb.cpp             | Split, Concat, Reshape, Softmax                   | YOLOv8 旋转框检测 |
| 40  | scrfd_crowdhuman.cpp       | Permute, Reshape, Sigmoid                         | SCRFD 针对拥挤场景的优化 |
| 41  | yolov8_world_v2.cpp        | SiLU, DepthwiseConvolution, Softmax               | YOLOv8-World v2 |

（注：表中“主要算子”为示例性抽取，完整算子列表见 JSON 字段 operators；算子接口签名见 operator_interfaces。）

### 算子覆盖概览

覆盖的常见算子类别包括但不限于：

- 数据预处理：from_pixels_resize, substract_mean_normalize  
- 推理流程控制：create_extractor, input, extract  
- 卷积/通用：Convolution, DepthwiseConvolution, BottleneckCSP  
- 归一化与激活：BatchNorm, ReLU, SiLU, Mish, LeakyReLU, Sigmoid, Softmax  
- 空间/形变：Pooling, SPP, ROIPooling, Permute, Reshape  
- 结构性：Split, Concat, Layer

## 使用与测试流程（建议）

1. 先完成 HAL / 算子迁移并编译对应的加速模块（ncnn 或自研后端）。  
2. 在项目根目录或 examples 对应目录下创建构建目录，例如：  
   mkdir -p build_yolov5 && cd build_yolov5  
3. 运行 cmake 指定依赖库路径（示例）：  
   cmake -DNCNN_DIR=/path/to/ncnn/install -D NCNN_DIR=/path/to/ncnn/build ..  
4. make 编译生成可执行文件，运行示例并对比输出或精度基准（参考 operator_interfaces 提供的调用签名）。  
5. 若示例有可视化（imshow），在无 GUI 的服务器上请改为输出结果文件或禁用可视化。

依赖（可选，用于可视化）：  
- libgtk2.0-dev + pkg-config（用于 imshow 等）
