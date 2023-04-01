---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

permalink: /portfolio
title: Portfolio
layout: home

---

![sponza](/assets/portfolio/Sponza2022_gi_2.png)

# StratusGFX

StratusGFX is a realtime 3D rendering engine.

-> [Graphics Feature Reel](/rendering/stratusgfx/feature_reel)

-> [GitHub Repo](https://github.com/KTStephano/StratusGFX)

-> [Video Tech Demo](https://www.youtube.com/watch?v=s5aIsgzwNPE)

-> [Technical Breakdown of a Single Frame](/rendering/stratusgfx/frame_analysis)

Built Using:
* C++17
* OpenGL 4.6

Graphics features currently supported:
* Physically based metallic-roughness pipeline
* Realtime global illumination
* Raymarched volumetric lighting and shadowing
* Cascaded shadow mapping
* Deferred lighting
* Mesh LOD generation and selection
* GPU Frustum Culling
* Screen Space Ambient Occlusion (SSAO)
* Filmic tonemapping
* Fog
* Bloom
* Fast Approximate Anti-Aliasing (FXAA)

Engine features:
* Pool allocators
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