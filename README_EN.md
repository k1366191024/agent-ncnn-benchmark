# 🚀 Agent-Driven NCNN Cross-Architecture Migration Benchmark

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](./LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![RISC-V](https://img.shields.io/badge/Platform-RISC--V-red)](https://riscv.org/)
[![NCNN](https://img.shields.io/badge/Framework-NCNN-yellow)](https://github.com/Tencent/ncnn)

> 🇨🇳 **中文说明 (Chinese Version)**: [README.md](./README.md)

---

## 📖 Table of Contents

- [Introduction](#-introduction)
- [Background (Why this Benchmark?)](#-background-why-this-benchmark)
- [Key Challenges](#-key-challenges)
- [Comparison](#-comparison)
- [Dataset & Hierarchy](#-dataset--hierarchy)
- [Test Cases](#-test-cases)
- [SoTA Results](#-sota-results)
- [Getting Started](#-getting-started)
- [Citation](#-citation)

---

## 📝 Introduction

To foster the prosperity of the RISC-V basic software ecosystem and encourage the academic and industrial exploration of AI Agents for cross-architecture code migration, we have constructed the industry's first **Agent-Driven NCNN Cross-Architecture Migration Benchmark**, based on the **NCNN inference framework** known for its extreme edge-side performance optimization.

While several project-level code benchmarks exist, they are ill-suited for code migration tasks across different hardware architectures (specifically targeting SIMD instruction sets). The goal of this project is to provide a standardized testing platform to evaluate the ability of Large Language Model (LLM) Agents to automatically migrate vectorized implementations from mature architectures (x86/ARM/MIPS/LoongArch) to the RISC-V Vector (RVV) architecture, with a focus on both code correctness and performance optimization.

---

## 💡 Background (Why this Benchmark?)

### Status Quo & Pain Points
Most existing deep learning inference frameworks do not natively support RISC-V Vector backends. While a few frameworks have manually supported some operators with RVV backends, they often suffer from **performance regression** compared to scalar implementations, failing to fully utilize the vector extension modules of RISC-V processors for acceleration.

Manually implementing RISC-V vectorization for these frameworks one by one incurs massive human and time costs. Therefore, **there is an urgent need to automate vectorization support for these deep learning inference frameworks and effectively improve model inference speeds.**

### The Value of Automated Migration
Automated code migration is crucial for system maintenance. Since systems often undergo multiple migrations throughout their lifecycle, automation avoids repetitive manual labor. Supporting RISC-V vectorization for deep learning frameworks is essentially a process of cross-architecture migration from existing mature vector implementations (x86, ARM, etc.) to RISC-V.

### The Opportunity for LLM Agents
As LLM parameter scales expand, their performance in the coding domain has improved significantly. In recent years, LLM Agents have been widely applied in software engineering tasks such as code generation, completion, refactoring, repair, test case generation, and code migration.

Utilizing LLM Agents for automatic operator performance optimization not only saves significant manual migration costs but also significantly improves end-to-end model inference speed.

---

## 🧩 Key Challenges

The challenges faced by this project go beyond simple instruction set translation; they involve crossing the dual chasm of **hardware architecture paradigms** and **software engineering complexity**.

### 1. Architectural Paradigm Shift
* **Fixed vs. Scalable**: x86 AVX/SSE relies on fixed-length registers, and the code is full of hardcoded strides (e.g., `i += 8`). RISC-V RVV adopts a hardware-agnostic Vector Length (VLEN). Migration requires refactoring "fixed-length loops" into `vsetvli`-based **Strip-mining loops**.
* **Instruction Semantic Gap**: Many complex x86 instructions (e.g., `_mm256_shuffle_ps`) have no 1:1 mapping in RISC-V. They require a combination of multiple basic instructions like `vrgather`, `vmerge`, and `vslide` to achieve equivalent functionality, severely testing the agent's logical reconstruction ability.

### 2. Complex Register & Memory Management
* **Dynamic Trade-off of LMUL**: RVV's unique **LMUL (Logic Vector Length Multiplier)** mechanism requires the Agent to dynamically infer the optimal register grouping strategy based on the operator's computational density to balance instruction throughput against register spill risks.
* **NCNN-Specific Data Packing**: NCNN enforces an `elempack` data layout. The Agent must possess **"Layout-Aware"** capabilities to correctly handle Load/Store and boundary alignment under non-standard layouts; otherwise, segmentation faults are inevitable.

### 3. Project-Level "Hallucination"
This is the biggest obstacle moving from demos to real engineering.
* **Implicit Dependency Chains**:
    * Real operator implementations are not isolated; they often depend on macros, struct declarations, and helper functions scattered across `headers/` and `utils/`.
    * **Challenge**: If the Agent focuses only on the current source file, it will **"hallucinate"** non-existent APIs or wrong function signatures due to missing type definitions. Conversely, introducing too many irrelevant files causes distraction due to Context Window noise.
* **The "Fog" of Preprocessors (Macro Obfuscation)**:
    * NCNN heavily uses C preprocessors (Macros) to generate template code (e.g., `DEFINE_LAYER_CREATOR`).
    * **Challenge**: This meta-programming technique masks the real control flow. It is difficult for Agents to precisely expand macros like a compiler, leading to misjudgments in understanding code logic and resulting in migration code that references incorrect symbols or logic branches.

> **Summary**: This Benchmark requires the Agent to possess **Full-Stack Capabilities**: mastering RVV assembly and micro-architecture features at the bottom, and parsing complex C++ project dependencies at the top to accurately manage context and suppress hallucinations.

---

## ⚖️ Comparison

> *This section deeply analyzes the fundamental differences between this Benchmark, high-performance vision libraries (OpenCV), and existing code generation datasets (SWE-bench, HumanEval).*

### 1. vs. OpenCV

Although NCNN and OpenCV are both C++ libraries in the computer vision domain, there is a dimensional difference in their **Optimization Philosophy** and **Migration Difficulty**.

#### 🚩 1.1 Optimization Philosophy
* **OpenCV (Generality)**
    * **Goal**: Cover a wide range of vision scenarios, maintain API stability.
    * **Features**: Standardized data formats (HWC/CHW), easy interoperability; operators are relatively independent; vectorization is concentrated only in hotspots.
* **NCNN (Extremism)**
    * **Goal**: Squeeze **every clock cycle** out of edge-side inference, pursuing extreme performance.
    * **Features**: Aggressive memory layout (`elempack`) sacrifices generality for continuous access; highly optimized assembly Intrinsics constitute a very high proportion.

#### 🧩 1.2 Operator Semantic Complexity Gap

| Dimension | NCNN Deep Learning Operators | OpenCV Traditional Operators |
| :--- | :--- | :--- |
| **Compute Density** | **Very High** (e.g., Winograd Conv, thousands of lines per op) | **Medium/Low** (mostly simple pixel ops) |
| **Data Dependency** | **Complex** (Multi-level loop nesting, Tiling strategies) | **Simple** (Mostly pixel-wise/neighborhood ops) |
| **Precision** | Needs to handle `int8`/`fp16` quantization details | Usually `float32`/`uint8` suffices |

> **Migration Challenge**: NCNN requires the Agent to deeply understand the mathematical essence of algorithms (e.g., Winograd transform) to correctly map them to RVV vectorization strategies; whereas OpenCV's challenge lies mainly in the sheer volume of operators.

#### 📦 1.3 Unique Data Layout Pitfalls

* **NCNN's `elempack` Mechanism**:
    ```cpp
    // input shape: [batch, channels, height, width]
    // When elempack=4, actual storage becomes: [batch, channels/4, height, width, 4]
    ```
    The Agent must identify the **implicit assumption** of packing granularity (4/8/16) in the code. Load/Store instructions must strictly match the `vl` stride, otherwise **misaligned access** will occur.
* **OpenCV's Standard Layout**:
    Uses standard `[rows * cols * channels]` continuous storage, accessing patterns align with conventional thinking.

#### 🕸️ 1.4 The "Deep Trap" of Engineering Dependencies
* **NCNN (High Vertical Depth)**: `Operator` → `Layer Base` → `Allocator` → `Option` → `CPU Feature Detection`. The dependency chain is narrow but deep; Agents easily lose context while tracing layer by layer.
* **OpenCV (Large Horizontal Breadth)**: `Core` ← `HAL` ← `ImgProc` ← `Parallel_for`. Low coupling between modules, but involves a huge total number of operators, facing "Context Explosion."

#### 🧠 1.5 Agent Capability Matrix

| Capability | NCNN Migration | OpenCV Migration |
| :--- | :---: | :---: |
| **Algorithm Understanding** | ⭐⭐⭐⭐⭐ (Deep Learning knowledge) | ⭐⭐⭐ (Traditional CV) |
| **Architecture Mapping** | ⭐⭐⭐⭐⭐ (Deep RVV utilization) | ⭐⭐⭐⭐ (Basic SIMD) |
| **Context Management** | ⭐⭐⭐⭐⭐ (Deep dependency chains) | ⭐⭐⭐⭐⭐ (Broad module interaction) |
| **Macro Parsing** | ⭐⭐⭐⭐⭐ (Heavy reliance) | ⭐⭐⭐ (Relatively restrained) |

> **Summary**: OpenCV migration is **"Breadth-First Search"**, while NCNN migration is **"Depth-First Exploration"**. This Benchmark uses NCNN to target and test the **Deep Thinking** capabilities of Agents in complex engineering migration scenarios.

---

### 2. vs. Related Benchmarks

This Benchmark fills the gap in existing code evaluation datasets at the intersection of **Cross-Architecture Migration** and **Performance-Critical Optimization**.

#### 📊 Comparison Table

| Benchmark | Granularity | Language | Correctness | Perf | Cross-Arch | Task Focus |
| :--- | :--- | :--- | :---: | :---: | :---: | :--- |
| **ClassEval** | Class-level | Python | ✅ | ❌ | ❌ | Class-level code gen |
| **SWE-bench** | Repo-level | Python | ✅ | ❌ | ❌ | Real GitHub Issue solving |
| **JavaBench** | Repo-level | Java | ✅ | ❌ | ❌ | Pro Java dev skills |
| **KernelBench** | Repo-level | CUDA | ✅ | ✅ | ❌ | HPC GPU Kernel gen |
| **ParEval** | Func-level | CUDA/C++ | ✅ | ✅ | ❌ | Parallel program gen |
| **Swe-Perf** | Repo-level | Python | ✅ | ✅ | ❌ | Repo-level bottleneck fix |
| **CISC-RISC** | Func-level | Assembly | ✅ | ❌ | ✅ | Assembly translation |
| **Agent-NCNN (Ours)** | **Repo-level** | **C++ / Intrinsic** | **✅** | **✅** | **✅** | **Cross-Arch Op Migration & Opt** |

#### 🚀 Key Differentiators

1.  **More "Hardcore" than General Code Gen**:
    Unlike benchmarks like SWE-bench that focus on logical correctness, Agent-NCNN introduces **"Performance Constraints,"** requiring the generated RVV code to outperform scalar implementations.

2.  **More "Complex" than HPC Libraries**:
    Unlike KernelBench's standalone Kernel generation, Agent-NCNN operates at the **Repo-level**. The migration process cannot be detached from project contexts like `ncnn/layer.h`, forcing the Agent to handle **cross-file dependencies** and **macro fog**.

3.  **More "Engineering-Oriented" than Simple Translation**:
    Unlike CISC-RISC's instruction translation, Agent-NCNN involves migration from **x86 Intrinsic (C++)** to **RVV Intrinsic (C++)**, involving algorithmic refactoring from **fixed-length vectors** to **variable-length vectors (VLEN)**.

---

## 📊 Dataset & Hierarchy

This Benchmark adopts a hierarchical methodology, as described in [Level Design Introduction](design/intro_NCNNbenchmark_level.md), constructing an evaluation system across three dimensions: **Operator**, **Model**, and **Library**.

### 1. Test Levels

#### 🟢 Level 1: Operator Level
The cornerstone of the Benchmark, focusing on the migration quality of single operators from x86/ARM to RISC-V RVV.
* **Unit Testing (Correctness)**:
    * Strict unit testing of migrated operators.
    * **Standard**: Must pass human-written Test Cases; output results must match the scalar implementation bit-exact or within allowable error margins.
* **Performance Benchmarking**:
    * Comparative evaluation of migrated RVV operator performance.
    * **Baseline**: RISC-V Scalar Implementation.
    * **Reference**: Human-Optimized RVV Operator Performance.

#### 🟡 Level 2: Model Level
Focuses on end-to-end performance when multiple operators work together.
* **Step 1: Operator Coverage**: Unit tests for all operator types involved in the target model.
* **Step 2: Inference Accuracy**:
    * Run the complete model inference flow.
    * **Metric**: Compare inference results with RISC-V scalar implementation, calculating Top-1/Top-5 accuracy loss to ensure no logic errors cause accuracy collapse.
* **Step 3: E2E Performance**:
    * Test the total latency for the entire model to complete one inference pass.
    * **Evaluation**: Compare the Speedup against scalar implementations and human implementations.

#### 🔴 Level 3: Library Level
* *🚧 Coming Soon* - Aims to evaluate the compilation, build, dependency management, and system-level compatibility of the entire NCNN library on RISC-V architecture.

---

### 2. Difficulty Standards

To scientifically evaluate the boundaries of Agent capabilities, we designed fine-grained difficulty levels.

#### 🧩 Operator Difficulty
Graded based on dual dimensions of **Software Logic** and **Hardware Features**.
* **Basis**: Lines of code, control flow complexity (Software), and instruction throughput/vectorization difficulty (Hardware). See [Operator Difficulty Basis](design/intro_ops_dim.md).
* **Data**: See [Operator Difficulty Matrix](design/operator_difficulty_matrix.csv).

---

## 🧪 Test Cases

Based on the above design, we provide a rich set of test cases covering evaluations from basic operators to complex model scenarios.

### 1. Operator Cases
The dataset contains **16 Core NCNN Inference Framework Operators**, aiming to cover critical inference paths and verify RVV vectorization acceleration. See [Operator Level Cases](data/operators_level/NCNN_operator_README.md). Main categories:

* **Activation & Norm**: `ReLU`, `Swish`, `Sigmoid`, `TanH`, `GELU` (including approx implementation), `BatchNorm`, `Dropout`.
* **Convolution & Sequence**: `Convolution1D`, `DepthwiseConvolution`, `LSTM`, `InnerProduct`.
* **Structure & Data**: `Pooling`, `Softmax`, `Concat` (multi-input splicing), `Eltwise`.

### 2. Model Cases
A total of **41 models**, divided into two levels for end-to-end migration assessment. See [Level 1 Models](data/models_level/Level1_README.md) and [Level 2 Models](data/models_level/Level2_README.md).

#### Level 1: Basic Model Migration (17 Models)
Tests Agent capability in low-level operator migration within real-world scenarios without operator overlap.
* **Detection**: `MobileNetSSD`, `NanoDet`, `MobileNetV2/V3 SSDLite`, `SqueezeNetSSD`, `PeleeNet SSD`.
* **Classification**: `SqueezeNet`, `ShuffleNetV2`, `YOLOv8/v11-Cls`.
* **OCR & Specialized**: `PP-OCRv3/v4`, `SimplePose`, `RVM`, `P2PNet`, `Piper`.

#### Level 2: Complex Application Scenarios (24 Models)
Tests Agent capability in handling diverse operator sets and complex data flows in complete model scenarios.
* **YOLO Series**: `YOLOv2` through `YOLOv11`, including `YOLOX`, `YOLOv8-World`, `OBB`, `Pose`, `Seg`.
* **Advanced Detection**: `Faster R-CNN`, `YOLACT`, `R-FCN`.
* **Face Analysis**: `RetinaFace`, `SCRFD`.

---

## 📊 SoTA Results

To analyze Agent performance on different operator types, we provide itemized evaluation results for core operators. See [Operator Evaluation Data](design/sota_llm_result.csv).

<details>
<summary>🔻 Click to expand detailed operator pass rates</summary>

| Operator | Joy(DS-V3) | Joy(GPT-4o) | OH(Claude 4.5) | OH(Gemini 3) | Trae(DS-V3) |
|:---|:---:|:---:|:---:|:---:|:---:|
| `concat` | 0% | 0% | **100%** | **100%** | - |
| `relu` | **100%** | - | - | - | - |
| `dropout` | **100%** | - | - | - | - |
| `eltwise` | 0% | 0% | - | - | - |
| `batchnorm` | 20% | - | - | - | - |
| `pooling` | - | - | **100%** | **100%** | 0% |
| `conv_dw` | - | - | - | - | 0% |
| `swish` | 20% | **40%** | - | - | - |
| `sigmoid` | 40% | **60%** | - | - | - |
| `innerproduct`| 0% | 0% | - | - | - |
| `tanh` | 0% | - | - | - | - |
| `conv1d` | 0% | - | - | - | - |
| `softmax` | - | - | 0% | 0% | 12% |
| `lstm` | - | - | 0% | 0% | 0% |
| `gelu` | - | - | - | - | **100%** |
| `convolution` | - | - | 0% | 0% | - |

*(Note: `-` indicates the model was not tested on this operator)*

</details>

---

## 🛠️ Getting Started

```bash
# 1. Clone the repository
git clone [https://github.com/YourOrg/Agent-NCNN-Benchmark.git](https://github.com/YourOrg/Agent-NCNN-Benchmark.git)

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run the benchmark
python run_benchmark.py --target riscv --model gpt-4