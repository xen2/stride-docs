# Effects and shaders

Stride uses a programmable shading pipeline. You can write [custom shaders](custom-shaders.md), create @'Stride.Graphics.Effect's from them, and use them for drawing. The @'Stride.Rendering.EffectSystem' class provides an easy way to load an effect.

## Load an effect

Use:

```cs
var myEffect = EffectSystem.LoadEffect("MyEffect").WaitForResult();
```

You can then bind the effect as [pipeline state](../low-level-api/pipeline-state.md).

An effect also often defines a set of parameters. To set these, you need to [bind resources](../low-level-api/resources.md) before drawing.

## Shaders

Shaders are authored in the [Stride's shading language](shading-language/index.md), which is an extension of `HLSL`. They provide true composition of modular shaders via [inheritance](shading-language/shader-classes-mixins-and-inheritance.md), shader [mixins](shading-language/composition.md) and [automatic weaving of shader in-out attributes](shading-language/automatic-shader-stage-input-output.md).

## Effects

[Effects](effect-language.md) in Stride use C#-like syntax to further combine shaders. They provide conditional composition of shaders to generate effect permutations.

Games can compile shaders at runtime on every platform, iOS and Android included, but a new permutation then stalls the frame that first needs it. To compile them ahead of time, effect permutation files (`.sdeffectlog`) enumerate the permutations a game uses.

## Target everything

Each shader is compiled once, then effects mix and compose the compiled shaders into SPIR-V. Stride converts it automatically to the target graphics platform: `SPIR-V` for Vulkan, `DXIL` for Direct3D 12, and `HLSL` for Direct3D 11. On macOS and iOS, Vulkan runs on Metal through MoltenVK.

## In this section

* [Effect language](effect-language.md)
* [Shading language](shading-language/index.md)
    - [Shader classes, mixins and inheritance](shading-language/shader-classes-mixins-and-inheritance.md)
    - [Composition](shading-language/composition.md)
    - [Compile shaders](compile-shaders.md)
    - [Templates](shading-language/templates.md)
    - [Shader stage input/output automatic management](shading-language/automatic-shader-stage-input-output.md)
* [Custom shaders](custom-shaders.md)
* [Shader development with Visual Studio Code](shader-development-vscode.md)
