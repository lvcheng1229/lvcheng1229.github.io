---
title: CryEngine PostProcess:Tiled Based Bloom Using Indirect Compute
date: 2023-07-30 23:08:25
tags:
---

The complexity of postprocess effects varies depending on the screen tile they are applied to. For example, we can only take one sample(disable blur) for pixels whose velocity is relatively low using branch operation. However, dynamic branch operation on the GPU is costly since we cannot ensure that multiple threads don't diverge.

The probability of divergence will be significantly reduced if different shaders can be executed on different screen tiles. Therefore, the problem is how to dispatch different shaders on various tiles. 

The answer is: Indirect-Compute Shader! How to implement this? The first step is to dispatch a compute shader to classify the tiles and determine the tile count and index for each shader complexity level. Then, dispatch a variety of indirect-compute shaders for each tile with the arguments generated in the last step.The result is that each tile executes a specific version of the shader and eliminates execution divergence.

This article will take the bloom pass as an example of implementing the tile-based post process. 



