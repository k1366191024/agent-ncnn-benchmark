# 🚀 Agent-Driven NCNN Cross-Architecture Migration Benchmark
# 智能体驱动的 NCNN 库跨架构代码迁移 Benchmark

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](./LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![RISC-V](https://img.shields.io/badge/Platform-RISC--V-red)](https://riscv.org/)
[![NCNN](https://img.shields.io/badge/Framework-NCNN-yellow)](https://github.com/Tencent/ncnn)

---

## 📖 目录 (Table of Contents)

- [项目介绍 (Introduction)](#-项目介绍-introduction)
- [为什么需要这个 Benchmark？(Background)](#-为什么需要这个-benchmark-background)
- [迁移难点 (Key Challenges)](#-迁移难点-key-challenges)
- [与其他 Benchmark 的差异 (Comparison)](#-与其他-benchmark-的差异-comparison)
- [测试集设计与难度分级 (Dataset & Hierarchy)](#-测试集设计与难度分级-dataset--hierarchy)
- [测试题目 (Test Cases)](#-测试题目-test-cases)
- [基准测试结果 (SoTA Results)](#-基准测试结果-sota-results)
- [使用方法 (Getting Started)](#-使用方法-getting-started)
- [引用 (Citation)](#-引用-citation)

---

## 📝 项目介绍 (Introduction)

为了促进 RISC-V 基础软件生态繁荣，同时推动学术界和产业界利用智能体（AI Agents）进行跨架构代码迁移的探索，我们基于端侧性能极致优化的 **NCNN 推理框架**，构建了业界首个**“智能体驱动的 NCNN 库跨架构迁移”Benchmark**。

虽然目前已经有一些项目级代码 Benchmark 发布，但是，这些 Benchmark 并不适用于跨不同硬件架构（特别是针对 SIMD 指令集）的代码迁移任务。本项目的目标是提供一个标准化的测试平台，评估大模型智能体将现有成熟架构（x86/ARM/MIPS/LoongArch）的向量化实现自动迁移到 RISC-V Vector (RVV) 架构的能力，并关注代码的正确性与性能优化。

---

## 💡 为什么需要这个 Benchmark？(Background)

### 现状与痛点
现有深度学习推理框架大多数并不原生支持 RISC-V 向量扩展后端。虽然仅有少数框架手动支持了部分算子的 RVV 后端，但往往存在相比标量实现**性能负优化**的算子，导致无法充分利用 RISC-V 处理器的向量扩展模块来加速模型推理。

对这些深度学习框架逐个手动支持 RISC-V 向量化需要耗费大量的人力和时间成本。因此，**亟需对这些深度学习推理框架实现自动化支持向量化，并提升模型推理速度。**

### 自动化迁移的价值
自动化代码迁移在维护系统方面至关重要，可以避免重复的手动工作，因为系统在其生命周期中通常会经历多次迁移。深度学习推理框架对 RISC-V 的向量化支持，本质上是从现有成熟的 x86、ARM 等架构的向量化支持，跨架构迁移到 RISC-V 的过程。

### 大模型智能体的机遇
随着大模型参数规模的不断扩张，其在代码领域的性能取得了显著提升。近年来，大模型智能体在代码生成、补全、重构、修复、测试用例生成以及最近的代码迁移等软件工程任务中得到了广泛应用。

因此，利用大模型智能体进行自动算子性能优化，不仅能节省手动迁移的大量成本，还能显著提升端到端的模型推理速度。

---

## 🧩 迁移难点 (Key Challenges)

> *本章节详细阐述从 x86/ARM Intrinsics 迁移至 RISC-V RVV 的核心技术难点。*

* **语义鸿沟 (Semantic Gap):** x86 (SSE/AVX) 与 RVV (可变长向量) 在编程模型上的本质差异。
* **寄存器管理:** ...
* **内存访问模式:** ...
* **(此处请补充更多具体的迁移难点描述)**

---

## ⚖️ 与其他 Benchmark 的差异 (Comparison)

> *本章节对比本 Benchmark 与现有的代码生成/迁移数据集（如 HumanEval, MBPP）以及其他高性能库（如 OpenCV）的差异。*

### 🆚 vs. 通用代码 Benchmark
* 大多数现有 Benchmark 关注 Python/Java 等通用逻辑，缺乏对 **底层硬件指令 (Intrinsics)** 和 **并行计算** 的考量。

### 🆚 OpenCV vs. 其他库
* **(陈鹏飞部分):** 介绍与 OpenCV 迁移任务的差异...
### 🆚 NCNN vs.  其他库

与 HumanEval、MBPP 等关注通用逻辑生成的 Benchmark 不同，NCNN 作为一个端侧极致性能的推理框架，其代码迁移面临着**系统级软件工程**的独特挑战：

* **💥 硬件紧耦合的编程范式 (Hardware-Coupled Programming)**
    * **现状**：NCNN 的核心算子并非标准的 C++ 实现，而是大量采用了**平台特定的内联函数 (Intrinsics)** 和**手写汇编**，以手动管理寄存器分配和指令流水线。
    * **挑战**：智能体无法仅通过“翻译逻辑”完成迁移，必须理解 x86/ARM 指令集与 RISC-V Vector (RVV) 在**向量寄存器长度 (VLEN)**、**指令延迟**及**分组策略 (LMUL)** 上的语义鸿沟，进行微架构层面的代码重构。

* **📦 向量化友好的内存布局 (SIMD-Friendly Data Layout)**
    * **现状**：为了最大化 SIMD 指令的吞吐量，NCNN 引入了独特的 **Packing (数据打包)** 策略（如 `elempack=4/8/16`）。数据在内存中并非总是标准的 NCHW 排列，而是根据硬件向量宽度进行了重排。
    * **挑战**：代码迁移过程中，必须严格维护这种数据布局约束。智能体需要具备**布局感知 (Layout-Aware)** 能力，正确生成对应的 load/store 指令及数据重排逻辑，否则会导致严重的内存访问错误或性能下降。

* **🚀 极致的指令级优化 (Instruction-Level Optimization)**
    * **现状**：源端代码（x86/ARM）通常包含了针对特定 CPU 流水线优化的技巧，如激进的**循环展开 (Loop Unrolling)** 和**指令交错 (Instruction Interleaving)** 以隐藏延迟。
    * **挑战**：直接迁移这些优化结构到 RISC-V 可能会导致指令缓存溢出或寄存器压力过大（RISC-V 处理器特性不同）。迁移任务要求智能体能够**识别并剥离**旧架构的过时优化，并针对 RVV 特性生成新的优化模式（如利用 RVV 的 mask 机制消除尾部循环）。

---
## 📊 测试集设计 (Benchmark Design)

为了全面评估智能体在代码迁移中的能力，我们设计了**“软件-硬件”双维度的难度分级体系**。

### 📐 难度分级体系 (Difficulty Matrix)

我们基于算子的**代码逻辑复杂度 (Software)** 和 **硬件指令特性 (Hardware)** 对 27 个核心算子进行了交叉评估。详细数据请查阅 [operator_difficulty_level.csv](operator_difficulty_level.csv)。

#### 1. 软件复杂度 (Software Complexity)
* 🟢 **L1 (Basic)**: 逻辑简单，无复杂依赖（如 `Relu`, `Concat`）。
* 🟡 **L2 (Advanced)**: 涉及复杂计算图、非连续内存或滑动窗口（如 `Conv`, `LSTM`）。

#### 2. 硬件复杂度 (Hardware Complexity)
* 🟢 **H1 (Throughput)**: 访存模式规则，主要受限于计算吞吐量。
* 🔴 **H2 (Latency/Vector)**: 涉及复杂向量指令（如归约、查表）、非对齐访问或对流水线延迟敏感。

---

### 🎯 算子分布明细 (Operator Distribution)

<details open>
<summary>🔻 点击折叠/展开详细列表</summary>

| 算子名称 (Operator) | 软件难度 (SW) | 硬件难度 (HW) | 特性描述 |
| :--- | :---: | :---: | :--- |
| `concat` | L1 | H1 | 简单的内存拼接，带宽敏感 |
| `dropout` | L1 | H1 | 随机掩码生成，元素级操作 |
| `relu` | L1 | H1 | 简单的激活函数 |
| `slice` | L1 | H1 | 张量切片操作 |
| `ShuffleChannel` | L1 | **H2** | 涉及跨通道数据重排 (Shuffle)，硬件开销大 |
| `eltwise` | L2 | H1 | 元素级广播操作 |
| `batchnorm` | L2 | H1 | 批归一化，涉及均值方差计算 |
| `pooling` | L2 | H1 | 池化操作，规则的局部窗口 |
| `convolutiondepthwise`| L2 | H1 | 深度可分离卷积，访存密集 |
| `swish` | L2 | **H2** | 复合激活函数 (x * sigmoid) |
| `sigmoid` | L2 | **H2** | 超越函数，通常需要泰勒展开或查表 |
| `innerproduct` | L2 | **H2** | 全连接层，大矩阵乘法 |
| `tanh` | L2 | **H2** | 双曲正切，高计算强度 |
| `Convolution1d` | L2 | **H2** | 一维卷积 |
| `softmax` | L2 | **H2** | 指数运算 + 全局归约 (Reduction) |
| `lstm` | L2 | **H2** | 循环神经网络，存在时间步依赖 |
| `gelu` | L2 | **H2** | 高斯误差线性单元 |
| `scale` | L2 | **H2** | 缩放操作 |
| `reshape` | L2 | **H2** | 维度变换，可能涉及内存拷贝 |
| `binaryop` | L2 | **H2** | 二元操作 |
| `interp` | L2 | **H2** | 插值算法 |
| `lrn` | L2 | **H2** | 局部响应归一化 |
| `convolution` | L2 | **H2** | 标准卷积，Im2Col + GEMM |
| `Deconvolution` | L2 | **H2** | 转置卷积 |
| `GroupNorm` | L2 | **H2** | 分组归一化 |
| `Flatten` | L2 | **H2** | 展平操作 |
| `DeconvolutionDepthWise`| L2 | **H2** | 深度转置卷积 |

</details>


### 三大类测试设计
| 类别 ID | 类别名称 | 描述 | 典型算子 |
| :--- | :--- | :--- | :--- |
| **Type-I** | 1-to-1 Mapping | 直接的指令映射，逻辑简单 | Add, Sub |
| **Type-II** | Logic Adaptation | 需要调整循环结构或内存布局 | Pooling, Padding |
| **Type-III** | Algorithm Redesign | 需要针对 RVV 特性重写算法 | GEMM, Conv |

### 难度分级标准
* **Level 1 (Easy):** 简单的元素级操作...
* **Level 2 (Medium):** 涉及归约 (Reduction) 或 跨车道 (Cross-lane) 操作...
* **Level 3 (Hard):** 复杂的滑动窗口或不规则内存访问...

(详细设计请参考: [三大类测试设计文档](./intro_NCNNbenchmark_level.md) & [难度分级设计文档](./intro_ops_dim.md))

---

## 🧪 测试题目 (Test Cases)

本 Benchmark 包含模型级和算子级两个维度的测试。

### 1. 算子级测试 (Operator Level)
* **输入:** C++ (x86 AVX/SSE 实现)
* **目标:** C++ (RISC-V RVV 实现)
* **Metric:** 正确性 (Pass@1), 性能加速比 (Speedup)

### 2. 模型级测试 (Model Level)
* **覆盖模型:** ResNet, MobileNet, YOLO, etc.
* **评估指标:** 端到端推理延迟 (Latency), 精度损失 (Accuracy Drop)

---

## 🏆 基准测试结果 (SoTA Results)

我们评估了当前最先进的大模型智能体（如 GPT-4o, Claude 3.5, DeepSeek-Coder 等）在本 Benchmark 上的表现。

| Model / Agent | Strategy | Easy (Pass@1) | Medium (Pass@1) | Hard (Pass@1) | Average Speedup |
| :--- | :--- | :--- | :--- | :--- | :--- |
| GPT-4o | Zero-shot | - | - | - | - |
| Claude 3.5 Sonnet | CoT | - | - | - | - |
| **Ours (Agent)** | **Multi-Agent** | **-** | **-** | **-** | **-** |

---

## 🛠️ 使用方法 (Getting Started)

```bash
# 1. Clone the repository
git clone [https://github.com/YourOrg/Agent-NCNN-Benchmark.git](https://github.com/YourOrg/Agent-NCNN-Benchmark.git)

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run the benchmark
python run_benchmark.py --target riscv --model gpt-4