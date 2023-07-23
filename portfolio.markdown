---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

permalink: /portfolio
title: Portfolio
layout: home

---

# StratusGFX Realtime Graphics Engine

![sponza](/assets/v0.10/SponzaGI_Front.png)

(3D Model: Intel Sponza)

![sanmiguel](/assets/v0.10/FinalAfterPostProcessing.png)

(3D Model: San Miguel)

![cornell_front](/assets/v0.10/Cornell_Front.png)

![cornell_back](/assets/v0.10/Cornell_Back.png)

(3D Model: Cornell Box)

-> [Graphics Feature Reel](/rendering/stratusgfx/feature_reel)

-> [GitHub Repo](https://github.com/KTStephano/StratusGFX)

-> [Video Tech Demo](https://www.youtube.com/watch?v=s5aIsgzwNPE)

-> [Technical Breakdown of a Single Frame](/rendering/stratusgfx/frame_analysis_v0_10)

Built Using:
* C++17
* OpenGL 4.6

Graphics features currently supported:
* Physically based metallic-roughness pipeline
* Realtime global illumination
* Spatiotemporal image denoising
* Raymarched volumetric lighting and shadowing
* Cascaded shadow mapping
* Deferred lighting
* Mesh LOD generation and selection
* GPU Frustum Culling
* Screen Space Ambient Occlusion (SSAO)
* Reinhard or ACES Tonemapping
* Fog
* Bloom
* Fast Approximate Anti-Aliasing (FXAA)
* Temporal Anti-Aliasing (TAA)

Engine features:
* Pool allocators
* GPU memory allocators/managers
* Multi threaded utilities
* Concurrent hash map
* Entity-Component System (ECS)
* Logging

Modern graphics API features used:
* Compute shaders
* Direct state access
* Programmable vertex pulling
* Multi draw elements indirect
* Shader storage buffers