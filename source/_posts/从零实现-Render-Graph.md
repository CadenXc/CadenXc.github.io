---
title: 从零实现 Render Graph
tags:
  - Vulkan
  - Render Graph
description:
  - 介绍Render Graph 的学习实现心得。
date: 2026-02-28 22:32:34
---
 现代图形 API 将内存管理、多线程管理等大部分工作交由开发者负责，这让开发者能有极高的自由度去榨干硬件性能，但也呈指数级增加了 API 使用的复杂度和困难度。例如，使用原生 Vulkan 画一个三角形就需要写上千行的代码。
 软件工程的本质就是管理复杂度。现代引擎必须实现一个中间层架构，在利用图形 API 性能优势的同时，有效降低其复杂度，并让整体渲染管线更加可扩展、易于调试。这个全新的渲染调度大脑就是 **RenderGraph**。
 RenderGraph（也叫 Frame Graph）最早由 EA 寒霜引擎（Frostbite Engine）团队在[GDC 2017 大会](https://www.gdcvault.com/play/1024612/FrameGraph-Extensible-Rendering-Architecture-in)上首次提出。它主要由 **RenderPass** 和 **Resource** 构成：
 - **RenderPass** 是一份元数据，定义了一个完整的渲染流程需要用到的所有资源。
 - **Resource** 是指 RenderPass 使用的 RenderTarget、Texture、Shader 等物理资产。
 
 每个 RenderPass 都会显式指定对应的 Input 和 Output 资源。通过这种依赖关系，RenderPass 和 Resource 就形成了一个严密的 **有向无环图（DAG）** 结构。
![RenderGraphDAG](/images/RenderGraphDAG.png)

 因为 RenderGraph 在执行实际绘制前拥有**完整一帧的全部信息**，通过分析各个 RenderPass 之间的依赖关系，引擎可以极其方便地对底层资源做全局优化。DAG 也很容易做到图表化，极大方便了现代复杂渲染管线的 Debug。最重要的是，它可以自动化推导 Pass 的执行顺序、自动插入硬件同步屏障，并利用**显存复用（Memory Aliasing）** 显著减少整体的 VRAM 占用。

RenderGraph 的核心理念就是**延迟执行（Deferred Execution）**，将整个渲染流程严格拆分为三个生命周期阶段：

### 核心理念：延迟执行 (Deferred Execution)
 RenderGraph 的核心理念是“延迟执行”，它将整个渲染流程严格拆分为三个界限分明的生命周期阶段：
#### Phase 1: Setup（声明意图 —— 只声明，不干活）
在此阶段，上层业务层并不直接操作物理显存。我们使用轻量级的**虚拟资源句柄 (RGResourceHandle)** 来描述所有纹理和 Buffer。
 ```c++
 // 案例：声明一个光追阴影 Pass
 graph.AddPass<ShadowData>("RT_Shadow",
     // Setup 闭包：声明依赖
     [&](PassBuilder& builder, ShadowData& data) {
         data.output = builder.Write("ShadowSignal", ...); // 声明写入
         builder.Read("Depth"); // 声明读取 G-Buffer 深度
     },
     // Execute 闭包：被冻结，直到 Execute 阶段才被唤醒
     [=](const ShadowData& data, ExecutionContext& ctx) {
         // 真正的 GPU 指令录制被封装在这里
         // ctx.TraceRays(...);
     }
 );
 ```

#### Phase 2: Compile（编译优化 —— 算法大脑）
 
这是调度器最显功力的地方，引擎在此阶段执行两项“硬核”优化：

- **拓扑排序与并行分层**：利用 Kahn 算法解析资源依赖。具有相同依赖深度的 Pass 会被划分到同一执行层级，这为 Vulkan 的 Async Compute（异步计算）并行执行提供了完美的调度方案。
- **显存别名重用 (Memory Aliasing)**：引擎精准扫描每个资源的生命周期（首次写入至末次读取）。生命周期互不重叠的临时资源（例如延迟渲染的中间图与后期的 Bloom 缓冲区）将共享同一块物理显存偏移量。这有效根治了 4K 渲染下的显存吞噬问题。

#### Phase 3: Execute（执行指令 —— 自动化同步）
在此阶段，调度器线性遍历拓扑层级，激活自动化同步引擎。
- **状态机推导**：引擎实时比对资源的前后状态（Layout 和 Access Mask）。
 - **自动插入屏障**：如果发现状态不匹配，系统会自动调用 Vulkan 的 `vkCmdPipelineBarrier2` 指令。开发者再也不用去关心复杂的写后读（RAW）等冲突，引擎从架构层面确保了每一帧执行的正确。