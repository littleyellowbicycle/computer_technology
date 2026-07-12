---
title: CUDA 系统学习路线：从入门到 MMA 版 FlashAttention
source: raw_20260712_211114_cf13c987
source_url: https://www.zhihu.com/question/263832290/answer/88074576627
created: '2026-07-12'
updated: '2026-07-12'
tags:
- CUDA
- 高性能计算
- PyTorch
- Tensor Cores
- FlashAttention
- 学习路线
status: processed
note_type: content
related:
- CUDA蒙特卡洛入门：GPU并行化抽卡模拟
- 计算机科学自学路线与经典书籍
- 新手使用GitHub与计算机基础学习路线
---

# CUDA 系统学习路线：从入门到 MMA 版 FlashAttention

## 核心结论
> CUDA 学习应当遵循"由易到难、CUDA Core 与 Tensor Core 并重、结合 PyTorch 验证"的原则。DefTruth 维护的 CUDA-Learn-Notes v3.0 仓库提供近 200 个 kernel 实现（覆盖 FP32/TF32/FP16/BF16/FP8/INT8）与 PyTorch binding，其中 HGEMM 性能达到 cuBLAS 的 98%~100%、手搓的 MMA 版 FlashAttention-2 达到官方 95%~99% 性能，是一份兼具面试刷题与工程实战价值的高质量学习资料。

## 详细内容
## 为什么需要一份系统的 CUDA 学习资料

CUDA 学习最大的痛点在于"看了一堆书和教程，依然写不出像样的 kernel"。一个高质量的学习路径应当覆盖三个维度：

1. **覆盖面**：从基础 [[元素级操作]]、[[矩阵转置]]、[[Warp Reduce]]/[[Block Reduce]]，到 [[NMS]]、[[ReLU]]、[[GELU]]、[[Layer Norm]]、[[RMS Norm]]、[[Online Softmax]]、[[点积]]、[[Embedding]]、[[RoPE]] 等常见算子；
2. **数据类型**：覆盖 [[FP32]]、[[TF32]]、[[FP16]]、[[BF16]]、[[FP8]]、[[INT8]] 等当前主流精度；
3. **进阶优化**：[[Tensor Cores]]、[[MMA PTX]]、[[WMMA]]、访存合并、向量化、Bank Conflict 消除、Warp Shuffle、Shared Memory Swizzle、寄存器 Double Buffer 等。

## 推荐资源：CUDA-Learn-Notes v3.0

仓库地址：https://github.com/xlite-dev/CUDA-Learn-Notes（已收获约 2k stars）

全称 *Modern CUDA Learn Notes with PyTorch for Beginners*，其特点可以概括为：

- **近 200 个 kernel**，按主题组织，每个主题的工作流统一为：自定义 CUDA kernel 实现 → [[PyTorch]] Python 绑定 → Python 测试数值与性能
- **难度分级**为 5 档：简单 ⭐️、中等 ⭐️⭐️、困难 ⭐️⭐️⭐️、困难+ ⭐️⭐️⭐️⭐️、困难++ ⭐⭐⭐️⭐️⭐️
  - 简单与中等部分覆盖基础算子与各类精度的基本用法
  - 困难部分深入 [[SGEMV]]、[[SGEMM]]、[[HGEMV]]、[[HGEMM]] 等矩阵乘算子
  - 困难+ 与困难++ 部分集中在 [[FlashAttention]]-2 的 MMA 实现以及衍生工作 [[FFPA]]
- **HGEMM 性能**：最优 MMA 实现在 Tensor Cores 上达到 [[cuBLAS]] 98%~100% 性能，特性包括 Tile Block、Tile MMAs、Pack LDST、SMEM Padding、SMEM Swizzle、Block/Warp Swizzle、Multi-Stages、Registers Double Buffer、NN/TN Layout、Collective Store 等
- **FlashAttention-2 MMA**：在 MMA Acc F32 下达到官方 [[FlashAttention-2]] 95%~98% 性能；MMA Acc F16 版本在 4090 等设备上可达 1.5x 原生性能（但存在溢出风险）
- **FFPA 拓展**：将 FlashAttention 的 SRAM 复杂度降至 O(1)，headdim 可扩展到 1024，在 L20 上对 D=512 场景达到 104 TFLOPS，比 [[SDPA]]（Efficient Attention）快约 1.8x
- **100+ 技术博客**：按主题分类整理，方便按需取用

## 环境与上手建议

- 推荐 Python ≥ 3.10、PyTorch ≥ 2.4.0、CUDA ≥ 12.4（推荐 PyTorch 2.5.1 + CUDA 12.5）
- 直接使用 NVIDIA 官方镜像 `nvcr.io/nvidia/pytorch:24.10-py3`，避免环境配置踩坑
- **强烈不推荐 CUDA 11**：FP8 与最新 MMA 指令在 CUDA 11 下不可用
- 通过 `TORCH_CUDA_ARCH_LIST=Ada`（或其他架构）控制编译目标，避免全架构编译耗时过长
- 每个 kernel 都附带 Python 测试脚本，跑完日志即可直接对比手写 kernel 与 PyTorch 算子的数值与性能

## 学习路径建议（针对原问题：高二自学者）

1. **入门阶段**：阅读简单 ⭐️ 部分，先理解 [[CUDA 编程模型]]（Grid/Block/Warp/Thread）与内存层次，理解基础 kernel 的并行拆分思路
2. **进阶阶段**：练习中等 ⭐️⭐️ 部分的算子实现，掌握访存合并、向量化和基本 reduce 模式
3. **面试阶段**：刷困难 ⭐️⭐️⭐️ 部分的 HGEMM 八股，理解 [[Tensor Cores]] 编程（WMMA 与 MMA PTX 两条路线）
4. **专家阶段**：研读困难+ 与困难++ 部分的 FlashAttention-2 MMA 与 [[FFPA]] 实现，理解 SRAM 与寄存器的精细优化

对于自学者，关键是**跑通 → 改 kernel → 再跑通**的循环，而不是停留在"读代码"层面。

## 相关笔记
- [[CUDA蒙特卡洛入门：GPU并行化抽卡模拟]]
- [[计算机科学自学路线与经典书籍]]
- [[新手使用GitHub与计算机基础学习路线]]