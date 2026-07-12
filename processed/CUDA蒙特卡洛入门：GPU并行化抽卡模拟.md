---
title: CUDA蒙特卡洛入门：GPU并行化抽卡模拟
source: raw_20260712_211119_d316e82d
source_url: https://zhuanlan.zhihu.com/p/32491301173
created: '2026-07-12'
updated: '2026-07-12'
tags:
- CUDA
- GPU并行计算
- 蒙特卡洛模拟
- PCG32
- 性能优化
status: processed
note_type: content
related:
- CUDA 系统学习路线：从入门到 MMA 版 FlashAttention
- GPU架构入门：从SIMD到SIMT的执行模型
---

# CUDA蒙特卡洛入门：GPU并行化抽卡模拟

## 核心结论
> 蒙特卡洛方法天然适合GPU并行暴力模拟，而CUDA通过Grid→Thread Block→Warp→Thread的软件层级与GPU→SM→CUDA Core的硬件层级一一对应，使得移植CPU随机模拟程序只需将函数标为__global__/__device__并借助cudaMalloc/cudaMemcpy管理跨设备内存即可。作者在RTX 4060上实现了约4.5倍加速，但受限于4060缺乏64位整数硬件单元以及GPU缺失分支预测带来的Warp分化损耗，性能仍有较大提升空间。

## 详细内容
## 一、为什么用GPU跑蒙特卡洛

[[蒙特卡洛方法]]通过海量随机采样求解复杂概率问题，计算密集且子任务相互独立，是[[并行计算]]的典型场景。CPU核心少（通常<100）且偏重复杂逻辑，而GPU拥有上千个[[CUDA Core]]专为数值并行设计，因此成为蒙特卡洛的天然加速器。

## 二、GPU硬件模型

GPU采用[[冯诺依曼架构]]，但层级更深：

```
GPU
├── SM 0 (Streaming Multiprocessor)
│     ├── CUDA Core Array（含大量CUDA Core，每个Core有独立Register和PC）
│     ├── Shared Memory & L1 Cache
│     └── Control
├── SM 1
├── ...
└── Global Memory & L2 Cache
```

- **SM（Streaming Multiprocessor）**：GPU的基本调度单位，内部所有CUDA Core共享同一控制器，粗略认为执行同一条指令。
- **CUDA Core**：实际运算单元，仅有寄存器，无独立指令寄存器。
- **内存速度**：Register > Shared Memory > Global Memory（板载显存）。

相比CPU每个Core独享L1/L2 Cache并共用L3 Cache的浅层结构，GPU通过Shared Memory实现SM内线程的快速数据共享。

## 三、CUDA软件模型

CUDA将软件层级映射到硬件层级：

```
GPU Device → Grid → Thread Block → Warp → Thread
```

- **Grid**：一次核函数启动的所有线程，按`<<<GRID_SIZE, BLOCK_SIZE>>>`配置。
- **Thread Block**：由多个Warp组成，运行在同一个SM上，可使用Shared Memory通信。
- **Warp**：32个线程的调度基本单位，同一Warp内线程必须执行同一条指令（SIMT）。

### 函数修饰符

- `__host__`：CPU侧执行（默认）。
- `__global__`：核函数，从CPU调用、在GPU每个线程上执行，必须返回`void`。
- `__device__`：仅GPU调用。
- 注意：`__global__`中禁止调用任何`__host__`函数。

## 四、最简CUDA程序四步

1. 申请GPU/CPU内存（可选）。
2. CPU→GPU数据拷贝（可选）。
3. 启动`__global__`核函数。
4. GPU→CPU结果拷贝（可选）。

关键API：
- `cudaMalloc(void** devPtr, size_t size)`：申请显存。
- `cudaMemcpy(dst, src, count, kind)`：跨设备拷贝，`kind`取`cudaMemcpyHostToDevice` / `cudaMemcpyDeviceToHost`。
- `cudaMemset`：显存初始化。

## 五、CPU到GPU的改造要点

以抽卡模拟为例，原CPU线程函数改写为`__global__ threadSimulate`时需注意：

1. 函数和工具库（如PCG32随机数）全部加`__device__`修饰。
2. 全局变量改用`__device__`声明，常量数组可用`__constant__`直接驻留显存。
3. `math.h`函数不可用，改用`cuda_runtime_api.h`（函数名相同）。
4. 结果数组在显存中分配，CPU端用`cudaMemcpy`回传。

作者实测在RTX 4060上相比CPU线程版本（180秒）降至40秒，约**4.5倍加速**。

## 六、性能瓶颈分析

### 1. 4060缺乏64位整数硬件

[[PCG32]]随机引擎涉及大量64位整数运算，而4060的SM内没有专用64位整数单元，需拆解为多条32位指令，速度显著下降。可考虑改用CUDA内置的[[cuRAND]]库。

### 2. GPU缺失分支预测

GPU以Warp为单位调度，Warp内遇到`if`分支时会**先执行所有真分支线程再执行假分支线程**，造成SIMT分化（warp divergence）。抽卡逻辑中存在大量条件判断，浪费大量执行周期。

## 七、编译指令

```bash
nvcc -o WuwaPullSimulateCUDA WuwaPullSimulate.cu -O4 -gencode arch=compute_80,code=sm_80 --ptxas-options=-v
```

跨架构兼容可用`nvcc -O4 WuwaPullSimulate.cu`简化编译。

## 相关笔记
- [[CUDA 系统学习路线：从入门到 MMA 版 FlashAttention]]
- [[GPU架构入门：从SIMD到SIMT的执行模型]]