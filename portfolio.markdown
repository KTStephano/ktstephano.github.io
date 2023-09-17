---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

permalink: /portfolio
title: Portfolio
layout: home

---

* TOC
{:toc}

# StratusGFX Realtime Graphics Engine

![sponza1](/assets/v0.10/SponzaGI_Front.png)
![sponza2](/assets/v0.10/SponzaGI.png)

(3D Model: Intel Sponza)

![sanmiguel1](/assets/v0.10/FinalAfterPostProcessing.png)

![sanmiguel2](/assets/v0.10/SanMiguel_Balcony2.png)

(3D Model: San Miguel)

![bistro](/assets/v0.10/Bistro1.png)

(3D Model: Bistro)

![cornell_front](/assets/v0.10/Cornell_Front.png)

![cornell_back](/assets/v0.10/Cornell_Back.png)

(3D Model: Cornell Box)

-> [GitHub Repo](https://github.com/KTStephano/StratusGFX)

-> [Video Tech Demo](https://www.youtube.com/watch?v=dj0wVxwd1ng)

-> [Image Feature Reel](/rendering/stratusgfx/feature_reel)

-> [Technical Breakdown of a Single Frame](/rendering/stratusgfx/frame_analysis_v0_10)

Stratus is a realtime, GPU-driven graphics engine which uses a deferred rendering pipeline. Key features include physically-based volumetric lighting, global illumination, image denoising and other modern features. It is optimized to run on lower-end hardware while still providing good visual quality.

A full list of features can be found on [the GitHub repo](https://github.com/KTStephano/StratusGFX).

# Sparse Virtual Shadow Maps

![vsm_bistro](/assets/v0.11/svsms/VSM1_Bistrov2.png)

<img src="/assets/v0.11/svsms/VSM_Invalidate.gif" alt="invalidate" />
(Virtual address space shifting as light rotates)

<img src="/assets/v0.11/svsms/VSM_Cascade_Change.gif" alt="cascade_change" />
(Cascades increasing in resolution as camera moves closer)

This is a virtual memory system and data caching solution for high-quality realtime shadows over large distances. A full tech writeup can be found here: [Sparse Virtual Shadow Maps](/rendering/stratusgfx/svsm)

# Interactive Software Raytracing

![cornell_left](/assets/v0.11/Cornell6.png)

![cornell_right](/assets/v0.11/Cornell5.png)

(3D Model: Cornell Box)

![rt_csponza](/assets/v0.11/RTSponza12.png)

(3D Model: Crytek Sponza)

(Emissive Texture: [https://www.pxfuel.com/en/desktop-wallpaper-ioptr](https://www.pxfuel.com/en/desktop-wallpaper-ioptr))

This is an experimental feature still under development. The goal is to provide interactive software ray tracing on the GPU which does not require specialized RT hardware.

Features:

* Multi-bounce ray traced lighting
* Directional lights and emissive surfaces

To maintain interactive frame rates, the GPU ray traces a simplified version of the scene. Basic denoising with temporal accumulation is then applied.

Runs on:

* GTX 10-series cards and above