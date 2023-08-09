---
title: CryEngine PostProcess:Tiled Based Bloom Using Indirect Compute
date: 2023-07-30 23:08:25
tags:
---

The complexity of postprocess effects varies depending on the screen tile they are applied to. For example, we can only take one sample(disable blur) for pixels whose velocity is relatively low using branch operation. However, dynamic branch operation on the GPU is costly since we cannot ensure that multiple threads don't diverge.

The probability of divergence will be significantly reduced if different shaders can be executed on different screen tiles. Therefore, the problem is how to dispatch different shaders on various tiles. 

The answer is: Indirect-Compute Shader: In the first step, dispatch a compute shader to classify the tiles and determine the tile count and index for each shader complexity level. Then, dispatch a variety of indirect-compute shaders for each tile with the arguments generated in the last step.The result is that each tile performs a specific version of the shader and eliminates divergence.

This article will take the bloom pass as an example of implementing the tile-based post process.

The bloom algorithm is based on the implementation of CryEngine in this demo. So, we are going to discuss the implementation of CryEngine first. There are three parts to CryEngine's bloom algorithm. The first part is downsampling. CryEngine takes two passes to downsample the render target. The output of the downsample is a quarter-resolution render target relative to the original RT. Then, CryEngine performs a Gaussian blur with a quarter-resolution render target. In this step, CE takes four passes to blur the source texture: two for vertical and two for horizontal. Each pass samples the texture 16 times, so the total number of samples is 64 * Resolution! Finally, CE performs a lerp between scene color and bloom results as the result in the tone mapping pass.

The tiled-based bloom has four parts: 


1.Mask screen tiles contain pixels whose luminance exceeds the threshold.

```cpp
if(Luminance > BloomThreshlod)
{
	Result.rgb = BloomOUT.rgb;
	MaskInfo[GroupThreadID.y * GROUP_SIZE_X + GroupThreadID.x] = 1;
}
```


![flowinfo](https://cdn.jsdelivr.net/gh/lvcheng1229/lvcheng1229.github.io@main/PicGoImgflowinfo.jpg)


