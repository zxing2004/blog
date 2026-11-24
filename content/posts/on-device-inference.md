---
title: "端侧智能落地：ONNX Runtime 与模型量化的工程路径"
date: 2026-06-30
draft: false
description: "端侧推理受限于内存、算力与功耗。本文讲清量化原理，并用 ONNX Runtime 与 WebGPU 给出可运行的落地代码路径。"
tags: ["端侧推理", "ONNX", "量化", "WebGPU"]
categories: ["工具与效率"]
---

## 热点引入：智能正在从云端走向设备

2026 年 6 月的 WWDC 上，Apple 用 Gemini 重建 Siri，端侧智能再次成为行业焦点。与此呼应，前端也通过 WebGPU 获得了在浏览器里跑本地推理的能力。两条线索指向同一个趋势：**推理正在从云端下沉到设备端**。

驱动力很现实——隐私（数据不出端）、延迟（无需网络往返）、成本（省掉云端算力）、离线可用性。但端侧不是把云端模型缩小一点那么简单，它面对的是一套完全不同的约束：内存以几百 MB 计、算力有限、功耗敏感、冷启动要快。想让模型在手机或浏览器里跑得动、跑得快，量化几乎是必经之路。本文就沿着"约束 → 量化原理 → ONNX Runtime 落地 → 浏览器 WebGPU"这条路径讲清楚。

## 技术原理：端侧约束与量化

### 端侧的四道坎

- **内存**：移动端可用内存有限，一个 FP32 权重就占 4 字节，动辄数十亿参数的小模型也要数 GB，根本放不下。
- **算力**：端侧 GPU/NPU 的峰值算力远低于数据中心显卡，浮点运算代价高。
- **功耗与发热**：持续高负载推理会迅速耗电、触发降频，体验反而变差。
- **冷启动**：每次打开都要加载模型，体积直接影响首屏时间。

### 量化：用低精度换体积与速度

量化的本质是把权重和激活从高精度浮点映射到低精度表示：

- **FP32 → FP16**：体积减半，多数硬件原生支持，精度损失极小，通常是最稳妥的第一步。
- **FP16 → INT8**：体积再减半，整数运算更快更省电，但精度损失更明显，需要校准或训练来补偿。

要区分两类量化：

- **权重量化**：只量化模型权重，激活仍用浮点。实现简单、精度损失小，收益主要来自体积和内存带宽。
- **激活量化**：连中间激活也量化，才能充分发挥整数算力的速度优势，但精度更敏感，实现更复杂。

还有两条技术路线：

- **PTQ（训练后量化）**：训练完成后直接量化，用少量校准数据确定量化范围，成本低、见效快，是首选。
- **QAT（量化感知训练）**：训练时就把量化误差纳入考量，精度更高，但需要重新训练，成本大，用于 PTQ 精度不达标的场景。

一个直观的量化公式（对称量化）：`q = round(x / scale)`，反量化为 `x ≈ q × scale`，其中 scale 由校准数据的数值范围确定。

## 实操：ONNX Runtime 加载与量化

### Python 端 PTQ 量化

下面用 `onnxruntime.quantization` 对一个 FP32 模型做动态量化到 INT8：

```python
from onnxruntime.quantization import (
    quantize_dynamic, QuantType
)

FP32_MODEL = "model_fp32.onnx"
INT8_MODEL = "model_int8.onnx"

# 动态量化：仅量化权重，根据运行时激活范围动态确定 scale
quantize_dynamic(
    model_input=FP32_MODEL,
    model_output=INT8_MODEL,
    weight_type=QuantType.QInt8,
    per_channel=True,        # 逐通道量化，精度更好
    reduce_range=False,      # 某些旧 CPU 需设为 True
)
print("量化完成:", INT8_MODEL)
```

对于精度要求更高的场景，用静态量化并提供校准数据集：

```python
from onnxruntime.quantization import (
    quantize_static, QuantType, QuantFormat, CalibrationDataReader
)

class MyCalibrator(CalibrationDataReader):
    def __init__(self, samples):
        self.samples = iter(samples)   # samples: list of dict[name -> ndarray]
    def get_next(self):
        return next(self.samples, None)

calib = MyCalibrator(calibration_samples)
quantize_static(
    model_input=FP32_MODEL,
    model_output="model_static_int8.onnx",
    calibration_data_reader=calib,
    quant_format=QuantFormat.QDQ,        # 端侧多为 QDQ 格式，兼容性最好
    activation_type=QuantType.QInt8,     # 注意：不是 quant_type
    weight_type=QuantType.QInt8,
    per_channel=True,
)
```

`quant_type` 这个参数名只在 `quantize_dynamic` 里存在，`quantize_static` 把激活和权重分开给，写成 `quant_type=` 会直接抛 `TypeError: unexpected keyword argument`。

### 加载并推理

```python
import onnxruntime as ort
import numpy as np

sess = ort.InferenceSession(
    INT8_MODEL,
    providers=["CPUExecutionProvider"],   # 端侧按可用硬件切换
)
inp = sess.get_inputs()[0].name
out = sess.get_outputs()[0].name
result = sess.run([out], {inp: input_data.astype(np.float32)})
print(result[0].shape)
```

### 浏览器端：ONNX Runtime Web + WebGPU

前端通过 WebGPU 后端获得 GPU 加速推理能力：

```javascript
import * as ort from "onnxruntime-web";

// 关键：优先 WebGPU，失败时回退 WASM
ort.env.wasm.numThreads = 4;

async function createSession() {
  const options = {
    executionProviders: ["webgpu", "wasm"],  // 按顺序尝试
    graphOptimizationLevel: "all",
  };
  try {
    return await ort.InferenceSession.create("/model_int8.onnx", options);
  } catch (e) {
    console.warn("WebGPU 不可用，回退 WASM:", e);
    return await ort.InferenceSession.create("/model_int8.onnx", {
      executionProviders: ["wasm"],
    });
  }
}

const session = await createSession();
const input = new ort.Tensor("float32", float32Array, [1, 3, 224, 224]);
const output = await session.run({ [session.inputNames[0]]: input });
console.log(output[session.outputNames[0]].data);
```

WebGPU 路径下权重常以 FP16 存储以契合 GPU 的内存带宽优势；WASM 作为通用回退，覆盖面广但性能较弱。`executionProviders` 数组提供有序回退链，是端侧兼容性的关键设计。

## 工程实践建议与踩坑

- **算子兼容性是最大的坑**。量化或转换后，某些算子可能不被目标 provider 支持。落地前用工具逐算子核对支持列表，对不支持的算子做拆分、替换或自定义实现。
- **一定要有回退策略**：WebGPU → WASM → 云端。硬件千差万别，不能假设用户设备都支持某后端。
- **量化后必须做精度回归**。INT8 在小模型上可能精度掉得明显，建立一组基准样本对比量化前后的输出差异，设定可接受阈值。
- **冷启动要实测**。模型体积、解压、初始化时间共同决定首字延迟，必要时做模型分片与懒加载。
- **模型选型：小模型 + 专用任务**。端侧不是跑通用大模型的场子。选一个小而专的模型，把任务收窄（如意图识别、关键词抽取、图像分类），往往比"缩小版通用模型"效果更好、体验更稳。

## 小结

端侧智能的工程路径已经比较清晰：先用 ONNX 统一模型格式，再用量化（PTQ 优先、必要时 QAT）把模型压到设备能承受的体积，然后用 ONNX Runtime 在端侧按硬件选择执行后端，浏览器里则依赖 WebGPU 并获得 WASM 回退。全程要盯住算子兼容性、精度回归与冷启动。记住核心思路——**用专用小模型解决明确问题，而不是把云端模型硬塞进设备**。约束不是障碍，而是促使我们做对架构设计的动力。
