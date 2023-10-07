---
title: Implement virtual shadow map in CryEngine
date: 2023-08-01 00:06:36
tags:
---

# Abstract

Real-time shadow is an essential field of game development. Without real-time shadow, movable objects will appear floating on the ground and out of tune with the surrounding environment, which decreases game reality. The most used real-time shadow technique is shadow mapping. This technique causes two artifacts: shadow acne and aliasing. The reason behind these artifacts is that many pixels on the screen correspond to one texel on the shadow map. 

Ideally, we can solve these problems by allocating an infinite shadow map texture. However, we can't do this, since video memory is limited. For real-time shadow rendering, most games use CSM (cascade shadow map) to achieve a balance between shadow map quality and game performance. The key idea of CSM is to render a high-quality shadow map for the scene closer to the camera and decrease the shadow quality for further objects.

Virtual shadow map is a possible solution to solving these problems. The main points of Virtual-SM are:1. Only render shadow casters whose shadow can be seen by the camera. 2. Assume the video memory and shadow map texture is large enough (like virtual memory). 3. Manage the scene's objects and dispatch draw commands on the GPU.
# Related techniques used in Virtual-SM

Before discussing Virtual-SM, we will introduce some related techniques: virtual texture and GPU-driven rendering, since Virtual-SM's idea is similar to the above techniques.

## Virtual Texture

The virtual texture is proposed based on a technique called mega texture. Like virtual memory in OS, virtual texture assumes the texture resolution is large enough and does not require the entire texture to be loaded. Instead, we only need to load the part of the texture that is actually used in the current view.

Virtual texture has many variations depending on the situation: streaming virtual texture (SVT), run-time virtual texture (RVT), and its improved version adaptive virtual texture (AVT). The ideas behind SVT/RVT/AVT are also contained in the Virtual-SM.

### Streaming Virtual Texture

Streaming virtual texture has four parts:

1.Divided the virtual texture into NxN tiles or pages.
2.Collect the tile and the mipmap of the texture required in the current view. This step can be executed in a separate pass or combined with the PreZ / GBuffer Pass.
3.Feedback and analyze the results and load the texture tiles used.
4.Construct and update the indirect texture, which stores the coordinates mapping from virtual texture to physical texture.

Compared to mipmap texture streaming, SVT has the following advantages:

1.Save video memory (we only load the texture tiles required).
2.Finer granularity (the minimum load unit is texture tile rather than texture mip).
3.More accurately (mip texture streaming is computed on CPU and will get wrong results for complex texture types, such as Atlas. SVT is computed on the GPU based on the ddx/ddy operation).

### Runtime Virtual Texture

The texture content of SVT is loaded from the disk, that is, generated offline. And RVT generates texture content at runtime. It is most used in terrain rendering. The key idea of RVT in terrain rendering is caching the texture blend result into a runtime virtual texture. This has several advantages compared to the SVT and traditional splat map method:

1.The texture size is too large to store on the disk if we use SVT to blend the terrain texture offline
2.We need to sample many times for terrain textures (normal / base color/splat map) for the traditional splat map method. There are only four channels in a splat map, so we can only record four terrain weights, which limits the art effect.
3.RVT has better performance than the splat map method, and the texture size stored on disk is smaller than SVT. In other words, RVT is a compromise solution to SVT and splat map.

### Adaptive Virtual Texture

Adaptive virtual texture is an improved version based on the RVT. We will get a large indirect table if the terrain is wide enough. AVT solves this problem by allocating different VT sizes depending on camera distance.

## GPU-Driven Rendering

In virtual shadow map, each shadow map tile corresponds to a frustum. This means that Virtual-SM will perform culling many times, which is unaffordable for the CPU. GPU-driven rendering can solve these problems by moving culling and submitting tasks from CPU to GPU. Let's introduce the basic GPU-driven pipeline first. 

GPU-driven rendering pipeline has three parts: preparing GPU data, culling on GPU, and work submission.

### Prepare GPU data

1.Before submitting data to the GPU, we can perform a coarse quad tree culling on the CPU, which reduces the data uploaded to the GPU.
2.Prepare the data that will be updated. This includes 1. instance/primitive data (transform/lod factor/bounding box), 2. Light map data etc.
3.Batch draw calls and update the GPU data required in the next step

### Culling on GPU

Culling on GPU, contains the following steps:
1.Instance culling, perform frustum culling and HIZ culling on GPU. And generate the cluster chunk for those instances passing culling.
2.Perform the frustum and occlusion culling for each cluster based on the cluster's transform and bound. And perform back-face culling based on cluster orientation precomputed offline.
3.Index buffer compaction, culling the index unused.

### Work Submission

Generate commands buffer and use indirect draw to submit the work.

# A brief introduction to Virtual-SM

Virtual-SM assumes the shadow map resolution is large enough, similar to virtual texture. The shadow map is split into many tiles and only renders the tiles required in the current view.

In general, Virtual-SM can be divided into the following passes:

1.Mask the tiles required in the current view based on the Pre-Z depth texture.
2.Generate an indirect table that maps coordinates from virtual texture to physical texture.
3.Collect shadow caster objects and prepare GPU data.
4.Using GPU to cull shadow casters and generate indirect draw commands.
5.Work submission and shadow projection.
6.Generate a shadow mask texture.

# Virtual Shadow Map Implmentation

## Mask the tiles

We allocate a buffer with the size of the virtual tile number. This buffer stores whether the corresponding tile is required by the current view.

```cpp
m_vsmTileFlagBuffer.Create(VSM_VIRTUAL_TILE_SIZE_WH * VSM_VIRTUAL_TILE_SIZE_WH, sizeof(uint32), DXGI_FORMAT_R32_UINT, CDeviceObjectFactory::BIND_SHADER_RESOURCE | CDeviceObjectFactory::BIND_UNORDERED_ACCESS, NULL);
```

Reconstruct the world position from the input scene depth texture, and project it into shadow space. Then, mask the tiles if UV is valid.

```cpp
float4 worldPosition = ......;
float4 shadowScreenPOS = mul(worldPosition,c_tileFlagGenConstants.lightViewProj);
shadowScreenPOS.xyz /= shadowScreenPOS.w;

float2 uvVSM = ......;

if(all(abs(uvVSM) < 1.0f))
{
    uint2 uvBuffer = uvVSM * c_tileFlagGenConstants.vsmVirtualTileNum.xy;
    uint bufferIndex = uvBuffer.y * c_tileFlagGenConstants.vsmVirtualTileNum.x + uvBuffer.x;
    p1_vsmTileFlags[bufferIndex] = 1;
}
```

## Generate the indirect table

Allocate an indirect table buffer with the virtual tile number size. This table maps the virtual texture tile to the physical texture tile. And we need a buffer to count the current tile number.

```cpp
m_vsmTileTableBuffer.Create(VSM_VIRTUAL_TILE_SIZE_WH * VSM_VIRTUAL_TILE_SIZE_WH, sizeof(uint32), DXGI_FORMAT_UNKNOWN, CDeviceObjectFactory::USAGE_STRUCTURED | CDeviceObjectFactory::BIND_UNORDERED_ACCESS | CDeviceObjectFactory::BIND_SHADER_RESOURCE, NULL);
...
m_vsmValidTileCountBuffer.Create(1, sizeof(uint32), DXGI_FORMAT_UNKNOWN, CDeviceObjectFactory::USAGE_STRUCTURED | CDeviceObjectFactory::BIND_UNORDERED_ACCESS, NULL);
```

Generate an indirect table from the tile flag buffer. For each virtual texture tile, compute its physical texture tile index. If this tile is masked as required, use InterlockedAdd to count the tile index, and store the result into indirect table.

```cpp
if(p2_vsmTileFlags[bufferIndex] == 1)
{
    p2_validTileCount.InterlockedAdd(0,uint(1),tileIndex);
}
......
p2_vsmTileTable[bufferIndex] = tileIndex;
```

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/VSM2.jpg)

## Prepare GPU data

Each Virtual-SM tile correspond to a shadow frustum, which is incompatible with the existing frustum type in CryEngine. We need to add a new frustum type: e_VSM. Then, create and update the e_VSM Frustum in the function UpdateGSMLightSourceShadowFrustum. Additionally, skip the shadow casters culling in COctreeNode::UpdateCullMask, since vsm perform culling on GPU. 

After that, all rendered items are stored in ShadowView. With the data required for GPU culling ready, we can update the GPU data. GPU data contains the following members:

1.Instance data per render item. It contains two members:
Instance transform and instance bounding box, used for GPU culling.

```cpp
struct SRenderItemGPUData
{
	Matrix44 m_Matrix;
	
	Vec3 m_boundingBoxCenter;
	float padding_0;
	Vec3 m_boundingBoxExtent;
	float padding_1;
};
```
2.GPU draw command, used for indirect draw, contains the following members:
2.1 Shader group index (Index to PSO array, will be described below).
2.2 Constant buffers (Including per-object buffer and per-shadow frustum buffer).
2.3 Index buffer and vertex buffer
2.4 The indirectct draw command
```cpp
struct CDeviceGPUDrawCmd
{
	uint32 m_shaderGroupIndex;

	std::vector<CDeviceBuffer*> m_constBuffers;

	const CDeviceInputStream* m_indexStreamSet;
	const CDeviceInputStream* m_vertexStreamSet;

	int32 IndexCountPerInstance;
	int32 InstanceCount;
	int32 StartIndexLocation;
	int32  BaseVertexLocation;
	uint32 StartInstanceLocation;
};
```
Iterate over all rendered items, get the data needed, and fill the culling data buffer and draw the command buffer.

```cpp
	const SRendItem& ri = (*renderItems)[i];
	......
	//gpu culling data
	m_riGpuCullingData.push_back(
		SRenderItemGPUData
		{
			ri.pCompiledObject->GetInstancingData().matrix,
			ri.pCompiledObject->m_aabb.GetCenter(),
			0,
			ri.pCompiledObject->m_aabb.max - ri.pCompiledObject->m_aabb.GetCenter(),
			0
		});

	//pso array todo:cache pso
	m_renderItemsPSO.push_back(ri.pCompiledObject->m_pso[m_stageID][m_passID].get());

	//indirect draw command
	std::vector<CDeviceBuffer*>constBuffers;
	constBuffers.push_back(ri.pCompiledObject->m_perDrawCB->m_buffer);
	constBuffers.push_back(ri.pCompiledObject->m_perDrawCB->m_buffer);//placeholder buffer,set on gpu
	m_gpuDrawCommands.push_back(
		CDeviceGPUDrawCmd
		{
			......
		}
	);
```


## GPU Culling

Create and update the Frustum projection buffer, store the project matrix and Frustum offset. Then, gather the GPU addresses of these buffers, and upload the address buffer to the GPU.

```cpp
for (uint32 indexX = 0; indexX < VSM_VIRTUAL_TILE_SIZE_WH; indexX++)
{
	for (uint32 indexY = 0; indexY < VSM_VIRTUAL_TILE_SIZE_WH; indexY++)
	{
		Matrix44A lightViewProjMatrix = ......;
		SShadowProjectMatrix ShadowProjectMatrix{ lightViewProjMatrix ,indexX,indexY };
		m_vsmFrustumProjectBuffer[indexX + indexY * VSM_VIRTUAL_TILE_SIZE_WH]->UpdateBuffer(&ShadowProjectMatrix,sizeof(SShadowProjectMatrix));
	}
}
```

Perform GPU culling. It contains the following steps:
1.Compute the aabb corner of the bounding box
```cpp
float4 corner[8];
[unroll]
for(uint i = 0; i < 8 ; i++)
{
    corner[i] = float4(rhiGpuCullingData[index].boundingBoxCenter ,0)  + float4(rhiGpuCullingData[index].boundingBoxExtent,0) * BoundingBoxOffset[i];
}
```

2.Project it to shadow space and get the coverage of the shadow tile of this bonding box
```cpp
[unroll]
for(uint j = 0; j < 8 ; j++)
{
    float4 screenPosition = mul(float4(corner[j].xyz,1.0f),c_VSMCmdBuildConstants.lightViewProj);
    screenPosition.xyz/=screenPosition.w;
    float2 screenUV = ......;
    uvMin = min(uvMin,screenUV);
    uvMax = max(uvMax,screenUV);
}
```

3.For each tile covered by a bounding box, create a GPU draw command if this tile is masked as visible in the first pass. Then use a counter buffer and RWStructuredBuffer to simulate the append buffer. Get the address of the corresponding Frustum constant buffer, and assign this value to the GPU draw command. Get the index of this draw command by InterlockedAdd, then write the result to the outputCommands buffer for work submission. This may split a draw call into many DC, we can improve it in future work.
```cpp
for(uint indexX = tileIndexMin.x ; indexX <= tileIndexMax.x ; indexX++)
{
    for(uint indexY = tileIndexMin.y ; indexY <= tileIndexMax.y ; indexY++)
    {
        ......
        if(visible >= 0)
        {
            uint cmdIndex = 0;
            p3_vsmCmdCount.InterlockedAdd(0,uint(1),cmdIndex);
            
            SvsmDrawCmd cmd = (SvsmDrawCmd)0;
            if(cmdIndex < c_VSMCmdBuildConstants.cmdBuildPara.z)
            {
                cmd.addressPerVSMView = frustumMatrixAddress[indexY * tileIndexMax.x + indexX];
                outputCommands[cmdIndex] = cmd;
                return;
            }
        }
    }
}
```

## Work Submission

We need to introduce two prerequisite techniques, Vulkan buffer device address and Vulkan NV device-generated commands. They are essential parts of the GPU work submission.

### Vulkan buffer device address

Vulkan buffer device address extension allows to fetch raw GPU pointers to a buffer and pass it for usage in a shader code.

#### Create a buffer

To create a buffer that enables BDA, we need to set VKBufferCreateInfo usage as VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT, indicating that this buffer can be accessed by device address in the shader. And set the VMA allocate flag to VK_MEMORY_ALLOCATE_DEVICE_ADDRESS_BIT.

```cpp
VkMemoryAllocateFlagsInfoKHR flags_info{VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_FLAGS_INFO_KHR};
flags_info.flags             = VK_MEMORY_ALLOCATE_DEVICE_ADDRESS_BIT_KHR;
memory_allocation_info.pNext = &flags_info;
```

Use vkGetBufferDeviceAddressKHR to get the buffer address. The return result type is VkDeviceAddress (uint64_t).

#### Vulkan GLSL

Here is an example of a buffer device address. We get the GL_EXT_buffer_reference extension which allows us to declare buffer blocks not as SSBOs, but faux pointer types instead.

```cpp
#extension GL_EXT_buffer_reference : enable

layout(buffer_reference, buffer_reference_align=16, scalar) readonly buffer ExamplelBuffer {
  BufferData  data;
};
  
layout(push_constant, scalar) uniform pushConstants {
  layout(offset=8)
  ExamplelBuffer v;
};
```
buffer_reference tags the type accordingly, and buffer_reference_align is used to mark that any pointer which is of this type is at least 16 bytes aligned. 

Finally, we place the buffer reference inside the push constant. Buffer_reference can also be placed inside SSBO or UBO. The reason we use push constant is that Device Generated Commands discussed later only support push constant.

### Vulkan NV device-generated commands

One problem with existing indirect techniques is that the command buffer must be prepared for the worst-case scenario on the CPU. You may also require more memory than necessary for the distribution of indirect commands, given that it is not known which state buckets they will end up in. To solve these problems, this extension adds further capabilities, such as the ability to switch between different shaders.


The following steps generate commands on the device:

1.Define a sequence of commands to be generated, using a VkIndirectCommandsLayoutNV object.
2.For the ability to change shaders, create the graphics pipelines with VK_PIPELINE_CREATE_INDIRECT_BINDABLE_BIT_NV and then create an aggregate graphics pipeline that imports those pipelines as graphics shader groups using VkGraphicsPipelineShaderGroupsCreateInfoNV, which extends VkGraphicsPipelineCreateInfo.
3.Create a preprocess buffer based on sizing information acquired using vkGetGeneratedCommandsMemoryRequirementsNV.
4.Fill the input buffers for the generation step and set up the VkGeneratedCommandsInfoNV struct accordingly. It is used in preprocessing as well as execution. Some commands require buffer addresses, which can be acquired using vkGetBufferDeviceAddress.
5.Run the execution via vkCmdExecuteGeneratedCommandsNV.

Most engines don't support Vulkan device-generated commands. We need to develop this extension for CryEngine. The first step is to upgrade CryEngine‘s Vulkan SDK version, since it does not match the current DGC extension version :(

#### Define indirect command layout

Create indirect command layout tokens. It contains five types:

1.Shader group type, indexing to PSO this command used.
2.Index/Vertex buffer type
3.Push constant type, storing the constant buffer this command requires.
4.Draw indexed type, which describes the draw parameters, such as instance count, start vertex location,etc.

```cpp
struct TMP_RENDER_API SDeviceResourceIndirectLayoutToken
{
	enum  class ETokenType : uint8
	{
		TT_ShaderGroup,
		TT_IndexBuffer,
		TT_VertexBuffer,
		TT_PushConstant,
		TT_DrawIndexd,
	};

	ETokenType m_tokenType;

	//push constant
	CDeviceResourceLayoutPtr m_pDeviceResourceLayout;
	EShaderStage m_pcShaderStage;
	uint32 m_pcOffset;
	uint32 m_pcSize;
};
```

Then, add tokens to the indirect desc. Since the virtual shadow map draw command needs two constant buffers, the size of the push constant is sizeof(VkDeviceAddress) * 2.

```cpp
    ......
	SDeviceResourceLayoutDesc layoutDesc;
	layoutDesc.SetResourceSet(EResourceLayoutSlot_PerPassRS, m_perPassResources);
	layoutDesc.AddPushConstant(EShaderStage_Vertex | EShaderStage_Pixel, sizeof(uint64) * 2, 0);
	m_pResourceLayout = GetDeviceObjectFactory().CreateResourceLayout(layoutDesc);//see m_graphicsPipeline.CreateScenePassLayout(m_perPassResources);
	m_perPassResources.AcceptChangedBindPoints();

	SDeviceResourceIndirectLayoutDesc deviceResourceIndirectLayoutDesc;
	deviceResourceIndirectLayoutDesc.AddPushConstant(m_pResourceLayout, EShaderStage_Vertex | EShaderStage_Pixel, 0, sizeof(uint64) * 2);
	......
	deviceResourceIndirectLayoutDesc.AddDrawIndexed();
```
For each RHI layout token, convert it to Vulkan Layout token. Push constant needs to be handled specially, we should assign VkPipelineLayout to member pushconstantPipelineLayout.

```cpp
	std::vector<VkIndirectCommandsLayoutTokenNV> inputInfos;
	
	uint32 offsetGloabl = 0;

	for (auto iter = desc.m_indirectLayoutTokens.begin(); iter != desc.m_indirectLayoutTokens.end(); iter++)
	{
		VkIndirectCommandsTokenTypeNV tokenTypeNV = ConvertToVKIndirectCmdType(iter->m_tokenType);
		VkIndirectCommandsLayoutTokenNV input = { VK_STRUCTURE_TYPE_INDIRECT_COMMANDS_LAYOUT_TOKEN_NV, 0,tokenTypeNV };
		input.stream = 0;
		input.offset = offsetGloabl;

		offsetGloabl += ConvertToTTSize(iter->m_tokenType, iter->m_pcSize);
        
        ......

		if (tokenTypeNV == VK_INDIRECT_COMMANDS_TOKEN_TYPE_PUSH_CONSTANT_NV)
		{
			input.pushconstantPipelineLayout = static_cast<const CDeviceResourceLayout_Vulkan*>(iter->m_pDeviceResourceLayout.get())->GetVkPipelineLayout();
			......
		}

		inputInfos.push_back(input);
	}

	uint32_t interleavedStride = offsetGloabl;

	VkIndirectCommandsLayoutCreateInfoNV genInfo = { VK_STRUCTURE_TYPE_INDIRECT_COMMANDS_LAYOUT_CREATE_INFO_NV };
	genInfo.tokenCount = (uint32_t)inputInfos.size();
	genInfo.pTokens = inputInfos.data();
	genInfo.streamCount = 1;
	genInfo.pStreamStrides = &interleavedStride;

	VkResult result = VK_RESULT_MAX_ENUM;
	if (Extensions::EXT_device_generated_commands::IsSupported)
	{
		result = Extensions::EXT_device_generated_commands::CmdCreateIndirectCommandsLayout(m_pDevice->GetVkDevice(), &genInfo, NULL, &indirectCmdsLayout);
	}
```

In addition, we use the interleaved command layout in Virtual-SM, which is an array of structures (AOS). It leads to generating dynamic commands on GPU more easily. DGC provides a structure of arrays (SoA) layout for compact, cache-friendly streams.

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/image4-768x432.png)

#### Create PSO and proprocess buffer

For the ability to change shaders, create the graphics pipelines with VK_PIPELINE_CREATE_INDIRECT_BINDABLE_BIT_NV and then create an aggregate graphics pipeline that imports those pipelines as graphics shader groups using VkGraphicsPipelineShaderGroupsCreateInfoNV, which extends VkGraphicsPipelineCreateInfo.

```cpp
VkGraphicsPipelineShaderGroupsCreateInfoNV groupsCreateInfo = { VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_SHADER_GROUPS_CREATE_INFO_NV };
{
	std::vector<VkPipeline> referencedPipelines;
	for (uint32_t m = 1; m < psoDesc.indirectPso.size(); m++)
	{
		referencedPipelines.push_back(static_cast<CDeviceGraphicsPSO_Vulkan*>(psoDesc.indirectPso[m])->GetVkPipeline());
	}
	groupsCreateInfo.pPipelines = referencedPipelines.data();
	groupsCreateInfo.pipelineCount = (uint32_t)referencedPipelines.size();

	//m = 0;
	VkGraphicsShaderGroupCreateInfoNV shaderGroup = { VK_STRUCTURE_TYPE_GRAPHICS_SHADER_GROUP_CREATE_INFO_NV };
		
    ......
	groupsCreateInfo.groupCount = 1u;
	groupsCreateInfo.pGroups = &shaderGroup;
	graphicsPipelineCreateInfo.pNext = &groupsCreateInfo;
}
```
Create a preprocess buffer based on the indirect layout and indirect PSO prepared above. Preprocess buffer stores the information to execute generated commands.

```cpp

m_preprocessBuffer.CreatePreprocessBuffer(CRenerItemGPUDrawer::m_maxDrawSize, m_pResourceIndirectLayout, m_pIndirectGraphicsPSO);

void CDevice::CreatePreProcessBuffer(uint32 drawCount, VkIndirectCommandsLayoutNV indirectCmdsLayout, VkPipeline indirectPSO, CBufferResource* pOutputResource, uint32 countOffset, uint32& outSize) threadsafe
{
	VkGeneratedCommandsMemoryRequirementsInfoNV memInfo = { VK_STRUCTURE_TYPE_GENERATED_COMMANDS_MEMORY_REQUIREMENTS_INFO_NV };
	memInfo.maxSequencesCount = drawCount;
	memInfo.indirectCommandsLayout = indirectCmdsLayout;
	memInfo.pipeline = indirectPSO;
	memInfo.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;

	VkMemoryRequirements2 memReqs = { VK_STRUCTURE_TYPE_MEMORY_REQUIREMENTS_2 };
	if (Extensions::EXT_device_generated_commands::IsSupported)
	{
		Extensions::EXT_device_generated_commands::CmdGetGeneratedCommandsMemoryRequirements(GetVkDevice(), &memInfo, &memReqs);
	}
	
	VkBufferCreateInfo createInfo = { VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO };
	......

	VkBuffer hVkResource = VK_NULL_HANDLE;
	VK_ASSERT(vkCreateBuffer(GetVkDevice(), &createInfo, nullptr, &hVkResource));
	......
	CMemoryHandle boundMemory;
	if ((boundMemory = GetHeap().Allocate(memReqs.memoryRequirements, kHeapTargets)))
	{
		......
	}
}
```

#### Execution generated command and perform shadow rendering

All the data we needed for shadow map rendering is ready, it contains:

1.Culled GPU draw command buffer
2.Counter buffer
3.Indirect layout
4.PSO
5.Preprocess buffer

Fill in VkGeneratedCommandsInfoNV with the information prepared above and perform the final vkCommand : vkCmdExecuteGeneratedCommandsNV.

```cpp
......
#vkextbegin
[[vk::push_constant]]
SCbPointer cbPointer;
#vkextend

void VSMProjectPS(vtxOut IN)
{
#vkextbegin
    uint2 IndexXY = vk::RawBufferLoad<uint2>(cbPointer.PerViewAddress + 16,4);
#vkextend

    uint PageTableIndex = PagetableInfos[IndexXY].x;//TODO : Move To Vertex Shader
    uint PhysicalIndexX = PageTableIndex % uint(PhysicalTileWidthNum);
    uint PhysicalIndexY = PageTableIndex / uint(PhysicalTileWidthNum);
    
    float2 UVWtrite = IN.PositionIn.xy / TileSize;

    float2 WritePos = float2(PhysicalIndexX,PhysicalIndexY) + UVWtrite;
    WritePos/=PhysicalTileWidthNum;
    WritePos*=PhysicalSize;

    uint UintDepth = IN.PositionIn.z * (1<<30);
    InterlockedMax(PhysicalShadowDepthTexture[uint2(WritePos)],UintDepth);
}
```
In the shader code, we get the page table index by vk::RawBufferLoad for the address stored in vk::push_constant SCbPointer. We add a custom parse step for the Vulkan-HLSL extension, since CryEngine couldn't parse these Vulkan tokens.

Transform the depth to uint, and use InterlockedMax to store the result depth.

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/VSM2.jpg)

## Shadow Mask Generation

By looking up the indirect page table with the virtual tile index, we get the physical tile index when creating the shadow mask texture.And get the physical shadow texture coords by TileIndex and SubTileUV.

```cpp
uint ObjectShadowDepth = (ShadowScreenPOS.z)* (1<<30);

float ComputeShadowFactor(float2 UVShadowSpace ,uint ObjectShadowDepth , uint Bias)
{
    uint2 TileIndexXY = uint2(UVShadowSpace * PageNum /*- 0.5f*/);
    uint PageTableIndex = PagetableInfos[TileIndexXY].x;

    uint PhysicalIndexX = PageTableIndex % uint(PhysicalTileWidthNum);
    uint PhysicalIndexY = PageTableIndex / uint(PhysicalTileWidthNum);

    float2 SubTileUV = (UVShadowSpace * PageNum) - uint2(UVShadowSpace * PageNum);
    float2 ShadowDepthPos = float2(PhysicalIndexX,PhysicalIndexY) + SubTileUV; 
    ShadowDepthPos/=PhysicalTileWidthNum;
    ShadowDepthPos*=PhysicalSize;

    uint ShadowDepth = PhysicalShadowDepthTexture[uint2(ShadowDepthPos)].x;

    if((ObjectShadowDepth + Bias )< ShadowDepth) 
    {
        return 1.0f;
    }

    return 0.0;
}
```

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/mask.jpg)


