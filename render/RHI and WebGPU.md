# RHI & WebGPU 两种跨平台模式
为什么WebGPU是一种新的图形API？它和RHI的区别是什么？
从Native来看，WebGPU其实做的是RHI的事情，最终执行的指令依然是DX/VK/GL。而且也没有任何硬件驱动。
## 共性
从实现跨平台的这个设计目标来看，二者很类似，虽然实现有些区别：
- Cross-platform
- 基于Command/Encoder
    - 二者都放弃了OpenGL 全局状态机的设计。
    - bgfx: 使用 Encoder 接口记录commands，然后提交。
    - WebGPU: 使用 CommandEncoder 录制 CommandBuffer，然后提交到 Queue。
- PSO管理
    - bgfx 使用基于位掩码的动态状态。
    - WebGPU 则是完全使用现代图形 API 的 Pipeline Layout 和 BindGroup 概念,是一个创建成本很重的、固化的对象。
## 核心区别
我在看使用bgfx的渲染器时，发现找不到诸如对DrawCall进行排序从而优化性能的代码。
后来发现bgfx的源码其实做了这个事情了,我认为这就是两者的核心区别
### bgfx
它有一个核心概念叫 View。可以在任意时刻向任意 View 提交 Draw Call。bgfx 会在帧结束时，根据你设定的 View 顺序和 View 内部的 Sort Key 对所有指令进行重排序,比如 UI 层永远在 view 255，不透明物体在 view 0。
### WebGPU
而WebGPU更像 Vulkan/Metal 的简化版。你必须显式地开启一个 RenderPass，录制命令，结束 Pass，它不负责帮你自动排序 Draw Call，你录入的顺序就是 GPU 执行的顺序。所以渲染器本身需要对drawcall做Sort、Culling等事情

## Submit
### bgfx::submit()
是逻辑层的动作。它只是向一个临时的Bucket里扔了数据，并不立即触碰图形驱动。真正的渲染提交在 bgfx::frame()之后，把处理完的数据丢到渲染线程。
bgfx::submit和真正的提交之间，其实做了很多drity work:
#### 调用 submit() 时
1. State Snapshotting
    拷贝一份数据副本到内存中
2. 生成 64-bit Sort Key
    算出一个key（但是不排序）
3. 写入内部 Command Buffer
    bgfx内部维护的指令列表
#### frame() 之前
1. Linear Allocator
    分配一块大内存，和渲染器用的方式一样
2. 引用计数与生命周期追踪
#### 调用 frame() 时
1. Double Buffering Swap
2. Kick Render Thread
3. The Aftermath
    这发生在 frame() 返回之后，这一步才进行排序和State Deduplication（为了Pipelining）
### WebGPU
是驱动层的动作。它接收开发者提交的CommandBuffer，进行Resource Tracking和barrier插入，然后立即推送到对应API的GPU queue。
## 观点
两者的设计哲学看似类似其实很不同
- bgfx把一个渲染器需要的功能全都封装进去了，使用起来很简单，但是开发者控制能力差,适合用在游戏引擎
- WebGPU只是一个API，适合用在高性能/想要高度定制的渲染器,它比 Vulkan 易用，比 OpenGL 现代
