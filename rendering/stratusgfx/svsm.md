---
permalink: /rendering/stratusgfx/svsm
title: Sparse Virtual Shadow Maps
layout: page
use_math: true
---

![bistro1](/assets/v0.11/svsms/VSM1_Bistrov2.png)
![bistro2](/assets/v0.11/svsms/VSM2_Bistrov2.png)

The top is the Bistro scene rendered with multiple 8K resolution sparse virtual shadow maps. The bottom is a visualization of the physical memory pages (squares) where each color represents a different shadow clipmap/cascade.

**This is a WIP/rough draft! Feedback is greatly appreciated.**
**Last edited: Oct 24, 2023**

# Collaborators

This was developed in loose collaboration with:

* [Jaker v3](https://juandiegomontoya.github.io)
* [LVSTRI](https://github.com/LVSTRI)

# Table of Contents
* TOC
{:toc}

# Introduction

This post goes through the high-level steps needed to create a virtual memory system for realtime shadows. This was inspired by Unreal Engine 5's virtual shadow maps. Directional lights are the only ones considered here, but it is possible to extend the system to support other light types such as point or spotlights. 

The directional shadows make use of clipmaps for incremental updating of the shadowmaps and handling multiple different cascades. Each clipmap cascade can be set to very high virtual resolutions such as 4K, 8K or 16K.

### Sparsity

Part of the strength of this method comes from its ability to represent shadows using sparse textures. If nothing is currently present in one of the clipmap cascades, that cascade's memory footprint and processing overhead can be reduced to almost nothing.

### Virtual

This system replaces normal shadow map uv coordinates with virtual uv coordinates. To either write to or read from the shadow maps, virtual uv coordinates have to be translated to physical uv coordinates. The actual physical memory pool can be much smaller than what the virtual uv coordinate space would suggest. Conceptually this is extremely similar to virtual memory systems in modern operating systems.

### Caching

As the camera moves around the scene, new virtual pages become visible and are rendered incrementally. Previously rendered pages that are still visible and haven't changed can be reused from frame to frame to save on performance. For either debugging purposes or to make it easier to build an initial prototype, caching can be disabled or skipped in favor of being added later.

# Motivation and Comparison

The motivations for using this approach fall into two categories:

* Significant increase in shadow map resolution while maintaining highly configurable (even dynamic) performance costs
* Provide the ability to cover massive world distances with a single shadow technique

### Comparison to Cascaded Shadow Maps

An important aspect mentioned in the introduction is that if a cascade doesn't have much geometry inside of it, processing overhead and memory footprint for that cascade can be reduced close to 0. This opens the possibility of maintaining 10, 15 or even 20 cascades. In addition, if the virtual resolution is set to 8K or 16K, every cascade gets its own 8K or 16K virtual shadow map. This allows for high quality, consistent shadows that cover huge world distances.

Standard cascaded shadow maps (CSMs) struggle to handle this without heavy optimization. In many ways sparse virtual shadow maps are an extension of CSMs with the optimizations required to improve memory usage, increase resolution, cover even larger max distances, and avoid duplicate shadow reprocessing for data that hasn't changed since last frame.

### Other Light Types

Even though this article focuses on directional lights, it is possible to virtualize other light types such as point or spotlights. To do this, the shadow maps for each light which use fixed texture allocations will be replaced with virtual shadow textures that are mostly sparse. For point lights this would result in 6 virtual textures (virtual cubemap) each with virtual mipmaps while spotlights would only need 1 virtual texture with virtual mipmaps. Physical backing memory for the virtual shadow textures of all the different light types can come from the same shared texture memory pool.

# Foundations

This technique is built off two core foundations: sparse virtual texturing and clipmaps.

### Sparse Virtual Texturing

*Helpful resources:*
* [Sparse Virtual Textures](https://www.youtube.com/watch?v=MejJL87yNgI)
* [Another Sparse Virtual Textures](https://studiopixl.com/2022-04-27/sparse-virtual-textures)
* [Understanding Virtual Memory](https://performanceengineeringin.wordpress.com/2019/11/04/understanding-virtual-memory/)

With regular virtual memory, all addresses used by a user mode application are virtual addresses. When the memory needs to be used in some way, the OS must translate the virtual addresses into physical addresses. Using this system, some of the bits of the virtual address are used as an offset into a translation (page) table, and that entry in the page table is used to determine where in physical memory to go. Each entry in the page table represents a physical page/block of memory with a fixed size (ex: 4 kb).

An important part of a virtual memory system is that it is not required to always have everything available in RAM. Parts that aren't needed can be paged out to secondary storage and kept there until requested. When a process makes a request to access a piece of memory that is either out of date or not present in RAM at all, this is known as a page fault. When this happens, it is the job of the OS to handle it by making sure that the memory the process is trying to use is present and up to date. Once this is done the process can resume execution.

Sparse virtual texturing uses the same approach. For this to work, each physical page of memory is a 2D block of size 128x128 texels (or some other power of two supported by the hardware). For an 8K x 8K virtual texture, this would result in 64x64 page table entries. This 64x64 grid is our version of the page table.

Like with other virtual memory systems, sparse virtual texturing only maintains the minimum amount of texture data in memory for the application to continue normally. It does this by analyzing the current camera view of the scene to determine what is needed verses what can be left unallocated.

At runtime, shaders will use virtual coordinates rather than physical coordinates. These virtual coordinates are translated to physical coordinates using the page table. When the data that the virtual coordinates reference is not currently resident in physical memory, this results in a page fault. Some mechanism (such as readback SSBO) needs to be used to tell the CPU to perform an allocation and prepare the physical memory for use.

Sparse virtual shadow mapping borrows all these ideas with one exception: page faults don't require us to read from secondary storage. Instead, page faults represent a piece of memory that we need to render shadows into so that the shaders can compute shadowing or post processing effects properly.

### Clipmaps

*Helpful resources:*
* [The Clipmap: A Virtual Mipmap](https://dl.acm.org/doi/pdf/10.1145/280814.280855)

Directional light shadow maps are represented by a series of expanding rings around the player camera known as clipmap rings/cascades. Conceptually they are similar to cascades in cascaded shadow maps.

A clipmap is an incrementally updatable, fixed-size texture region that is meant to represent a subset of a full texture's mipmap chain. A clipmap is defined by a clip origin and a clip size. Together these two determine which part of the full texture that the clipmap currently represents. As the clip origin shifts from frame to frame, new data is streamed in and old data streamed out along the borders of the texture. 

Each clipmap ring is required to cover double the area of the previous clipmap ring. This means that each successive ring is coarser/lower resolution than the previous since the memory footprint remains the same for each ring regardless of how much area they cover. Because coarser clipmap rings fully overlap finer clipmap rings, we can conceptualize them as being a version of texture mipmapping.

![clipmap](/assets/v0.11/svsms/VSM_Clipmap.png)

![clipmap2](/assets/v0.11/svsms/VSM_Clipmaps_As_Mipmaps.png)

When sampling from a clipmap, we select the most detailed (lowest ring/cascade) that's available. If not available, we clip to the next best fit. Later on this article will discuss the configurable shadow render budget which can lead to situations where we have to fallback to lower resolution data while higher resolution data is processed over multiple frames.

For shadow mapping, we can imagine that the "true" shadow map is something massive like 128K x 128K or more. As the player camera moves around, we will shift the clip origin to match the new camera location and update only the parts that changed.

![clipshift](/assets/v0.11/svsms/VSM_ClipOrigin.png)

The arrow in the above picture is a 2-component motion vector. If converted to NDC space, it can be used to project from current frame to previous frame or vice versa in order to determine which virtual pages have changed.

![clipupdate](/assets/v0.11/svsms/VSM_ClipUpdate.png)

The new data wraps around and is written to the region where the old, no longer needed data used to be.

Since this method focuses on using hardware rasterization, some amount of duplicate work will be done during the update. This means that there will be cases where parts of the green areas will be re-generated. Some possible approaches to reducing this will be discussed later.

# Physical Memory Format

The physical backing memory can be created as a 32-bit floating point texture. Depending on which operation is being done (read vs write), the texture may be converted to a compatible format that is more convenient for us to use. The main example in this article is during shadow map rendering where we bind it as a 32-bit unsigned integer image in order to gain access to image atomic min/max.

# Method Overview

![overview](/assets/v0.11/svsms/VSM_Overview.png)

This is a brief overview of all the steps required to render shadows for a single frame. The next sections will go over the steps in a lot more detail.

### Step 1: Analyze Depth Buffer
![step1](/assets/v0.11/svsms/VSM_Step1.png)

The process starts after the main camera's depth buffer has been generated. For each depth sample, first transform it to world space coordinates. The light's view-projection matrix is then used to convert from world space to NDC and then to virtual texture coordinates. Once the virtual texture coordinates have been computed they can be wrapped around to the nearest page table entry. This page table entry is what is marked as needed for this frame. The reason we need to wrap around is because the `lightViewProj` matrix in the code example below is centered on the origin **(0, 0, 0)** with only rotation applied. This means that it can generate virtual texture coordinates outside of the **[0, 1]** range, so we have to perform coordinate wrapping.

Here is some pseudocode:

{% highlight cpp %}
    mat4 lightViewProj = directional.ViewProjection();

    vec3 worldSpace = WorldSpaceFromDepth(depth);

    vec3 ndc = vec3(lightViewProj * vec4(worldSpace, 1.0));

    vec2 virtualUV = ndc * 0.5 + 0.5;

    vec2 pageTableEntry = WrapAround(virtualUV * pageTableSize);

    MarkPageNeeded(pageTableEntry);
{% endhighlight %}

### Step 2: Consult Page Tables
![step2](/assets/v0.11/svsms/VSM_Step2.png)

After **step 1** has marked all the needed page tables for this frame, we can check each page table entry to see if extra work needs to be done such as allocation, deallocation or marking for rendering. Each cascade will have its own page table since each cascade has access to the full virtual shadow map resolution. For example, if we decide we want to use 8K x 8K resolution with 6 cascades, we will have 6 8K x 8K virtual textures which are allocated from a shared physical memory pool.

If the virtual page is already backed by a valid piece of physical memory that is up to date, no further processing is needed.

If the virtual page does not have a valid page table entry or the data is out of date, this represents a page fault. At this point two things are needed. The first is to mark that page table entry as dirty so that future parts of the pipeline know to render shadows for this piece of the scene. The second is that if we cannot reuse existing physical memory, we need to write some information into the CPU readback buffer to let it know that it needs to allocate some new physical memory for the GPU to use.

### Step 3: Handle Page Faults, Clear Memory
![step3](/assets/v0.11/svsms/VSM_Step3.png)
If there are any memory allocation/deallocation requests, this is something that the CPU needs to help the GPU with. On the CPU side this pseudocode represents what it needs to do with the readback:

{% highlight cpp %}
    allocDeallocRequests = readback.MapMemory();
    for each request
        if request.xy < 0 then deallocate page
        else allocate page
{% endhighlight %}

With this setup, if the GPU wrote a negative page coordinate to the buffer it is asking for it to be deallocated. Positive page coordinates are allocation requests.

Once there is valid memory, control can be returned to the GPU so that it can clear all dirty pages since they are now backed by physical memory.

**Note:** It is possible to reduce the need for most readback situations if a full (or close to full) GPU-driven SVSM implementation is used. In this scenario the physical backing memory is a fixed size 2D texture pool pre-allocated during initialization and made available to shaders with a bindless approach. The GPU can then manage its own allocations.

### Step 4: Compute Screen Bounds
![step4](/assets/v0.11/svsms/VSM_Step4.png)

Each cascade will have its own ViewProjection matrix for rendering, and this means that it can be represented as a virtual screen. For each virtual pixel we can map it to a page table entry, check if that entry is dirty and if so, that portion of the virtual screen needs to be rendered. At the end we will have a set of screen tiles (each as large as a page) that need to be rendered. In the picture this region of screen tiles that need to be rendered is green while everything in red can be skipped.

You'll notice that some pages marked 0 are included in the page bounds. This represents some degree of wasted effort, but for simplicity the renderable screen region is made rectangular. Object culling can help with this situation since it can reduce or eliminate fragments being generated for screen tiles that are within the update region but don't need to be updated.

### Step 5: Cull Objects
![step5](/assets/v0.11/svsms/VSM_Culling.png)

Using the screen bounds computed in **step 4**, objects can be culled per cascade. To do this project their AABBs into NDC space using the render ViewProjection matrix for each cascade. This will give better-than-nothing results. Better results can be gained by using a hierarchical culling structure based on depth or based on information about residency/cache status from the page table.

### Step 6: Render To Physical Memory
![step6](/assets/v0.11/svsms/VSM_Step5.png)
*Helpful Resources:*
* [Matrix Setup For Tiled Rendering](https://stackoverflow.com/questions/28155749/opengl-matrix-setup-for-tiled-rendering)

During rendering, even though the rasterizer hardware is being used we are going to have to disable the normal render target so that we can manually write the depth to the texture. This has the unfortunate side effect of disabling the possibility of early z fragment discard. Instead we need to bind the physical backing texture (created as 32-bit float) as a 32-bit unsigned integer image which should be one of the compatible image format conversions. This will allow us to convert float depth values to unsigned int using `floatBitsToUint` and then write to the texture with `imageAtomicMin` (GLSL) in the fragment/pixel shader.

Next we can augment each cascade's ViewProjection matrix to only encompass the screen bounds computed in **step 4**. This will enable us to render the part that changed and skip what didn't much more effectively which matches up with the description of a clip map given above. To do this the matrix can be split into tiles and fractionally translated so that we can set the viewport size to be exactly the **step 4** bounds.

If we have a page table that is 64x64, this means we also have 64x64 screen tiles. NDC space goes from **[-1, 1]** meaning it is 2 units wide, high and deep. This means each tile is 2/64 = 1/32 units in size. What we can do is sum up the number of tiles we want to render in the X and Y directions and merge them into larger groups. Here is some pseudocode that will do that:

**Note:** Depending on how good your object culling is, this step may end up being unnecessary. What it does is help reduce the size of viewport that is being rendered to in order to offset any false positives that the culling algorithm produces. The less false positives there are, the less there is a need for viewport bounds changes.

{% highlight cpp %}
    // Index of the min and max screen tiles we want to render
    ivec2 minScreenTilesXY = minimum from step 4;
    ivec2 maxScreenTilesXY = maximum from step 4;

    ivec2 screenTileRange = maxScreenTilesXY - minScreenTilesXY;
    
    // Merge them into larger screen tile groups
    vec2 newNumScreenTiles = vec2(totalNumScreenTilesXY) / vec2(screenTileRange);

    // Convert minimum screen tiles to [0, 1] range
    vec2 normalizedScreenTiles = vec2(minScreenTilesXY) / vec2(totalNumScreenTilesXY);

    // Translate the normalized screen tiles into fractional locations within
    // the new merged screen tiles
    vec2 newMinScreenTilesXY = normalizedScreenTiles * newNumScreenTiles;
{% endhighlight %}

Once we've done this, the last part is to augment our ViewProjection matrix with a scale and translate identical to the way [this link](https://stackoverflow.com/questions/28155749/opengl-matrix-setup-for-tiled-rendering) describes.

{% highlight cpp %}
    vec2 inverseXY = 1.0 / newNumScreenTiles;

    float tx = - (-1.0f + inverseXY.x + 2.0f * inverseXY.x * newMinScreenTilesXY.x);
    float ty = - (-1.0f + inverseXY.y + 2.0f * inverseXY.y * newMinScreenTilesXY.y);

    mat4 scale(1.0f);
    MatScale(scale, vec3(newNumScreenTiles, 1.0f));

    mat4 translate(1.0f);
    MatTranslate(translate, vec3(tx, ty, 0.0f));

    mat4 newProjectionViewRender = scale * translate * cascade[i].ProjectionViewRender();
{% endhighlight %}

Now for rendering this cascade, `newProjectionViewRender` is used. This will save on a lot of performance because we can set the viewport bounds to be only the area of the virtual screen that changed. But now we run into another problem. Our new projection view matrix will generate fragments with depth values that we want to store into the shadow map, but how to we translate them to physical memory locations?

# Mapping Virtual Memory to Physical Memory

### Page Table Structure

Each entry in the page table manages the state for a single `128x128` texel page. So, if the virtual resolution is set to `8182x8192`, the page table will need to be `64x64` elements in size.

Here is a possible implementation of a page table entry (32 bits per entry):

![pageentry](/assets/v0.11/svsms/VSM_PageEntry.png)

**Frame Marker:** Used for marking the page table entry as in use. 0 means unmarked, 1 means marked this frame, and anything greater than 1 means it was marked a few frames ago but is not currently needed for this frame. This is useful if you want to add a slight frame delay before evicting something from the cache.

**Physical Page Index X/Y:** Points to the lower-left corner of the physical page in memory. Reconstructing the lower-left physical texel from the page index is done using `Physical Page Index X/Y * ivec2(128)` where 128 represents texels per page in the x/y direction.

**Memory Pool Index:** This technique works by allocating physical memory from a series of either sparse or shared texture memory pools. In the case of sparse textures, the API allows for making pages resident or non-resident and doing sparse texture binding. In the case of shared texture memory, fixed-size textures are allocated from and returned to the shared texture pool as needed. The memory pool index can either refer to the array index (when using sparse texture arrays) or the texture index (when using separate textures pulled from a shared pool).

**Residency Status:** Tells the allocator whether a page is backed by physical memory or not.

**Valid/Dirty Status:** This tells the renderer whether a page needs to be re-generated or not. When the clip origin shifts, pages that fall in the update region can be marked as dirty.

### ClipMap Matrix Structure

At this stage we need to define a clear way of converting from world coordinates to virtual page coordinates and then to physical texel coordinates. To do this we are going to define two different groups of matrices: projection-view sample and projection-view render. Each frame the projection-view render matrix will potentially change via translation and represents the clip origin + extent, but the projection-view sample matrix will either never change or very rarely change (depending on your use case). The sample matrix is used to get virtual uv coords that we can use to access the page table and so its origin will be set to **(0, 0, 0)**. The render matrix is used when rendering the shadow map using normal hardware rasterization and its origin changes as the camera moves through the scene.

Once we have these two groups of matrices, the steps of moving from virtual to physical are as follows:

1) Translate local clip map uv coordinates (projection-view render) to virtual clip map uv coordinates (projection-view sample)

2) Perform coordinate wrap around for any values outside of **[0, 1]** range

3) Use wrapped coordinates to access the page table and pull-out physical X/Y offset and memory pool index

4) Use physical X/Y offset and memory pool index to construct physical uv coordinates

5) Access physical texture with physical uv coordinates for either sampling or writing

#### Projection View Sample (stable virtual addressing)

We are going to define the projection view sample matrix as being positioned at **(0, 0, 0)** and set up the light orthographic projection matrix as follows:

$$
    \begin{bmatrix} 
	2 / d_k & 0 & 0 & 0 \\
	0 & 2 / d_k & 0 & 0 \\
	0 & 0 & 1 / z & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
$$


Where $$d_k$$ is the maximum view-space extent of the first clipmap cascade and $$z$$ is the maximum depth. Each cascade after the first maintains the same maximum depth but doubles the extent ($$d_k$$). Because of this, only the first matrix needs to be passed into the shader and the rest can be derived.

The global clip origin **(0, 0, 0)** view-projection matrix is created by multiplying the orthographic projection above with the rotation-only directional light view matrix.

The view matrix looks like this:

$$
	\begin{bmatrix} 
	a & b & c & 0 \\
	d & e & f & 0 \\
	g & h & i & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
$$

where the upper 3x3 matrix contains only rotation.

#### Projection View Render

All local clip origin matrices share the same orthographic matrix from above. The only difference is that they add a translation component to their local directional light view matrix. As the player moves around the scene, we want to change the clip origin so that we can render the shadows in concentric rings around the player.

The view matrix looks like this:

$$
	\begin{bmatrix} 
	a & b & c & t_x \\
	d & e & f & t_y \\
	g & h & i & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
$$

The upper 3x3 matrix is identical to the one used for projection-view sample. Multiplying it with the projection matrix from above we get this:

$$
    \begin{align*}
    \begin{bmatrix} 
	2 / d_k & 0 & 0 & 0 \\
	0 & 2 / d_k & 0 & 0 \\
	0 & 0 & 1 / z & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
    \times
    \begin{bmatrix} 
	a & b & c & t_x \\
	d & e & f & t_y \\
	g & h & i & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
    =
	\begin{bmatrix} 
	2a/d_k & 2b/d_k & 2c/d_k & 2t_x/d_k \\
	2d/d_k & 2e/d_k & 2f/d_k & 2t_y/d_k \\
	g/z & h/z & i/z & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
    \end{align*}
$$

Now let's see what happens when we multiply this matrix by a world space position 

$$
    \begin{align*}
	\begin{bmatrix} 
	2a/d_k & 2b/d_k & 2c/d_k & 2t_x/d_k \\
	2d/d_k & 2e/d_k & 2f/d_k & 2t_y/d_k \\
	g/z & h/z & i/z & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
    \times
    \begin{bmatrix} 
    v_x \\
    v_y \\
    v_z \\
    1
	\end{bmatrix}
    =
    \begin{bmatrix} 
    2av_x/d_k + 2bv_y/d_k + 2cv_z/d_k + 2t_x/d_k\\
    2dv_x/d_k + 2ev_y/d_k + 2fv_z/d_k + 2t_y/d_k \\
    gv_x/z + hv_y/z + iv_z / z \\
    1
	\end{bmatrix}
    \end{align*}
$$

This gives us NDC coordinates for the first clip map cascade. What if we want to convert the result back to what it would have been if we had used projection-view sample? We can subtract off $$2t_x/d_k, 2t_y/dk$$ from the result.

What about for the other cascades? We know that for each successive clip map cascade, $$d_k$$ doubles in size but everything else remains the same including the translation component. So, to convert from cascade 0 to cascade n, we can multiply the result by $$1/2^c$$ where $$c$$ is the cascade index from **[0, cascades - 1]**.

Because of this we only need to pass in the matrix for the very first clip map cascade and we can use it in the shader to convert to any other cascade, or back to the projection-view sample space. Here is some GLSL code that does this:

{% highlight glsl %}
    #define BITMASK_POW2(shift) (1 << shift)

    // For first clip map - rest are derived from this
    uniform mat4 vsmClipMap0ProjectionViewRender;
    uniform uint vsmNumCascades;
    uniform uint vsmNumPagesXY;
    uniform uint vsmTexelResolutionXY;

    layout (r32ui) coherent uniform uimage2DArray pageTable;

    vec3 VsmConvertClip0ToClipN(in vec3 original, in int clipMapIndex) {    
        vec3 result = original;                                          
        result.xy *= vec2(1.0 / float(BITMASK_POW2(clipMapIndex)));         
        return result;                                                   
    }

    // Returns values on the range of [-1, 1]
    vec3 VsmCalculateRenderClipValueFromWorldPos(in vec3 worldPos, in int clipMapIndex) {
        vec3 result = (vsmClipMap0ProjectionViewRender * vec4(worldPos, 1.0)).xyz;

        // Accounts for the fact that each clip map covers double the range of the
        // previous
        return VsmConvertClip0ToClipN(result, clipMapIndex);
    }

    // Subtracts off the scaled translate component
    vec3 VmCalculateSampleClipValueFromWorldPos(in vec3 worldPos, in int clipMapIndex) {
        vec3 result = VsmCalculateRenderClipValueFromWorldPos(worldPos, clipMapIndex);

        // Accounts for the fact that each clip map covers double the range of the
        // previous
        return result - VsmConvertClip0ToClipN(vsmClipMap0ProjectionView[3].xyz, clipMapIndex);
    }
{% endhighlight %}

### Performing Coordinate Translation

Now we have everything we need to go from world space to virtual coordinates to physical coordinates. For a given world space coordinate, we will calculate the sample clip value and convert that to virtual texture coordinates.

{% highlight glsl %}
    vec3 ConvertWorldPosToPhysicalCoordinates(in vec3 worldPos, in int clipMapIndex) {
        vec3 ndc = VmCalculateSampleClipValueFromWorldPos(worldPos, clipMapIndex);

        vec2 virtualTexCoords = ndc.xy * 0.5 + 0.5;

        ...
{% endhighlight %}

Next we need to perform coordinate wrap around. To do this we can use the fract function which computes `x - floor(x)`.

{% highlight glsl %}
        virtualTexCoords = fract(virtualTexCoords);
{% endhighlight %}

Next we can use these wrapped virtual coordinates to get the page table entry. Once we have that we can use bitmask operations to pull out the physical page X/Y offset and memory pool index.

{% highlight glsl %}
        ivec2 pageTableIndex = ivec2(floor(virtualTexCoords * vec2(vsmNumPagesXY)));
        uint entry = imageLoad(pageTable, ivec3(pageTableIndex, clipMapIndex)).r;

        ivec2 physicalPageXYOffset;
        int memoryPoolIndex;
        UnpackPhysicalMemoryData(entry, physicalPageXYOffset, memoryPoolIndex);
{% endhighlight %}

Once we have these values, what we need to do is figure out which texel the virtualTexCoords are trying to point to. What we know about the physical pages is that they are `128x128` texels in size. So, to figure out which texel we need to point to, we can calculate the virtual pixel coordinate and mod it by 128.

{% highlight glsl %}
        vec2 virtualPixelCoords = virtualTexCoords * vec2(vsmTexelResolutionXY);
        vec2 physicalPageTexelOffsets = mod(virtualPixelCoords, 128.0);
{% endhighlight %}

The final step is to convert these coordinates to actual physical coordinates that we can use to sample the physical texture. Since `physicalPageXYOffset` points to the lower-left corner of the physical page (in indices, not texels) and `physicalPageTexelOffsets` points to the texel within that page, we can multiply `physicalPageXYOffset` by the number of texels per page (128) and add it to `physicalPageTexelOffsets`.

{% highlight glsl %}
        return vec3(
            vec2(physicalPageXYOffset) * vec2(128) + physicalPageTexelOffsets,
            float(memoryPoolIndex)
        );
{% endhighlight %}

![mapping](/assets/v0.11/svsms/VirtualMapping.png)
(visual of above mapping code)

And we're done! We now have a way to convert from world space coordinates -> virtual coordinates -> physical memory coordinates. If we revisit the rendering **step 6** from above, what this means is that we will use `newProjectionViewRender` for each cascade to rasterize the scene. For each fragment that we generate we want to convert it to physical coordinates and write the depth to that location.

In practice this means something like `gl_Position` will be written in the vertex shader using `newProjectionViewRender`, but it will smooth out a set of texture coordinates that it generates using `ConvertWorldPosToPhysicalCoordinates` with the world position. This way the fragment shader uses the smoothed in texture coordinates to write out `gl_FragCoord.z` to the shadow map.

# Selecting Cascades

Given a world position, how can we quickly determine which cascade it will fall in? We can look at the NDC output from multiplying the first cascade's projection view render with a world position.

$$
    \begin{align*}
    \begin{bmatrix} 
    2av_x/d_k + 2bv_y/d_k + 2cv_z/d_k + 2t_x/d_k \\
    2dv_x/d_k + 2ev_y/d_k + 2fv_z/d_k + 2t_y/d_k \\
    gv_x/z + hv_y/z + iv_z / z \\
    1
	\end{bmatrix}
    =
    \begin{bmatrix} 
    n_x \\
    n_y \\
    n_z \\
    1
	\end{bmatrix}
    \end{align*}
$$

Looking at the X and Y component, we know that if the world position fell within the first cascade, the NDC will be on the range of **[-1, 1]**. If the result is outside of this range, it means that we need to multiply $$n_x, n_y$$ by $$1/2^c$$. If we can solve for positive integer values of c, we get the cascade index.

$$
    \begin{align*}
    \begin{bmatrix} 
    0 \\
    0 \\
	\end{bmatrix} \leq
    \begin{bmatrix} 
    n_x \times 1/2^c \\
    n_y \times 1/2^c \\
	\end{bmatrix}
    \leq     
    \begin{bmatrix} 
    1 \\
    1 \\
	\end{bmatrix}
    \end{align*}
$$

We can constrain this to positive values by taking the absolute value of $$n_x, n_y$$.

$$
    \begin{align*}
    \begin{bmatrix} 
    0 \\
    0 \\
	\end{bmatrix} \leq
    \begin{bmatrix} 
    | n_x | \times 1/2^c \\
    | n_y | \times 1/2^c \\
	\end{bmatrix}
    \leq     
    \begin{bmatrix} 
    1 \\
    1 \\
	\end{bmatrix}
    \end{align*}
$$

Focusing on solving for just one of them since both will be the same format:

$$
    \begin{aligned}
    0 &\leq | n_x | / 2^c \leq 1 \\
    => 0 &\leq | n_x | \leq 2^c \\
    => 0 &\leq log_2(| n_x |) \leq log_2(2^c) \\
    => 0 &\leq log_2(| n_x |) \leq clog_2(2) \\
    => 0 &\leq log_2(| n_x |) \leq c \\
    \end{aligned}
$$

Since this is undefined at $$n_x=0$$ and $$log_2(1)=0$$ (first cascade index), we can constrain $$n_x$$ using max.

$$
    \begin{aligned}
    0 &\leq log_2(max(| n_x |, 1)) \leq c \\
    \end{aligned}
$$

This can still give us fractional results but we only care about positive integer solutions. So we can use ceil on the result.

$$
    \begin{aligned}
    \lceil log_2(max(| n_x |, 1)) \rceil = c  \\
    \end{aligned}
$$

This is the solution for $$n_x$$, but $$n_y$$ will also give us a value. When there is a situation where the two results are different, the largest will be the correct answer. Here is some GLSL code:

{% highlight glsl %}
int VsmCalculateCascadeIndexFromWorldPos(in vec3 worldPos) {
    // Get the normalized device coordinates (NDC) for the first cascade
    vec2 ndc = VsmCalculateRenderClipValueFromWorldPos(worldPos, 0).xy;

    vec2 cascadeIndex = vec2(
        ceil(log2(max(abs(ndc.x), 1))),
        ceil(log2(max(abs(ndc.y), 1)))
    );

    return int(max(cascadeIndex.x, cascadeIndex.y));
}
{% endhighlight %}

Here is a visual of cascades changing as the camera moves closer or further from geometry:

<img src="/assets/v0.11/svsms/VSM_Cascade_Change.gif" alt="cascade_change" />

When pages become visually smaller and change in color tone this is an indication they are part of finer cascades. Visually larger pages are an indication they are part of coarser cascades.

# Cache Invalidation Events

There are a few situations where one or more cache entries become invalid. The first is the most common which is when a page is no longer needed for the current frame. In this case it can be marked invalid and its physical memory freed up for other pages.

### Light Rotation

Another common situation is when the directional light rotates. This is a case when projection-view sample changes, and because of this the entire virtual address mapping changes.

This situation results in all the pages becoming invalid at the same time. If the light is rotating very slowly it may be possible to reuse old data while staggering the updates to improve performance, but this is an open issue I don't have a good answer to right now. One approach mentioned in [this Fortnite article](https://www.unrealengine.com/en-US/tech-blog/virtual-shadow-maps-in-fortnite-battle-royale-chapter-4) is that if your scene will have frequent directional light changes, fine tune shadow rendering to prioritize page rendering performance. One way would be reducing the virtual resolution: drop down to 8K or 4K. Another is to adopt a meshlet-based rendering pipeline to reduce the number of pages each object overlaps (improves culling potential). Another is to avoid doing a continuous rotation and instead split it into several incremental rotations so that the shadow updates can be spread out over multiple frames.

Here is a visual of the location of the pages changing because of directional light rotation:

<img src="/assets/v0.11/svsms/VSM_Invalidate.gif" alt="invalidate" />

### Dynamic Objects

Another event resulting in cache invalidations is when objects in the scene move around. There are at least two approaches to dealing with this. The first is to render static and dynamic objects into the same clipmaps. When an object moves its AABB could be used to check which pages it overlaps with and then mark those as invalid. 

The second is to maintain two sets of clipmap cascades: one for static objects and one for dynamic objects. Dynamic object clipmaps can be tailored towards being updated almost entirely every frame whereas static objects can use the incremental update approach as the camera moves.

# Allocation Strategy

An easy way to handle allocation for this virtual memory system is to pack a list of free pages into a shader storage buffer object (SSBO). When the GPU wants to allocate, it can remove the topmost entry using shader atomics and add that page to the corresponding page table entry. When it wants to deallocate, it can add the page back to the front of the list again using shader atomics.

In the following gif the lower left shows the first memory pool as it pulls pages from the free list or adds them back. Black = unallocated.

Each implementation will need to decide how to deal with the case of a memory pool running out of memory during a given frame. It's possible that more pages will be requested than a single memory pool can handle. One approach would be to trigger a page fault like normal, but the CPU could then allocate a whole new block to pull memory from. Another approach would be to evict pages corresponding to coarser cascades and prioritize putting them in the cascades closer to the camera.

The decision for when to evict a page from the cache is also configurable. The page table has enough bits to count to 15 frames as a delay for evicting a page from the cache, but by default it marks pages free as soon as they are no longer required for a given frame (no delay). One reason to keep them around a longer even when not directly visible is if they are requested by a postprocessing effect. In that situation it is a good idea to only keep the lowest resolution version of the data that the post processing effect needs while freeing other levels.

### Hardware/API Sparse Memory

OpenGL, Vulkan and DirectX all have dedicated API support for hardware sparse memory allocation.

* [Sparse Textures OpenGL](https://gpuopen.com/learn/sparse-textures-opengl/)
* [Sparse Resources Vulkan](https://registry.khronos.org/vulkan/site/spec/latest/chapters/sparsemem.html)
* [Tiled Resources DirectX](https://learn.microsoft.com/en-us/windows/win32/direct3d11/tiled-resources)

The main issue is that even though these are supported, there have been reports of major issues with performance especially on Windows. For example:

* [vkQueueBindSparse is insanely slow](https://www.reddit.com/r/vulkan/comments/bw3wib/vkqueuebindsparse_is_insanely_slow/)
* [Sparse texture binding is painfully slow](https://forums.developer.nvidia.com/t/sparse-texture-binding-is-painfully-slow/259105)

Your mileage may vary and will require profiling for the target hardware and OS/drivers.

### Software Sparse Memory

In the absence of good support for hardware/API sparse memory, the main alternative is to fallback to regular textures and texture arrays. Each texture will have a fixed power of 2 size and be capable of representing some number of physical `128x128` texel pages. These textures can be allocated from and returned to a shared texture memory pool as needed.

### Allocation Strategy Comparison

<img src="/assets/v0.11/svsms/VSM_AllocationStrategy.gif" alt="allocator" />

Hardware sparse leverages driver/API sparse memory management+binding support. Software sparse manages memory manually and allocates from fixed-size texture pools.

# Shadow Render Budget

One possible performance optimization is to add a configurable render budget for shadow map generation. To conceptualize this, we can return to the clip map page from earlier.

![mips](/assets/v0.11/svsms/VSM_Mips.png)

From this image we can see that each cascade, even though not strictly part of a mip chain, can still be used as one since each successive cascade is fully capable of representing everything from the previous cascade, just at a lower effective resolution.

This allows us to specify one or more cascades to serve as the lowest/coarsest levels in the mip chain. When we perform the depth buffer analysis, not only will we mark the pages in the finest detail mip levels, but we will also mark the corresponding pages in the lowest mip levels.

Then what we can do when rendering is to set a budget for how many pages from the finest mips to process before saving them for the next frame. The lowest mips will be rendered fully every frame so we can use them as a fallback when data is not available at a higher resolution.

This gif shows what this looks like (slowed down to exaggerate the shadow pop in):

<img src="/assets/v0.11/svsms/VSM_Popin.gif" alt="popin" />

As the camera moves through the wall, you'll notice that part of the scene must temporarily fall back to lower (coarser) mip levels until all of the higher (finer) mip levels are processed.

This is a nice option to have since it provides a setting that can be tweaked for different hardware and different scene complexities. It could also be set dynamically to help when the rendering is struggling to maintain target frame times.

# Hardware Shadow Filtering

An issue comes up when we try to use hardware shadow filtering such as with `sampler2DShadow`. For most of the texels within a page the hardware can freely perform its filtering without issue, but the problem is at the page border:

![filter](/assets/v0.11/svsms/VSM_Filter.png)

Everything in the green represents texels that won't cause an issue with hardware shadow filtering. Everything in the red is where the problems start since the hardware will assume it can sample into the next adjacent pages. But the next pages will almost always contain data from completely different parts of the scene.

### Option 1: Border

For this option you can contract or expand the size of the shadow map so that you can add a 1 texel border around each page. Then it is possible to use hardware filtering without any issues or having to handle special cases.

### Option 2: Hybrid

Another option is to write a sampling function that checks to see if the starting texel is on the page boundary. If it is, you can perform software shadow filtering so that you can avoid sampling into memory that has nothing to do with the current page.

If not on a border, use hardware filtering as normal.

### Option 3: Manual

The last option is to abandon hardware shadow filtering completely and write your own software filtering code. This has the advantage of not requiring branching in the shader or adding any page borders.

# Post-Processing Effects

There will be times when certain post processing effects such as volumetric lighting might require data to be present in the shadow map that is outside of the current view of the camera. When this is the case, it will be necessary to add some extra logic during the page marking/depth prepass stage. The clipmap cascade that is marked will depend on what level of resolution the post processing effect will need. If it can get away with low resolution, then it can go with the further/coarser cascades to save on bandwidth and performance.

Thanks for reading!

# References
* [Virtual Shadow Maps in Unreal Engine](https://docs.unrealengine.com/5.1/en-US/virtual-shadow-maps-in-unreal-engine/)
* [Virtual Shadow Maps in Fortnite Battle Royale Chapter 4](https://www.unrealengine.com/en-US/tech-blog/virtual-shadow-maps-in-fortnite-battle-royale-chapter-4)
* [The Clipmap: A Virtual Mipmap](https://dl.acm.org/doi/pdf/10.1145/280814.280855)
* [Sparse Virtual Textures](https://www.youtube.com/watch?v=MejJL87yNgI)
* [Another Sparse Virtual Textures](https://studiopixl.com/2022-04-27/sparse-virtual-textures)
* [Understanding Virtual Memory](https://performanceengineeringin.wordpress.com/2019/11/04/understanding-virtual-memory/)
* [Terrain Rendering Using GPU-Based Geometry Clipmaps](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-2-terrain-rendering-using-gpu-based-geometry)
* [Fitted Virtual Shadow Maps](https://www.cg.tuwien.ac.at/research/publications/2007/GIEGL-2007-FVS/GIEGL-2007-FVS-Preprint.pdf)
* [Queried Virtual Shadow Maps](https://www.cg.tuwien.ac.at/research/publications/2007/GIEGL-2007-QV1/GIEGL-2007-QV1-Preprint.pdf)