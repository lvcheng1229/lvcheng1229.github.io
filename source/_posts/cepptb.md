---
title: Implement tiled-Based indirect bloom in CryEngine
date: 2023-07-30 23:08:25
tags:
---

The complexity of postprocess effects varies depending on the screen tile they are applied to. For example, we can only take one sample(disable blur) for pixels whose velocity is relatively low using branching. However, dynamic branching on the GPU is costly since we cannot ensure that multiple threads don't cause divergence.

The probability of divergence will be significantly reduced if different shaders can be executed on different screen tiles. Therefore, the problem is how to dispatch different shaders on various tiles. 

Indirect-Compute Shader is an answer: In the first step, dispatch a compute shader to classify the tiles and determine the tile count and index for each shader complexity level. Then, dispatch a variety of indirect-compute shaders for each tile with the arguments generated in the last step.The result is that each tile performs a specific version of the shader and eliminates divergence.

This article will take the bloom pass as an example of implementing the tile-based post process.

The bloom algorithm is based on the implementation of CryEngine in this demo. So, we are going to discuss the implementation of CryEngine first. There are three parts to CryEngine's bloom algorithm. The first part is downsampling. CryEngine takes two passes to downsample the render target. The output of the downsample is a quarter-resolution render target relative to the original RT. Then, CryEngine performs a Gaussian blur with a quarter-resolution render target. In this step, CE takes four passes to blur the source texture: two for vertical and two for horizontal. Each pass samples the texture 16 times, so the total number of samples is 64 * Resolution! Finally, CE performs a lerp between scene color and bloom results as the result in the tone mapping pass.

What we are going to do is optimize its sample count. Here is the flow graph of tiled-based bloom algorithm:

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/flowinfo2.jpg)

**Step1.Mask screen tiles contain pixels whose luminance exceeds the threshold.**

Each tile corresponds to a dispatch group with 8x8 threads and contains 8x8 pixels. We use a group shared unit array to record group luminance info. 

```cpp
groupshared uint MaskInfo[GROUP_SIZE_X*GROUP_SIZE_Y*1];

if(Luminance > BloomThreshlod)
{
	Result.rgb = BloomOUT.rgb;
	MaskInfo[GroupThreadID.y * GROUP_SIZE_X + GroupThreadID.x] = 1;
}
```

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/maskbuffer2.jpg)

Then, perform parallel reduction within each thread block to determine if this tile contains at least one pixel that exceeds the threshold. As a result, we will get a mask buffer with the size of one eighth of the input bloom texture.

```cpp
GroupMemoryBarrierWithGroupSync();

[unroll]
for (uint SetpIndex = 1; SetpIndex < 4; SetpIndex++)
{
	const uint TileSize = uint(GROUP_TILE_SIZE) >> SetpIndex;
	const uint ReduceBankSize = TileSize * TileSize;
	[branch]
	if (GroupThreadIndex < ReduceBankSize)
	{
		uint RawMask[4];
		[unroll]
		for (uint i = 0; i < 4; i++)
		{
			uint LDSIndex = GroupThreadIndex + i * ReduceBankSize;
			RawMask[i] = MaskInfo[LDSIndex];
		}
		uint MaskCombine = RawMask[0] || RawMask[1] || RawMask[2] || RawMask[3];
		MaskInfo[GroupThreadIndex] = MaskCombine;
	}
}

if(GroupThreadIndex == 0)
{
	BloomMaskOutUAV[GroupID.y * bloomCB.BufferSize.x + GroupID.x] = MaskInfo[0];
}
```

**Step2.Record the tile index and calculate the number of tiles that will perform the Gaussian blur.**

The results will be used as indirect arguments in the next step.

The kernel size of the first two Gaussian blur passes is 7 and each member of the mask buffer represents 8x8 pixels. Therefore, the tile being masked will affect the 3x3 tiles around it. 

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/maskbuffer3.jpg)

```cpp

uint CIndex = DispatchThreadID.y * MaskBufferSize.x + DispatchThreadID.x;
uint LCIndex = CIndex - 1;
uint RCIndex = CIndex + 1;

uint TIndex = (DispatchThreadID.y - 1) * MaskBufferSize.x + DispatchThreadID.x;
uint LTIndex = TIndex - 1;
uint RTIndex = TIndex + 1;
uint BIndex = (DispatchThreadID.y +1 ) * MaskBufferSize.x + DispatchThreadID.x;
uint LBIndex = BIndex - 1;
uint RBIndex = BIndex + 1;

uint CValue = IndexGen_MaskBuffer[CIndex].x;
uint LCValue = IndexGen_MaskBuffer[LCIndex].x;
uint RCValue = IndexGen_MaskBuffer[RCIndex].x;
uint TValue = IndexGen_MaskBuffer[TIndex].x;
uint LTValue = IndexGen_MaskBuffer[LTIndex].x;
uint RTValue = IndexGen_MaskBuffer[RTIndex].x;
uint BValue = IndexGen_MaskBuffer[BIndex].x;
uint LBValue = IndexGen_MaskBuffer[LBIndex].x;
uint RBValue = IndexGen_MaskBuffer[RBIndex].x;

uint CMaskValue = CValue || LCValue || RCValue;
uint TMaskValue = TValue || LTValue || RTValue;
uint BMaskValue = BValue || LBValue || RBValue;
uint MaskValue = CMaskValue || TMaskValue || BMaskValue;

if(MaskValue)
{
	uint PreCountOut;
	IndexGen_DispatchIndirectCount.InterlockedAdd(0, uint(1), PreCountOut);
	const uint TileIndex = (DispatchThreadID.x & 0xFFFF) | ((DispatchThreadID.y & 0xFFFF) << 16);
	IndexGen_TileIndexUAV[PreCountOut] = TileIndex;
}

```
We use the InterlockedAdd function to count the tile number and store the result in an RWByteAddressBuffer.

```cpp
RWByteAddressBuffer 			IndexGen_DispatchIndirectCount	: register(u1);

if(all(DispatchThreadID == 0))
{
	IndexGen_DispatchIndirectCount.Store3(0,uint3(0,1,1));
}

......

if(MaskValue)
{
	uint PreCountOut;
	IndexGen_DispatchIndirectCount.InterlockedAdd(0, uint(1), PreCountOut);
}
```

Taking the above figure as an example, the value of the DispatchIndirectCount buffer is (12,1,1) and the value of the tile index buffer is (4,2), (4,3), (4,4), ......, (5,2), (5,3), (5,4),......


![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/maskbuffer4.jpg)

The red blocks are the tiles that performed Gaussian blur in the first two passes, and their size is 8x8 pixels per tile. Next, we will generate the mask buffer for the last two Gaussian blur passes based on the above results. In the last two blur passes, the tile size is 16x16 since the kernel size in the last two blur passes is 15.

```cpp
		if(MaskValue)
		{
			uint PreCountOut;
			IndexGen_DispatchIndirectCount.InterlockedAdd(0, uint(1), PreCountOut);
			const uint TileIndex = (DispatchThreadID.x & 0xFFFF) | ((DispatchThreadID.y & 0xFFFF) << 16);
			IndexGen_TileIndexUAV[PreCountOut] = TileIndex;
#if %_RT_SAMPLE0
			IndexGen_TileMask[GroupThreadID.y * GROUP_SIZE_X + GroupThreadID.x] = 1;
#endif
		}

#if %_RT_SAMPLE0
		if(GroupThreadIndex % 4 == 0)
		{
			uint RawMask[4];
			[unroll]
			for (uint i = 0; i < 4; i++)
			{
				uint LDSIndex = GroupThreadIndex + i;
				RawMask[i] = IndexGen_TileMask[LDSIndex];
			}
			uint MaskCombine = RawMask[0] || RawMask[1] || RawMask[2] || RawMask[3];

			IndexGen_MaskOutputUAV[(DispatchThreadID.y / 2) * (MaskBufferSize.x / 2) + DispatchThreadID.x / 2] = MaskCombine;
		}
#endif
```

Repeat step 2. We will get all the parameters needed in the Gaussian blur pass.

**Step3. Dispatch indirect compute shader and perform Gaussian blur.**

The dispatch size is (12,1,1) in this example.

```cpp
m_passBloom[IndexPass][IndexAxis]->SetDispatchIndirectArgs(&m_dispatchIndirectCount[IndexPass], 0);
```

Get the index for those tiles performing Gaussian blur indexed by GroupID from the tile index buffer and calculate the global UV coordinates based on the tile index.

```cpp
uint TileIndex = BloomFinal_TileInfoUAV.Load(GroupID.x).x;
uint2 TileIndexXY = uint2(TileIndex & 0xFFFF,(TileIndex >> 16) & 0xFFFF);
uint2 UVCoord = TileIndexXY * uint2(GROUP_SIZE_X * 2,GROUP_SIZE_Y * 2) + uint2(GroupThreadIndex % (GROUP_SIZE_Y * 2), GroupThreadIndex / (GROUP_SIZE_X * 2));
```
Finally, blur the input texture. Here is the code copied from CryEngine's Gaussian blur implementation:

```cpp
	const float weights[15] = { 153, 816, 3060, 8568, 18564, 31824, 43758, 48620, 43758, 31824, 18564, 8568, 3060, 816, 153 };
	const float weightSum = 262106.0;
	
	float2 coords = float2(UVCoord) * bloomTexSize.zw - bloomParams.xy * 7.0;
	half3 vColor = 0;

	[unroll]
	for (int i = 0; i < 15; ++i)
	{
		vColor += GetTexture2DLod(BloomFinal_Tx2Tx_Source, BloomFinal_Tx2Tx_Sampler, coords,0.0).rgb * (weights[i] / weightSum);
		coords += bloomParams.xy;
	}

	BloomFinal_OutputUAV[UVCoord] = float4(vColor,1.0);
```

As Cryengine does not support indirect compute shaders, we must expand its renderer to support them.

```cpp
void CDeviceComputeCommandInterfaceImpl::DispatchIndirectImpl(const CDeviceBuffer* pBuffer, uint32 Offset)
{
	const CDeviceResourceLayout_Vulkan* pVkLayout = reinterpret_cast<const CDeviceResourceLayout_Vulkan*>(m_computeState.pResourceLayout.cachedValue);

	ApplyPendingBindings(GetVKCommandList()->GetVkCommandList(), pVkLayout->GetVkPipelineLayout(), VK_PIPELINE_BIND_POINT_COMPUTE, m_computeState.custom.pendingBindings);
	m_computeState.custom.pendingBindings.Reset();

	GetVKCommandList()->PendingResourceBarriers();
	vkCmdDispatchIndirect(GetVKCommandList()->GetVkCommandList(), pBuffer->GetBuffer()->GetHandle(), Offset);
}
```

This is the Vulkan capture result of Renderdoc.

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/vulkanresult.jpg)

|  | Renderdoc result |
| :-: | :-: |
| bloom input |![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/20230812213557.png)|
| bloom output|![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/20230812213642.png)|
| tone mapping output|![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/bloomresultj.jpg)|







