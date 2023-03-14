---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

permalink: /portfolio
title: Portfolio
layout: home

---

<iframe width="600" height="350" src="https://www.youtube.com/embed/s5aIsgzwNPE" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

# StratusGFX

StratusGFX is a 3D rendering engine written in C++17 using OpenGL 4.6.

[Click here to read a technical breakdown of a single frame.](/rendering/stratusgfx/frame_analysis)

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

Modern graphics API features used:
* Compute shaders
* Direct state access
* Programmable vertex pulling
* Multi draw elements indirect
* Shader storage buffers