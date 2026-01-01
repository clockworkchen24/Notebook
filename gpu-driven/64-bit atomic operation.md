# 64-bit atomic operation.md

64位原子操作是Nanite类的GPU Driven的一大限制。本文主要介绍原因和替代方案。



## 软光栅

**硬件光栅化的瓶颈：** 光栅化以 2x2 Quad 为单位的特性。当三角形小到像素级（Micro-polygons）时，Quad 的利用率极低（Helper Pixels 占用了大量带宽和计算），导致性能剧烈下降。

**GPU Driven ：** 为了解决上述问题，我们需要在 Compute Shader 中自行实现逻辑，跳过传统的固定管线。



## Atomic64

在软光栅中，我们需要在一个原子操作内同时更新 **Depth** 和 **TriangleID/InstanceID**。

如果没有 Atomic64，我们就无法在单个 Pass 里保证深度测试的原子性，我写过一个demo，会导致严重的画面闪烁。



## 硬件支持现状

**DirectX 12：** **Shader Model 6.6** 

**Vulkan：** 早期通过 `VK_KHR_shader_atomic_int64` 扩展提供支持，并在 **Vulkan 1.2** 中被提升为核心特性。

**Metal：** Apple 在 **Metal 3.1**（随 macOS Ventura / iOS 16 引入）中对 64 位原子操作提供了部分支持，主要集中在 `atomic_min` 和 `atomic_max`。

最主要的问题就是移动端。因此，**WebGPU**没有对Atomic64 进行支持。

所以Nanite没法在移动端（以及SM低的windows）运行。



## 代替方案

如果想在移动端跑GPU Driven，就必须舍弃Atomic64。

可以使用triangle culling来一定程度上代替软光栅，让Micro-polygons在进入光栅化之前就被干掉。（腾讯的移动端GPU Driven用的就是这个方案

这就让整个Driven从数据上分成了两个流派：

Visibility Buffer 和 Triangle Buffer

前者是极致的：低内存，高性能；但是兼容性差。UE为了展现Nanite的极致性能选择了这个方式。

后者能够在任意支持Compute Shader的平台实现GPU Driven