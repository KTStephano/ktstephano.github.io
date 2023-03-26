---
permalink: /rendering/stratusgfx/feature_reel
---

This page contains screenshots captured in realtime on an Nvidia GTX 1060 using the StratusGFX rendering engine.

## Realtime Global Illumination + Indirect Shadowing
Accomplished using virtual point lights.

![gi](/assets/portfolio/Sponza2022_gi_2.png)

![gi](/assets/portfolio/Sponza2022_gi_5.png)

![gi](/assets/portfolio/SanMiguel_gi_2.png)

![gi](/assets/portfolio/SanMiguel_gi_6.png)

![gi](/assets/portfolio/bistro_gi.png)

![gi](/assets/portfolio/bistro_gi_2.png)

## Raymarched Volumetric Lighting
Accomplished by using view space raymarching in combination with cascaded shadow maps.

![volumetric](/assets/portfolio/bistro_volumetric.png)

![volumetric](/assets/portfolio/sanmiguel_volumetric.png)

![volumetric](/assets/portfolio/Sponza2022_volumetric.png)

![volumetric](/assets/portfolio/warehouse_volumetric.png)

## Physically Based Metallic-Roughness Pipeline
Accomplished with single-scattering bidirectional reflectance distribution function (BRDF).

![pbr](/assets/portfolio/bistro_pbr.png)

![pbr](/assets/portfolio/Sponza2022_3.png)

## Deferred Lighting + Soft Shadowing
Accomplished using shadow caching with progressive dynamic updates and percentage closer soft shadows.

![deferred](/assets/portfolio/bistro_shadow_1.png)

![deferred](/assets/portfolio/bistro_shadow_2.png)

## ACES Filmic Tonemapping
Maps HDR color values to [0, 1] range before applying gamma correction.

![aces](/assets/portfolio/bathroom_aces.png)

![aces](/assets/portfolio/bathroom_aces2.png)

## Bloom
Accomplished with repeated downsampling of bright spots followed by repeated upsampling and combining with the main image.

![bloom](/assets/portfolio/Bloom.png)

## 3D Models Used In Screenshots

* [https://www.intel.com/content/www/us/en/developer/topic-technology/graphics-research/samples.html](https://www.intel.com/content/www/us/en/developer/topic-technology/graphics-research/samples.html)
* [https://developer.nvidia.com/orca/amazon-lumberyard-bistro](https://developer.nvidia.com/orca/amazon-lumberyard-bistro)
* [https://sketchfab.com/3d-models/the-bathroom-free-d5e5035dda434b8d9beaa7271f1c85fc](https://sketchfab.com/3d-models/the-bathroom-free-d5e5035dda434b8d9beaa7271f1c85fc)
* [https://casual-effects.com/data/](https://casual-effects.com/data/)
* [https://sketchfab.com/3d-models/abandoned-warehouse-1e40d433ed6f48fb880a0d2172aff7ca](https://sketchfab.com/3d-models/abandoned-warehouse-1e40d433ed6f48fb880a0d2172aff7ca)
* [https://sketchfab.com/3d-models/interogation-room-6e9151ec29494469a74081ddc054d569](https://sketchfab.com/3d-models/interogation-room-6e9151ec29494469a74081ddc054d569)