---
title: CryEngine PostProcess:Tiled Based Bloom Using Indirect Compute
date: 2023-07-30 23:08:25
tags:
---

The complexity of postprocess effects varies depending on the screen tile they are applied to. For example, we can only take one sample(disable blur) for pixels whose velocity is relatively low using branch operation. However, dynamic branch operation on the GPU is costly since we cannot ensure that multiple threads don't diverge.

The probability of divergence will be significantly reduced if different shaders can be executed on different screen tiles. Therefore, the problem is how to dispatch different shaders on various tiles. 

Indirect-Compute Shader is an answer: In the first step, dispatch a compute shader to classify the tiles and determine the tile count and index for each shader complexity level. Then, dispatch a variety of indirect-compute shaders for each tile with the arguments generated in the last step.The result is that each tile performs a specific version of the shader and eliminates divergence.

This article will take the bloom pass as an example of implementing the tile-based post process.

The bloom algorithm is based on the implementation of CryEngine in this demo. So, we are going to discuss the implementation of CryEngine first. There are three parts to CryEngine's bloom algorithm. The first part is downsampling. CryEngine takes two passes to downsample the render target. The output of the downsample is a quarter-resolution render target relative to the original RT. Then, CryEngine performs a Gaussian blur with a quarter-resolution render target. In this step, CE takes four passes to blur the source texture: two for vertical and two for horizontal. Each pass samples the texture 16 times, so the total number of samples is 64 * Resolution! Finally, CE performs a lerp between scene color and bloom results as the result in the tone mapping pass.

Tiled-based bloom has four parts: 

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/flowinfo2.jpg)

1.Mask screen tiles contain pixels whose luminance exceeds the threshold.

Each tile corresponds to a dispatch group and contains 8x8 pixels, which correspond to 8x8 threads. We use a group shared unit array to record tile luminance info. 

```cpp
groupshared uint MaskInfo[GROUP_SIZE_X*GROUP_SIZE_Y*1];

if(Luminance > BloomThreshlod)
{
	Result.rgb = BloomOUT.rgb;
	MaskInfo[GroupThreadID.y * GROUP_SIZE_X + GroupThreadID.x] = 1;
}
```

![](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImg/maskbuffer.jpg)

Then, perform parallel reduction within each thread block. As a result, we will get a mask buffer with the size of one eighth of the input bloom texture. The buffer indicates whether the corresponding tile should be blurred.

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