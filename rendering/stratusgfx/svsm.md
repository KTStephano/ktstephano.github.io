---
permalink: /rendering/stratusgfx/svsm
title: Sparse Virtual Shadow Maps
layout: page
use_math: true
---

![bistro1](/assets/v0.11/svsms/VSM1_Bistrov2.png)
![bistro2](/assets/v0.11/svsms/VSM2_Bistrov2.png)

The top is the Bistro scene rendered with multiple 8K resolution sparse virtual shadow maps. The bottom is a visualization of the physical memory pages (squares) where each color represents a different shadow clipmap/cascade.

**This is a WIP/rough draft!**
**Last edited: Sept 13, 2023.**

# Collaborators

This was developed in loose collaboration with:

* [Jaker v3](https://juandiegomontoya.github.io)
* [LVSTRI](https://github.com/LVSTRI)

TODO: How do they prefer to be linked to??

# Table of Contents
* TOC
{:toc}

# Introduction

This post goes through the high level steps needed to create a sparse virtual memory system for realtime shadows. This was inspired by Unreal Engine 5's virtual shadow maps. Directional lights are the only ones considered here, but it should be possible to extent the system to support other light types such as point or spotlights. The directional shadows make use of clipmaps for incremental updating of the shadowmaps and handling multiple different cascades.

### Sparsity

Part of the strength of this method comes from its ability to represent shadows using sparse textures. If nothing is currently present in one of the clipmap cascades, that cascade's memory footprint and processing overhead can be reduced to almost nothing.

### Virtual

This system replaces normal shadow map uv coordinates with virtual uv coordinates. To either write to or read from the shadow maps, virtual uv coordinates have to be translated to physical uv coordinates. The actual physical memory pool can be much smaller than what the virtual uv coordinate space would suggest. Conceptually this is extremely similar to virtual memory systems in modern operating systems.

### Caching

As the camera moves around the scene, new virtual pages become visible and are rendered incrementally. Previously-rendered pages that are still visible and haven't changed can be reused from frame to frame to save on performance.

# Foundations

This technique is built off of two core foundations: sparse virtual texturing and clipmaps.

### Sparse Virtual Texturing

*Helpful resources:*
* [Sparse Virtual Textures](https://www.youtube.com/watch?v=MejJL87yNgI)
* [Understanding Virtual Memory](https://performanceengineeringin.wordpress.com/2019/11/04/understanding-virtual-memory/)

With regular virtual memory, all addresses used by a usermode application are virtual addresses. The OS then translates these virtual addresses into physical addresses when the memory needs to be used in some way. Using this system, a piece of the virtual address is used as an offset into a translation (page) table, and that entry in the page table is used to determine where in physical memory to go. Each entry in the page table represents a physical page of memory with a fixed size (ex: 4 kb). It's possible that the physical memory that the virtual address is trying to access is not currently available for some reason, one of which being that it was moved from RAM to disk. This is a page fault and requires that the OS deal with it so that the process can continue normally.

Sparse virtual texturing uses the same approach. For this to work, each physical page of memory is a 2D block of size 128x128 texels. For an 8K x 8K texture, this would result in 64x64 page table entries. This 64x64 grid is our version of the page table.

Like with other virtual memory systems, sparse virtual texturing only maintains the minimum amount of texture data in memory for the application to continue normally. It does this by analyzing the current camera view of the scene to determine what is needed verses what can be left unallocated.

At runtime, shaders will use virtual coordinates rather than physical coordinates. These virtual coordinates are translated to physical coordinates using the page table. When the data that the virtual coordinates reference is not currently resident in physical memory, this results in a page fault. Some mechanism (such as readback SSBO) needs to be used to tell the CPU to perform an allocation and prepare the physical memory for use.

### Clipmaps

*Helpful resources:*
* [The Clipmap: A Virtual Mipmap](https://dl.acm.org/doi/pdf/10.1145/280814.280855)

Directional light shadow maps are represented by a series of expanding rings around the player camera known as clipmap rings/cascades. Conceptually they are similar to cascades in cascaded shadow maps.

A clipmap is an incrementally updatable, fixed-size texture region that is meant to represent a subset of a full texture's mipmap chain. A clipmap is defined by a clip origin and a clip size. Together these two determine which part of the full texture that the clipmap currently represents. Each clipmap ring is required to cover double the space of the previous clipmap ring, making each successive ring coarser than the previous since the memory footprint remains the same. Because coarser clipmap rings fully overlap finer clipmap rings, we can imagine them as being a version of texture mipmapping.

![clipmap](/assets/v0.11/svsms/VSM_Clipmap.png)

When sampling from a clipmap, we select the most detailed (lowest ring/cascade) that's available. If not available, we clip to the next best fit.

For shadow mapping, we can imagine that the "true" shadow map is something massive like 128K x 128K or more. As the player camera moves around, we will shift the clip origin to match the new camera location and update only the parts that changed.

![clipshift](/assets/v0.11/svsms/VSM_ClipOrigin.png)

![clipupdate](/assets/v0.11/svsms/VSM_ClipUpdate.png)

The new data wraps around and is written to the region where the old, no longer needed data used to be.

Since this method focuses on using hardware rasterization, some amount of duplicate work will be done during the update. This means that there will be cases where parts of the green areas will be re-generated. Some possible approaches to reducing this will be discussed later.

# Method Overview

![overview](/assets/v0.11/svsms/VSM_Overview.png)

This is a brief overview of all of the steps required to render shadows for a single frame. The next sections will go over the steps in a lot more detail.

### Step 1: Analyze Depth Buffer
![step1](/assets/v0.11/svsms/VSM_Step1.png)

The process starts after the main camera's depth buffer has been generated. For each depth sample, first transform it to world space coordinates. From world space the light view-projection matrix is used to convert to NDC and then to virtual texture coordinates. Once the virtual texture coordinates have been computed they can be wrapped around to the nearest page table entry. This page table entry is what is marked as needed for this frame.

Here is some pseudocode:

{% highlight cpp %}
    mat4 lightViewProj = directional.ViewProjection();

    vec3 worldSpace = WorldSpaceFromDepth(depth);

    vec3 ndc = vec3(lightViewProj * vec4(worldSpace, 1.0));

    vec2 virtualUV = ndc * 0.5 + 0.5;

    vec2 pageTableEntry = WrapAround(virtualUV * pageTableSize);

    MarkPageNeeded(pageTableEntry);
{% endhighlight %}

Why do we need to wrap around? The `lightViewProj` matrix is centered on the origin (0, 0, 0) with only rotation applied. This means that it can generate virtual texture coordinates outside of the [0, 1] range, so we have to perform coordinate wrapping.

### Step 2: Consult Page Tables
![step2](/assets/v0.11/svsms/VSM_Step2.png)

After step 1 has marked all the needed page tables for this frame, we can check each page table entry to see if extra work needs to be done such as allocation, deallocation or marking for rendering.

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

### Step 4: Compute Screen Bounds
![step4](/assets/v0.11/svsms/VSM_Step4.png)

Each cascade will have its own ViewProjection matrix for rendering, and this means that it can be represented as a virtual screen. For each virtual pixel we can map it to a page table entry, check if that entry is dirty and if so, that portion of the virtual screen needs to be rendered. At the end we will have a set of screen tiles (each as large as a page) that need to be rendered. In the picture this region of screen tiles that need to be rendered is green while everything in red can be skipped.

You'll notice that some pages marked 0 are included in the page bounds. This represents some degree of wasted effort, but for simplicity the renderable screen region is made rectangular.

### Step 5: Cull Objects
![step5](/assets/v0.11/svsms/VSM_Culling.png)

Using the screen bounds computed in step 4, objects can be culled per cascade. To do this project their AABBs into NDC space using the render ViewProjection matrix for each cascade. This will give better-than-nothing results, but to get much better results something more advanced such as Hierarchical Depth Buffer (HZB) culling can be used.

### Step 6: Render To Physical Memory
![step6](/assets/v0.11/svsms/VSM_Step5.png)
*Helpful Resources:*
* [Matrix Setup For Tiled Rendering](https://stackoverflow.com/questions/28155749/opengl-matrix-setup-for-tiled-rendering)

To do this we first have to augment each cascade's ViewProjection matrix to only encompass the screen bounds computed in step 4. This will enable us to render the part that changed and skip what didn't much more effectively which matches up with the description of a clip map given above. To do this the matrix can be split into tiles and fractionally translated so that we can set the viewport size to be exactly the step 4 bounds.

If we have a page table that is 64x64, this means we also have 64x64 screen tiles. NDC space goes from [-1, 1] meaning it is 2 units wide, high and deep. This means each tile is 2/64 = 1/32 units in size. What we can do is sum up the number of tiles we want to render in the X and Y directions and merge them into larger groups. Here is some pseudocode that will do that:

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

Each entry in the page table manages the state for a single `128x128` texel page. So if the virtual resolution is set to `8182x8192`, the page table will need to be `64x64` elements in size.

Here is a possible implementation of a page table entry (32 bits per entry):

![pageentry](/assets/v0.11/svsms/VSM_PageEntry.png)

**Frame Marker:** Used for marking the page table entry as in use. 0 means unmarked, 1 means marked this frame, and anything greater than 1 means it was marked a few frames ago but is not currently needed for this frame. This is useful if you want to add a slight frame delay before evicting something from the cache.

**Physical Page Offset X/Y:** Points to the lower-left corner of the physical page in memory. Reconstructing the pixel index is done using `Physical Page Offset X/Y * ivec2(128)` where 128 represents texels per page in the x/y direction.

**Memory Pool Index:** This technique works by allocating physical memory from a series of sparse memory pools. When one memory pool has no remaining free memory for this frame, the allocator moves to the next pool. Under the hood the physical backing memory is a sparse 2D texture array and the memory pool index refers to which array slice the physical page is in.

**Residency Status:** Tells the allocator whether a page is backed by physical memory or not.

**Valid/Dirty Status:** This tells the renderer whether a page needs to be re-generated or not. When the clip origin shifts, pages that fall in the update region can be marked as dirty.

### ClipMap Matrix Structure

At this stage we need to define a clear way of converting from world coordinates to virtual page coordinates and then to physical texel coordinates. To do this we are going to define two different groups of matrices: projection-view sample and projection-view render. Each frame the projection-view render matrix will potentially change via translation and represents the clip origin + extent, but the projection-view sample matrix will either never change or very rarely change (depending on your use case). The sample matrix is used to get virtual uv coords that we can use to access the page table. The render matrix is used to generate fragments normally when rendering the shadow map with the hardware rasterization pipeline.

Once we have these two groups of matrices, the steps of moving from virtual to physical are as follows:

1) Translate local clip map uv coordinates (projection-view render) to virtual clip map uv coordinates (projection-view sample)

2) Perform coordinate wrap around for any values outside of [0, 1] range

3) Use wrapped coordinates to access the page table and pull out physical X/Y offset and memory pool index

4) Use physical X/Y offset and memory pool index to construct physical uv coordinates

5) Access physical texture with physical uv coordinates for either sampling or writing

#### Projection View Sample (address translation)

We are going to define the projection view sample matrix as being positioned at (0, 0, 0) and set up the light orthographic projection matrix as follows:

$$
	\begin{bmatrix} 
	2 / d_k & 0 & 0 & 0 \\
	0 & 2 / d_k & 0 & 0 \\
	0 & 0 & 1 / z & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
$$


Where $d_k$ is the maximum view-space extent of the first clipmap cascade and $z$ is the maximum depth. Each cascade after the first maintains the same maximum depth but doubles the extent ($d_k$). Because of this, only the first matrix needs to be passed into the shader and the rest can be derived.

The global clip origin (0, 0, 0) view-projection matrix is created by multiplying the orthographic projection above with the rotation-only directional light view matrix.

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

All local clip origin matrices share the same orthographic matrix from above. The only difference is that they add a translation component to their local directional light view matrix. As the player moves around the scene, we want to change the clip origin so that we can render the shadows in a concentric rings around the player.

The view matrix looks like this:

$$
	\begin{bmatrix} 
	a & b & c & t_x \\
	d & e & f & t_y \\
	g & h & i & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
$$

Multiplying it with the projection matrix from above we get this:

$$
	\begin{bmatrix} 
	2a/d_k & 2b/d_k & 2c/d_k & 2t_x/d_k \\
	2d/d_k & 2e/d_k & 2f/d_k & 2t_y/d_k \\
	g/z & h/z & i/z & 0 \\
    0 & 0 & 0 & 1 \\
	\end{bmatrix}
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
    *
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

This gives us NDC coordinates for the first clip map cascade. What if we want to convert the result back to what it would have been if we had used projection-view sample? We can subtract off $2t_x/d_k, 2t_y/dk$ from the result.

What about for the other cascades? We know that for each successive clip map cascade, $d_k$ doubles in size but everything else remains the same including the translation component. So to convert from cascade 0 to cascade n, we can multiply the result by $1/2^i$.

Because of this we only need to pass in the matrix for the very first clip map cascade and we can use it in the shader to convert to any other cascade, or back to the projection-view sample space. Here is some GLSL code that does this:

{% highlight glsl %}
    #define BITMASK_POW2(offset) (1 << offset)

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
        ivec2 pageTableIndex = ivec2(floor(virtualTexCoords * vsmNumPagesXY));
        uint entry = imageLoad(pageTable, ivec3(pageTableIndex, clipMapIndex));

        ivec2 physicalPageXYOffset;
        int memoryPoolIndex;
        UnpackPhysicalMemoryData(entry, physicalPageXYOffset, memoryPoolIndex);
{% endhighlight %}

Once we have these values, what we need to do is figure out which texel the virtualTexCoords are trying to point to. What we know about the physical pages is that they are `128x128` texels in size. So to figure out which texel we need to point to, we can calculate the virtual pixel coordinate and mod it by 128.

{% highlight glsl %}
        vec2 virtualPixelCoords = virtualTexCoords * vsmTexelResolutionXY;
        vec2 physicalPageTexelOffsets = mod(virtualPixelCoords, 128);
{% endhighlight %}

The final step is to convert these coordinates to actual physical coordinates that we can use to sample the physical texture. Since `physicalPageXYOffset` points to the lower-left corner of the physical page (in indices, not texels) and `physicalPageTexelOffsets` points to the texel within that page, we can multiply `physicalPageXYOffset` by the number of texels per page (128) and add it to `physicalPageTexelOffsets`.

{% highlight glsl %}
        return vec3(
            vec2(physicalPageXYOffset) * vec2(128) + physicalPageTexelOffsets,
            float(memoryPoolIndex)
        );
{% endhighlight %}

And we're done! We now have a way to convert from world space coordinates -> virtual coordinates -> physical memory coordinates. If we revisit the rendering step 6 from above, what this means is that we will use `newProjectionViewRender` for each cascade to rasterize the scene. For each fragment that we generate we want to convert it to physical coordinates and write the depth to that location.

In practice this means something like `gl_Position` will be written in the vertex shader using `newProjectionViewRender`, but it will smooth out a set of texture coordinates that it generates using `ConvertWorldPosToPhysicalCoordinates` with the world position. This way the fragment shader uses the smoothed in texture coordinates to write out `gl_FragCoord.z` to the shadow map.

# Allocation Strategy

An easy way to handle allocation for this virtual memory system is to pack a set of free pages into a shader storage buffer object (SSBO). When the GPU wants to allocate, it can pull the next index from the list in a linear fashion. When it wants to free a page it can decrement the index and put the page indices there.

In the following gif the lower left shows the first memory pool as it pulls pages from the free list or adds them back. Black = unallocated.

<img src="/assets/v0.11/svsms/VSM_Allocator.gif" alt="allocator" />

When one memory pool runs out of free space and can't deallocate anything, it is up to the implementation to decide what to do. Evict least relevant? Allocate a new memory pool? Don't render any new shadow map requests until some pages can be freed?

For this implementation the approach was to have a maximum of N memory pools where N is the number of cascades. When one pool fills up it moves to the next. If all were to be full during a frame it would stop processing further requests until new memory became free again.

The decision for when to evict a page from the cache is also configurable. The page table has enough bits to count up to 15 frames as a delay for evicting a page from the cache, but by default it marks pages free as soon as they are no longer required for a given frame (no delay).

For actually implementing a sparse allocation strategy, see the following:
* [Sparse Textures OpenGL](https://gpuopen.com/learn/sparse-textures-opengl/)
* [Sparse Resources Vulkan](https://registry.khronos.org/vulkan/site/spec/latest/chapters/sparsemem.html)
* TODO: Sparse link for DX12?

# Shadow Render Budget

One possible performance optimization is to add a configurable render budget for shadow map generation. To conceptualize this we can return to the clip map page from earlier.

![mips](/assets/v0.11/svsms/VSM_Mips.png)

From this image we can see that each cascade, even though not strictly part of a mip chain, can still be used as one since each successive cascade is fully capable of representing everything from the previous cascade, just at a lower effective resolution.

This allows us to specify one or more cascades to serve as the lowest/coarsest levels in the mip chain. When we perform the depth buffer analysis, not only will we mark the pages in the finest detail mip levels, we will also mark the corresponding pages in the lowest mip levels.

Then what we can do when rendering is to set a budget for how many pages from the finest mips to process before saving them for the next frame. The lowest mips will be rendered fully every frame so we can use them as a fallback when data is not available at a higher resolution.

This gif shows what this looks like (slowed down to exaggerate the shadow pop in):

<img src="/assets/v0.11/svsms/VSM_Popin.gif" alt="popin" />

As the camera moves through the wall, you'll notice that part of the scene has to temporarily fall back to lower (coarser) mip levels until all of the higher (finer) mip levels are processed.

This is a nice option to have since it provides a setting that can be tweaked for different hardware and different scene complexities. It could also be set dynamically to help when the rendering is struggling to maintain target frame times.

# Hardware Shadow Filtering

An issue comes up when we try to use hardware shadow filtering such as with `sampler2DShadow`. For most of the texels within a page the hardware can freely perform its filtering without issue, but the problem is at the page border:

![filter](/assets/v0.11/svsms/VSM_Filter.png)

Everything in the green represents texels that won't cause an issue with hardware shadow filtering. Everything in the red is where the problems start since the hardware will assume it can sample into the next adjacent pages. But the next pages will almost always contain data from completely different parts of the scene.

### Option 1: Border

For this option you can contract or expand the size of the shadow map so that you can add a 1 texel border around each page. Then it is possible to use hardware filtering without any issues or having to handle special cases.

### Option 2: Hybrid

Another option is to write a sampling function that checks to see if the starting texel is on the page boundary. If it is you can perform software shadow filtering so that you can avoid sampling into memory that has nothing to do with the current page.

If not on a border, use hardware filtering as normal.

### Option 3: Manual

The last option is to abandon hardware shadow filtering completely and write your own software filtering code. This has the advantage of not requiring branching in the shader or adding any page borders.

# Post-Processing Effects

There will be times when certain post processing effects such as volumetric lighting might require data to be present in the shadow map that is outside of the current view of the camera. When this is the case it will be necessary to add some extra logic during the page marking/depth prepass stage. The clipmap cascade that is marked will depend on what level of resolution the post processing effect will need. If it can get away with low resolution then it can go with the further/coarser cascades to save on bandwidth and performance.

# Existing Problems/Future Work

TODO: Add

# References
* [Virtual Shadow Maps in Unreal Engine](https://docs.unrealengine.com/5.1/en-US/virtual-shadow-maps-in-unreal-engine/)
* [Virtual Shadow Maps in Fortnite Battle Royale Chapter 4](https://www.unrealengine.com/en-US/tech-blog/virtual-shadow-maps-in-fortnite-battle-royale-chapter-4)
* [The Clipmap: A Virtual Mipmap](https://dl.acm.org/doi/pdf/10.1145/280814.280855)
* [Sparse Virtual Textures](https://www.youtube.com/watch?v=MejJL87yNgI)
* [Understanding Virtual Memory](https://performanceengineeringin.wordpress.com/2019/11/04/understanding-virtual-memory/)
* TODO: Find link to id software megatexture presentation
* [Terrain Rendering Using GPU-Based Geometry Clipmaps](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-2-terrain-rendering-using-gpu-based-geometry)
* [Fitted Virtual Shadow Maps](https://www.cg.tuwien.ac.at/research/publications/2007/GIEGL-2007-FVS/GIEGL-2007-FVS-Preprint.pdf)
* [Queried Virtual Shadow Maps](https://www.cg.tuwien.ac.at/research/publications/2007/GIEGL-2007-QV1/GIEGL-2007-QV1-Preprint.pdf)