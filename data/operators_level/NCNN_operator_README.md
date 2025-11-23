## 数据集概述
该数据集包含 16 个 NCNN 推理框架核心算子及一个通道拼接特化变体的元信息，用于评估 AI Agent 在算子层面进行 RVV 加速迁移与正确性验证的能力。

## 数据集构建目的
1. 覆盖推理常见关键路径：卷积、归一化、激活、归约、序列与全连接等。
2. 量化迁移关注点：为每个算子标注 RVV 向量化、内存访问、数值稳定、布局重排等焦点。
3. 提供统一测试入口：给出典型输入形状，便于自动化生成用例。
4. 支持特化验证：包含 concat 通道维度特化变体，考察场景化优化能力。

## 选择原则
1. 推理高频覆盖：优先选择 Transformer/CNN 常见算子。
2. 可验证向量化：均具备明确的 RVV 加速切入点。
3. 形状代表性：输入张量覆盖 1D/2D/全连接/序列场景。
4. 可分解迁移焦点：每个算子明确 3 个左右迁移关注点，不冗余交叉。

## 类别分布
- 激活函数：swish, sigmoid, relu, tanh, gelu
- 归一化与正则：batchnorm, dropout
- 线性/矩阵运算：innerproduct, lstm
- 卷积相关：convolution1d, convolutiondepthwise
- 归约/分布：softmax
- 特征处理：pooling, eltwise, concat

## 迁移关注点主题汇总
- RVV向量化策略：循环展开、通道并行、批处理、掩码与宽度自适应
- 内存与布局：权重重排、连续块拷贝、cache 行对齐、指针递增访问
- 数值稳定与近似：exp/tanh/erf 近似、多项式/降级路径、归一化稳定性
- 分支与融合：激活融合、分支消除、推理快速跳过
- 序列与时间步：LSTM 时间循环最小化、门计算并行
- 量化与兼容：InnerProduct 量化路径验证、Dropout 推理行为一致性

## 算子列表

| ID  | 名称                     | 主类                         | 功能概述              | 典型输入示例              |
| --- | ---------------------- | -------------------------- | ----------------- | ------------------- |
| 1   | concat                 | ncnn::Concat               | 多输入按指定轴拼接         | [2,16,32,32] x 3    |
| 2   | swish                  | ncnn::Swish                | x * sigmoid(x) 激活 | [1,64,56,56]        |
| 3   | sigmoid                | ncnn::Sigmoid              | 1/(1+exp(-x))     | [1,128,28,28]       |
| 4   | eltwise                | ncnn::Eltwise              | 加/乘/最大逐元素融合       | [1,64,32,32] x 2    |
| 5   | innerproduct           | ncnn::InnerProduct         | 全连接矩阵乘 + 偏置       | [1,1024]→[1,256]    |
| 6   | relu                   | ncnn::ReLU                 | ReLU/带斜率变体        | [1,256,14,14]       |
| 7   | dropout                | ncnn::Dropout              | 训练置零/推理缩放         | [1,128,28,28]       |
| 8   | batchnorm              | ncnn::BatchNorm            | 通道标准化             | [1,64,112,112]      |
| 9   | tanh                   | ncnn::TanH                 | 双曲正切激活            | [1,256,20,20]       |
| 10  | convolution1d          | ncnn::Convolution1D        | 1D 卷积序列处理         | [1,64,512], k=3     |
| 11  | softmax                | ncnn::Softmax              | 指数归一化概率           | [1,1000]            |
| 12  | pooling                | ncnn::Pooling              | 最大/平均池化           | [1,64,56,56]→3x3 s2 |
| 13  | convolutiondepthwise   | ncnn::DepthwiseConvolution | DW 可选点卷积融合        | [1,128,28,28]       |
| 14  | lstm                   | ncnn::LSTM                 | 序列记忆单元            | T=32, hidden=256    |
| 15  | gelu                   | ncnn::GELU                 | 高斯误差线性单元          | [1,768]             |


## 使用建议
1. 按表中输入形状生成基准输入，验证功能与性能。
2. 优先实现迁移焦点列出的向量化与内存优化，再扩展到量化/融合。
3. 序列类（lstm、convolution1d）需关注步长与隐藏维度的可变形测试。
4. 激活近似路径（gelu, swish, tanh, sigmoid）应同时保留精确与近似实现开关。

## 测试示例流程（参考）
- 生成输入张量（随机/均匀/边界值）。
- CPU 标量实现 vs RVV 实现结果比对 (max_abs_diff, mean_diff)。
- 分析性能：记录 cycles 或 ns，统计加速比。
- 对含权重算子（innerproduct, convolution1d, depthwise）测试不同布局重排前后差异。

## 后续扩展建议
- 增补归约类（argmax、layernorm）。
- 增加多头注意力算子用于序列场景扩展。
- 引入量化版本字段（int8 path）便于统一管理。

---