# Graphics

This section explains how to use Game Studio and the Stride API for graphics and rendering.

## Shaders

Shaders are written in the [Stride shading language (SDSL)](effects-and-shaders/shading-language/index.md), an extension of `HLSL`. You build on the engine shaders through [inheritance and overrides](effects-and-shaders/shading-language/shader-classes-mixins-and-inheritance.md), [modules](effects-and-shaders/shading-language/shaders-and-modules.md), [composition](effects-and-shaders/shading-language/composition.md), and [streams](effects-and-shaders/shading-language/automatic-shader-stage-input-output.md) that the compiler passes between shader stages.

## Effects

[Effects](effects-and-shaders/effect-language.md) in Stride use C#-like syntax to combine shaders. They provide conditional composition of shaders to generate effect permutations.

## Target everything

Stride shaders are converted automatically to the target graphics platform, either plain `HLSL` for Direct3D or `SPIR-V` for Vulkan platforms.

## Advanced graphics

The graphics module provides a set of methods to display the game. Although Stride is available on multiple platforms, the whole system behaves like Direct3D 11 from the user perspective. You need a basic knowledge of the rendering pipeline to use it.

## In this section

* [Cameras](cameras/index.md)
* [Materials](materials/index.md)
* [Textures](textures/index.md)
* [Lights and shadows](lights-and-shadows/index.md)
* [Post effects](post-effects/index.md)
* [Window Configuration](window-configuration.md)
* [Graphics compositor](graphics-compositor/index.md)
* [Effects and shaders](effects-and-shaders/index.md)
* [Low-level API](low-level-api/index.md)
* [Rendering pipeline](rendering-pipeline/index.md)
* [Sprite fonts](sprite-fonts.md)
* [Voxel Cone Tracing GI](lights-and-shadows/voxel-cone-tracing-gi.md)
* [Graphics API](graphics-api.md)
