---
title: Vulkan Debug心得
date: 2026-02-24 20:18:21
tags:
 - Vulkan
 - Debug
description:
 - 写Vulkan混合渲染器时学到的Debug技巧。
---


最近在写毕业设计，一个Vulkan混合渲染器。Vulkan极其繁琐的API和合格的显式控制给本就困难的图形学Debug蒙上了又一道阴影。
下面总结了的几条我用上的技巧，有更好的方法欢迎在评论里指出，我会满怀感激得尝试。

### 验证层与结构化日志
首先是启用[Vulkan 验证层](https://geek-docs.com/vulkan/vulkan-tutorial/understand-validation-layers.html)。
同时配合使用开源日志库 [spdlog](https://github.com/gabime/spdlog) 实现一个自定义的日志类。
使用不同颜色的日志标识。这样当前验证层出错时能一眼看出。

### 给资源起名字
当验证层报错时，默认情况下它只会告诉你：“`VkImage 0x12ab34cd` 的布局不正确”。在一个拥有几十张贴图的混合管线中，这种报错无异于大海捞针。

可以使用下面的函数SetDebugName() 来给资源起名。
```c++
    void VulkanContext::SetDebugName(uint64_t handle, VkObjectType type, const char* name)
    {
        if (handle == 0 || name == nullptr)
        {
            return;
        }
        auto func = (PFN_vkSetDebugUtilsObjectNameEXT)vkGetDeviceProcAddr(GetDevice(), "vkSetDebugUtilsObjectNameEXT");
        if (func)
        {
            VkDebugUtilsObjectNameInfoEXT nameInfo{ VK_STRUCTURE_TYPE_DEBUG_UTILS_OBJECT_NAME_INFO_EXT };
            nameInfo.objectType = type;
            nameInfo.objectHandle = handle;
            nameInfo.pObjectName = name;
            func(GetDevice(), &nameInfo);
        }
    }
```



下面这段代码使用宏来强制检查错误。
```C++
#define VK_CHECK(result) \

    do { \

        VkResult res = (result); \

        if (res != VK_SUCCESS) { \

            throw std::runtime_error("Vulkan error: " + std::to_string(res)); \

        } \

    } while(0)
```


### 可视化
图形编程debug重要的还是可视化。
使用开源库Imgui，可以很好的实现可视化Debug，可以通过 UI 面板实时切换显示 Albedo、Normal、Motion Vector 或 Depth，可以调整相机，光源位置来。

因为本项目中我采用了RenderGraph管理渲染管线，所以实现了一键导出 [Mermaid](https://mermaid.live/) 流程图代码的功能。这样能可视化查看当前RenderGraph的拓扑逻辑。
[这个大佬的FrameGraph可视化](https://www.zhihu.com/zvideo/1484005811182084096)做得很好。

RenderDoc在我这次开发中没用到，不过[官方文档](https://www.howtovulkan.com/#graphics-debugger)有提到。