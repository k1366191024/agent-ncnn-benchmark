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

### 🆚 vs. OpenCV 等其他库
* **(陈鹏飞部分):** 介绍与 OpenCV 迁移任务的差异...
* **(况首旭部分):** 介绍开题时提到的差异点...

---

## 📊 测试集设计 (Benchmark Design)

我们基于 NCNN 源码特性，对算子进行了系统性的筛选与分级。

### 难度分级 (Difficulty Levels)
详见 [operator_difficulty_level.csv](./operator_difficulty_level.csv)

* **L1 (Basic)**: 包含 `relu`, `concat` 等基础算子。逻辑简单，易于自动化迁移。
* **L2 (Advanced)**: 包含 `convolution`, `softmax`, `lstm` 等复杂算子。通常涉及 Im2Col 变换、滑窗优化或复杂的向量规约（Reduction）。

<details>
<summary>🔻 点击展开查看 27 个核心算子明细</summary>

| 等级 | 算子列表 |
| :--- | :--- |
| **L1** | `concat`, `dropout`, `relu`, `slice`, `ShuffleChannel` |
| **L2** | `swish`, `eltwise`, `sigmoid`, `innerproduct`, `tanh`, `batchnorm`, `Convolution1d`, `softmax`, `pooling`, `convolutiondepthwise`, `lstm`, `gelu`, `scale`, `reshape`, `binaryop`, `interp`, `lrn`, `convolution`, `Deconvolution`, `GroupNorm`, `Flatten`, `DeconvolutionDepthWise` |

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